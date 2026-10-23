# `std::thread`的构造函数
`std::thread`一共有四个构造函数 分别是
1. 默认构造函数
2. 移动构造函数 即转移线程的所有权 详见[[线程#`std thread`转移所有权]]
3. 复制构造函数 因为两个`std::thread`不可能表示同一个线程 所以这个我们定义为弃置
4. 模板构造函数 构造新的`std::thread`对象并将它与执行线程关联 表示新的执行线程开始执行

前三个很简单 重点看第四个

举个例子
```cpp
struct X{
    X(X&& x)noexcept{}
    template <class Fn, class... Args>
    X(Fn&& f,Args&&...args){}
    X(const X&) = delete;
};

X x{ [] {} };
X x2{ x }; // 选择到了有参构造函数，不导致编译错误
```
在这个例子中 在重载决议的竞争中 模板构造函数赢了复制构造函数 所以 我们原来禁用掉的复制构造函数实际上是可以使用的 这就会导致多个`thread`表示同一个线程 这是不行的

所以我们要加一个约束
```cpp
template <class Fn, class... Args, std::enable_if_t<!std::is_same_v<std::remove_cvref_t<Fn>, X>, int> = 0>
```
简单来说就是不允许编译器匹配模板构造函数而且找复制构造函数

# 模板构造函数的内部

这一块比较难理解

先看源码：
```cpp
template <class _Fn, class... _Args>
void _Start(_Fn&& _Fx, _Args&&... _Ax) {
    using _Tuple                 = tuple<decay_t<_Fn>, decay_t<_Args>...>;
    auto _Decay_copied           = _STD make_unique<_Tuple>(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
    constexpr auto _Invoker_proc = _Get_invoke<_Tuple>(make_index_sequence<1 + sizeof...(_Args)>{});

    _Thr._Hnd =
        reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr, 0, _Invoker_proc, _Decay_copied.get(), 0, &_Thr._Id));

    if (_Thr._Hnd) { // ownership transferred to the thread
        (void) _Decay_copied.release();
    } else { // failed to start thread
        _Thr._Id = 0;
        _Throw_Cpp_error(_RESOURCE_UNAVAILABLE_TRY_AGAIN);
    }
}
```
这个可以大体理解为：**先把传入的参数或模板打包 传进新线程后再解包**
这是因为：**操作系统底层的线程 API 是 C 语言风格的 无法直接理解 C++ 的多参数或模板**
简单来说就是历史包袱

这里有几个比较重要的点要理解

## `using _Tuple = tuple<decay_t<_Fn>, decay_t<_Args>...>`

[`std::tuple`](https://zh.cppreference.com/w/cpp/utility/tuple)简单来说 就是将**多个不同类型的值**组合成一个单一的对象

[`std::decay_t`](https://zh.cppreference.com/w/cpp/types/decay)简单来说 就是进行类型退化 具体就是

|原始类型 `T`|`std::decay_t<T>` 的结果|说明|
|---|---|---|
|`int`|`int`|保持不变|
|`int&` / `int&&`|`int`|去掉引用|
|`const int`|`int`|去掉 `const`|
|`const int&`|`int`|同时去掉引用和 `const`|
|`int[5]`|`int*`|数组退化为指针|
|`void(int)`|`void(*)(int)`|函数退化为函数指针|

这是为了主线程在创建子线程时 传入的可调用对象和参数必须安全地传递到子线程的内存空间中
因此 利用`std::decay_t`把所有参数转为**纯净的无引用、无 cv 限定的类型**

## `auto _Decay_copied = _STD make_unique<_Tuple>(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...)`

简单来说 创建`std::make_unique`是为了在heap上保留一份参数 防止原线程结束后参数被销毁 参见[[图解OS/内存管理/malloc#Linxu进程的内存分布|栈与堆]]

但是问题出现了 底层API只能接受`void*`类型的参数

## `constexpr auto _Invoker_proc = _Get_invoke<_Tuple>(make_index_sequence<1 + sizeof...(_Args)>{})`

先看这两个函数源码
```cpp
 template <class _Tuple, size_t... _Indices>
 _NODISCARD static constexpr auto _Get_invoke(index_sequence<_Indices...>) noexcept {
     return &_Invoke<_Tuple, _Indices...>;
 }
 
 template <class _Tuple, size_t... _Indices>
 static unsigned int __stdcall _Invoke(void* _RawVals) noexcept /* terminates */ {
     // adapt invoke of user's callable object to _beginthreadex's thread procedure
     const unique_ptr<_Tuple> _FnVals(static_cast<_Tuple*>(_RawVals));
     _Tuple& _Tup = *_FnVals.get(); // avoid ADL, handle incomplete types
     _STD invoke(_STD move(_STD get<_Indices>(_Tup))...);
     _Cnd_do_broadcast_at_thread_exit(); // TRANSITION, ABI
     return 0;
 }
```

先看`static unsigned int __stdcall _Invoke(void* _RawVals) noexcept`
这个里的_RawVals就是指向之前保存在heap上的参数

接着  虽然这是个`void*` 但是可以将它转换为 `_Tuple*` 类型 构造一个独占智能指针 然后调用 get() 成员函数获取底层指针 解引用指针 得到元组的引用初始化`_Tup`

最后就是通过`_STD get<_Indices>(_Tup)`来解包

`std::invoke`是为了统一接口

### 为什么还要`std::move`

先看`_STD get<_Indices>(_Tup)`

Tup 是左值 
因此：`std::get<_Indices>(_Tup)` 得到的是左值引用  

但 `tuple` 中保存的这些对象已经属于新线程 
调用用户函数时可以把它们作为右值传给 std::invoke 

所以源码使用： 
`std::move(std::get<_Indices>(_Tup))` 
这样可移动类型可以移动给最终的调用对象
也符合 `std::thread` 对保存参数的调用语义。（[[线程#`std thread`转移所有权]]）

同时也避免再拷贝一次的性能开销 heap上的参数这时候的使命已经结束 之后会随着`_Decay_copied`销毁 所以直接变成右值交给用户函数

## `_Thr._Hnd = reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr, 0, _Invoker_proc, _Decay_copied.get(), 0, &_Thr._Id))`

调用 [`_beginthreadex`](https://learn.microsoft.com/zh-cn/cpp/c-runtime-library/reference/beginthread-beginthreadex?view=msvc-170) 函数来启动一个线程，并将线程句柄存储到 `_Thr._Hnd` 中。传递给线程的参数为 `_Invoker_proc`（一个静态函数指针，就是我们前面讲的 **`_Invoke`**）和 `_Decay_copied.get()`（存储了函数对象和参数的副本的指针）


# 一些问题

1. **如何做到的默认按值复制？**
   `_Start` 的第一行代码展示了这一点 我们将传入的所有参数包装成一个元组类型 这些类型先经过 `decay_t` 处理 去除了引用与 cv 限定 自然就实现了默认复制


2. **为什么需要 `std::ref`**
   实现中将类型先经过 `decay` 处理 如果要传递引用，则必须用类包装一下才行 使用 `std::ref` 函数就会返回一个包装对象


3. **如何支持只能移动的对象？**
   参数通过完美转发 最终调用时使用 `std::move` 这在线程实际执行的函数 `_Invoke` 中体现出来


4. 如何做到接受任意可调用对象？
   源码的实现很简单 主要是通过两层包装 最终将 `void*` 指针转换到原类型 然后使用 `std::invoke` 进行调用


5. **如何创建的线程？**
   MSVC STL 调用 Win32 API `_beginthreadex` 创建线程  libstdc++ 调用 `__gthread_create` 函数创建线程 在 Windows 上实际上就是调用 `CreateThread`。  
   `_beginthreadex` 和 `CreateThread` 都是微软提供的用于创建线程的 C 风格接口 它们的主要区别在于前者依赖于 C 运行时库 而后者更适合纯 Windows API 的情况  使用 `_beginthreadex` 可以确保正确初始化和清理 C 运行时库资源 而 `CreateThread` 则适用于不依赖于 C 运行时库的环境


6. **“`std::thread` 内部会将保有的参数副本转换为右值表达式进行传递”到底是如何做到的？**
   这就是第三个问题 差不多 无非是最后调用 `std::invoke` 函数之前 先 `std::move` 了