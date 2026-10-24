```c++
// 边界块
#include <climits>
#include <cstdlib>
#include <iostream>
#include <ostream>
struct BlockBorder {
  int Blocklength;
  bool isfree() {
    if (Blocklength > 0) {
      return true;
    } else {
      return false;
    }
  }
  int MemorySize() {
    if (Blocklength > 0) {
      return Blocklength;
    } else {
      return -Blocklength;
    }
  }
  int BlockSize() { return MemorySize() + 2 * sizeof(BlockBorder); }
};

// 空闲块头结构
struct FreeBlockHead : public BlockBorder {
  FreeBlockHead *prior;
  FreeBlockHead *next;
};

// MemoryPool类框架
class MemoryPool {
public:
  MemoryPool(unsigned int size) {
    PoolSize = size;
    Base = new char[PoolSize];
    // 申请大块内存空间
    First = (FreeBlockHead *)Base;
    First->Blocklength = size - 2 * sizeof(BlockBorder); // 实际可用空间
    First->prior = nullptr;
    First->next = nullptr; // 此时前后无相邻空闲块
    SetUsedBorder(Base, First->Blocklength);
  };
  ~MemoryPool() { delete[] Base; };

  void *allocate(int size) {
    FreeBlockHead *current = First;
    while (current && current->Blocklength < size) {
      current = current->next;
    }
    if (!current) {
      return nullptr;
    }
    int remain = current->Blocklength - size - 2 * sizeof(BlockBorder);
    if (remain > (int)sizeof(BlockBorder)) {
      SpiltBlock(current, size);
    } else {
      int _size = current->Blocklength;
      DeleteFreeBlock(current);
      current->Blocklength = -_size;
      BlockBorder *footer =
          (BlockBorder *)((char *)current + sizeof(BlockBorder) + _size);
      footer->Blocklength = -_size;
    }
    return (char *)current + sizeof(BlockBorder);
  }; // 最先拟合策略

  void *allocate_two(int size) {
    FreeBlockHead *current = First;
    FreeBlockHead *bestmin = nullptr;
    int min = INT_MAX;
    int remain;
    while (current) {
      remain = current->Blocklength - size - 2 * sizeof(BlockBorder);
      if (remain == 0) {
        bestmin = current;
        break;
      }
      if (remain < min) {
        min = remain;
        bestmin = current;
      }
      current = current->next;
    }
    if (bestmin) {
      if (remain > (int)sizeof(BlockBorder)) {
        SpiltBlock(bestmin, size);
      } else {
        int _size = bestmin->Blocklength;
        DeleteFreeBlock(bestmin);
        bestmin->Blocklength = -_size;
        BlockBorder *footer =
            (BlockBorder *)((char *)bestmin + sizeof(BlockBorder) + _size);
        footer->Blocklength = -_size;
      }
      return (char *)bestmin + sizeof(BlockBorder);
    }
    return nullptr;
  }; // 最优拟合策略

  void free(void *p) {
    if (!p) {
      return;
    }
    FreeBlockHead *current = (FreeBlockHead *)((char *)p - sizeof(BlockBorder));
    current->Blocklength = abs(current->Blocklength);
    TryMerge(current);
  };

  void rellocate() {}

  void test() {
    std::cout << "\n---Memory---" << std::endl;
    char *current = Base;
    int total = 0;
    while (total < PoolSize) {
      BlockBorder *p = (BlockBorder *)current;
      int len = p->MemorySize();
      bool isfree = p->isfree();
      std::cout << (void *)current << ' ' << isfree << ' ' << len << ' '
                << p->BlockSize() << std::endl;
      current = current + p->BlockSize();
      total = total + p->BlockSize();
    }
    std::cout << "------\n" << std::endl;
  }

private:
  char *Base;
  unsigned int PoolSize;
  FreeBlockHead *First;
  void InsertFreeBlock(FreeBlockHead *p, int size) {
    p->next = First;
    p->prior = nullptr;
    if (First) {
      First->prior = p;
    }
    First = p;
  };

  void SetUsedBorder(void *p, int size) {
    ((BlockBorder *)p)->Blocklength = size;
    BlockBorder *tail =
        (BlockBorder *)((char *)p + sizeof(BlockBorder) + abs(size));
    tail->Blocklength = size;
  };

  void DeleteFreeBlock(FreeBlockHead *p) {
    if (p->prior) {
      p->prior->next = p->next;
    } else {
      First = p->next;
    }
    if (p->next) {
      p->next->prior = p->prior;
    }
  }
  BlockBorder *Getpreblock(BlockBorder *p) {
    BlockBorder *prior_a = (BlockBorder *)((char *)p - sizeof(BlockBorder));
    if ((char *)prior_a <= Base) {
      return nullptr;
    }
    return (BlockBorder *)((char *)p - prior_a->BlockSize());
  };

  BlockBorder *Getnextblock(BlockBorder *p) {
    char *next_one = (char *)p + p->BlockSize();
    if (next_one >= Base + PoolSize) {
      return nullptr;
    }
    return (BlockBorder *)next_one;
  };

  BlockBorder *Getcurrentblock(void *p) {
    return (BlockBorder *)((char *)p - sizeof(BlockBorder));
  };

  void TryMerge(BlockBorder *p) {
    BlockBorder *_next = Getnextblock(p);
    if ((char *)_next < Base + PoolSize && _next->isfree()) {
      DeleteFreeBlock((FreeBlockHead *)_next);
      int newsize =
          p->Blocklength + _next->Blocklength + 2 * sizeof(BlockBorder);
      p->Blocklength = newsize;
      SetUsedBorder(p, newsize);
    }
    BlockBorder *_prior = Getpreblock(p);
    if ((char *)_prior > Base && _prior->isfree()) {
      DeleteFreeBlock((FreeBlockHead *)_prior);
      int newsize =
          p->Blocklength + _prior->Blocklength + 2 * sizeof(BlockBorder);
      p->Blocklength = newsize;
      SetUsedBorder(_prior, newsize);
      p = _prior;
    }
    InsertFreeBlock((FreeBlockHead *)p, p->Blocklength);
  };

  void SpiltBlock(FreeBlockHead *p, int size) {
    int oldsize = p->Blocklength;
    int remain = p->Blocklength - size - 2 * sizeof(BlockBorder);
    DeleteFreeBlock(p);
    SetUsedBorder(p, -size);
    p->Blocklength = -p->Blocklength;
    FreeBlockHead *remainblock =
        (FreeBlockHead *)((char *)p + size + 2 * sizeof(BlockBorder));
    remainblock->Blocklength = remain;
    BlockBorder *remainblock_footer =
        (BlockBorder *)((char *)remainblock + remain + sizeof(BlockBorder));
    remainblock_footer->Blocklength = remain;
    InsertFreeBlock(remainblock, remain);
  };
};
```

