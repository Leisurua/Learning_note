# 互斥量

如果一个类想要被当作“互斥锁” 那么必须满足三个核心操作：`lock()` `try_lock()` `unlock()`

## 互斥量不可移动不可复制

如果可以复制 如：
```cpp
std::mutex m1;
std::mutex m2=m1;
```

如果`m1` `m2` 指向同一把锁：
那么
```cpp
m1.lock();
m2.unlock();
```
这样子会使生命周期 所有权变得复杂

如果`m1` `m2` 不指向同一把锁 即复制出两把独立的锁
那么
```cpp
m1.lock();
	 data++;//线程A

m2.lock();
     data++;//线程B
```
两个线程可以同时访问`data` 互斥锁失效

# `std:lock_guard`

这里也用到了[[RAII]]的思想 析构时自动调用`unlock()` 

# 在不同作用域传递互斥量

 首先明白一点 Mutex是**不可复制不可移动**的 所以所谓的在不同的作用域传递互斥量 本质就是传递它们的指针或引用 可以利用类来传递 例如`std::unique_lock` 
 
 这里的核心 和[[线程#`std thread`转移所有权]]/[[thread的源码解析#为什么还要`std move`]]是类似的
 
 所以类似的 就是通过`std::move` 或函数返回来转移

例如：
```cpp
std::unique_lock<std::mutex> get_lock(){
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk{ some_mutex };
    return lk;
}
void process_data(){
    std::unique_lock<std::mutex> lk{ get_lock() };
    // 执行一些任务...
}
```


## 解释一下`extern` 

因为传递的是指针或引用 所以互斥量的生存周期必须大于使用它的作用域
所以 例如全局/静态存储期 也就是通过`extern` 声明的全局互斥量 类静态成员变量
如果不加`extern` 那么`get_lock()` 执行结束后就会自动销毁 造成垂悬指针

那为什么不在heap上创建一个mutex 而非要在stack上呢？

这是因为 如果我们手动`new` 即在heap上创建互斥量 那就要手动`delete` 在这种多线程的情况下 极易发生内存泄露

### 那为啥不用`std::unique` 呢

原因很简单 因为互斥量是不可移动不可复制的 那使用智能指针时 只能不断`std::move` 太麻烦了 也很容易出错


# 共享数据的初始化

分三类 双检锁 `std::call_once` 和静态局部变量初始化

## 双检锁

```cpp
Data* get_data() {
    if (p == nullptr) {                // 第一次检查

        std::lock_guard<std::mutex> lock(m);

        if (p == nullptr) {            // 第二次检查
            p = new Data;
        }
    }

    return p;
}
```

但是有个问题 第一次检查是否为`nullptr` 是在`mutex` 外的 这就导致了可能两个线程 线程A获得mutex后在初始化数据 但是B同时访问了`p` 这是未定义的

所以双检锁这种写法是不正确的

## `std::call_once`

记住一点：`std::call_once` 只需要接受可调用对象即可

核心就是`std::once_flag flag` 这是多线程共享的

所以`std::call_once(flag,f)` 意思就是：对于这个`flag` 保证`f` 成功执行一次

## 静态局部变量

简单来说 就是一个线程初始化 其他线程等待 等构造完成后再继续

# 读写锁

一句话：读的时候允许同时持有锁 但是写入的时候只允许一个持有
```cpp
#include <shared_mutex>

std::shared_mutex m;

std::shared_lock<std::shared_mutex> lock(m);//读取

std::unique_lock<std::shared_mutex> lock(m);//修改
```



# `std::recursive_mutex`

允许同一线程多次锁定同一个互斥量 而不会造成死锁

`lock()` 几次就要`unlock()` 几次


# `new`与`delete`

一句话：`new` 表达式线程安全要考虑三方面：`operator new` 构造函数（涉及共享资源） 修改指针（涉及共享资源）

`delete` 表达式线程安全考虑两方面：`operator delete` 析构函数（涉及共享资源）

C++ 只保证了 `operator new` `operator delete` 这两个方面的线程安全（不包括用户定义的）


# 线程存储期

简单来说 `thread_local` 创建的对象的生命周期是当前线程运行期间