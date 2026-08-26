**本人志在持续更新计算机系统、计算机网络、C++语言的核心知识点的系列合集，以易懂、全面的方式讲解底层知识。对于正在准备面试八股的朋友来说，本系列涵盖了本人面试中遇到的所有考点以及许多相关拓展知识，读完后能帮助你从容面对大部分面试拷打；对于想要深入学习计算机知识的朋友来说，本系列比较系统地介绍了操作系统和网络等重点内容，也举了不少例子，大大有助于你从底层的视角去理解计算机系统。*
	*先说明，本系列恐怕不是计算机小白或是想速通期末的朋友们的目标，它需要一定系统和语言基础，也并不是面向教材和考试要求去讲解，所以更适合那些实操过代码、了解一些计算机系统知识、并且想要深入底层和扎实基础的朋友们去耐心学习。如果你是这样的人，欢迎阅读该系列文章，并分享自己的理解或提出文章中的模糊、错误的地方（不排除有）。*
	想要阅读系列中其他内容或想要持续关注本系列更新可移步：https://github.com/feiyangyang11/Cpp-Core-CS-Interview-Guide.git。*

------

# vector 底层原理详解

## 朴素版源码与实现思路

下面是简化版的 vector 容器源码，主要突出核心成员变量、核心函数逻辑

### 实现思路

vector 本身一般是栈上对象，且并不是个数组，实际数据都放在一个堆数组上，它本身只有三个指针类型的成员变量

本质上用三个指针管理一块连续动态内存：`begin` 指向首元素，`end` 指向已构造元素末尾，`cap` 指向已分配空间末尾

元素内存通过 `allocator` 申请

管理器`allocator_traits` 统一负责 `allocate / construct / destroy / deallocate`（内存分配 / 对象构造 / 对象析构 / 内存回收）

支持动态扩容、移动和拷贝构造

整体核心就是：**连续内存 + 三指针边界管理 + 对象生命周期与内存生命周期分离**

### 源码

```cpp
#include <memory>       // std::allocator, allocator_traits
#include <utility>      // std::move_if_noexcept
#include <cstddef>      // size_t

template<class T>
class MyVector {
public:
    using Alloc = std::allocator<T>;
    using Traits = std::allocator_traits<Alloc>;

private:
    Alloc alloc_;

    // vector 本质上最核心就是三个指针
    T* begin_ = nullptr;     // 第一个元素
    T* end_ = nullptr;       // 最后一个有效元素的下一个位置
    T* cap_ = nullptr;       // 已分配内存的末尾

public:

    MyVector() = default;

    ~MyVector() {
        // 调用所有已经构造出来的 T 的析构函数
        clear();
        // destroy     -> 调用对象析构
        // deallocate  -> 释放原始内存
        if (begin_) {
            Traits::deallocate(
                alloc_,
                begin_,
                capacity()
            );
        }
    }

    size_t size() const {
        // 指针差 = 已经构造的元素数量
        return end_ - begin_;
    }

    size_t capacity() const {
        // [begin_, cap_) 是 vector 拥有的整块内存
        return cap_ - begin_;
    }

    bool empty() const {
        return begin_ == end_;
    }

    T& operator[](size_t index) {
        // vector 的 [] 本身不检查越界
        return begin_[index];
    }

    const T& operator[](size_t index) const {
        return begin_[index];
    }

    void clear() {
        // 从后往前析构所有元素
        while (end_ != begin_) {
            --end_;

            Traits::destroy(
                alloc_,
                end_
            );
        }
    }

    void push_back(const T& value) {
        // 左值版本 -> 新元素需要拷贝构造

        if (end_ == cap_) {
            // 没空间了，进行扩容
            grow();
        }
        // 在 end_ 指向的“未构造内存”上构造 T
        Traits::construct(
            alloc_,
            end_,
            value
        );

        ++end_;
    }

    void push_back(T&& value) {
        // 右值版本 -> 新元素移动构造
        if (end_ == cap_) {
            grow();
        }

        Traits::construct(
            alloc_,
            end_,
            std::move(value)
        );

        ++end_;
    }

    template<class... Args>
    T& emplace_back(Args&&... args) {
        if (end_ == cap_) {
            grow();
        }
        // 直接在 vector 内存中构造对象
        Traits::construct(
            alloc_,
            end_,
            std::forward<Args>(args)...
        );

        T* new_element = end_;
        ++end_;

        return *new_element;
    }

    void reserve(size_t new_capacity) {
        // reserve 只扩 capacity，不改变 size
        if (new_capacity <= capacity()) {
            return;
        }
        reallocate(new_capacity);
    }
    void resize(size_t n) {
    if (n < size()) {
        // 缩小
        while (size() > n) {
            --end_;
            Traits::destroy(alloc_, end_);
        }
    }
    else if (n <= capacity()) {
        // 容量够，直接构造新元素
        while (size() < n) {
            Traits::construct(alloc_, end_);
            ++end_;
        }
    }
    else {
        // 容量不够
        reallocate(/* new capacity >= n */);

        while (size() < n) {
            Traits::construct(alloc_, end_);
            ++end_;
        }
    }
}

private:

    void grow() {
        size_t old_capacity = capacity();
        // 实际 STL 不保证一定 ×2，这里只为了容易理解
        size_t new_capacity = old_capacity == 0 ? 1 : old_capacity * 2;
        reallocate(new_capacity);
    }

    void reallocate(size_t new_capacity) {
        T* new_begin =
            Traits::allocate(
                alloc_,
                new_capacity
            );

        T* new_end = new_begin;

        try {
            for (T* p = begin_; p != end_; ++p) {

                Traits::construct(
                    alloc_,
                    new_end,
                    std::move_if_noexcept(*p)
                );

                ++new_end;
            }
        }
        catch (...) {
            /*
             * 如果迁移过程中失败，抛出异常
             * 已经成功构造出来的 A、B 必须析构。
             */
            while (new_end != new_begin) {
                --new_end;
                Traits::destroy(alloc_,new_end);
            }
            // 再释放新申请的原始内存
            Traits::deallocate(
                alloc_,
                new_begin,
                new_capacity
            );
            // 继续把异常抛给上层
            throw;
        }

        size_t old_capacity = capacity();

        for (T* p = begin_; p != end_; ++p) {
            Traits::destroy(alloc_,p);
        }

        // 释放旧内存
        if (begin_) {
            Traits::deallocate(
                alloc_,
                begin_,
                old_capacity
            );
        }

        //最后把三个核心指针指向新内存。
        begin_ = new_begin;
        end_   = new_end;
        cap_   = new_begin + new_capacity;
    }
};
```

## 核心成员变量

vector 的核心数据对象都保存在堆上，在栈对象中通过三个指针进行管理，**左闭右开**

`T* begin_`：保存数组第一个有效元素的地址   

`T* end_`：保存数组最后一个有效元素的下一个位置的地址     

`T* cap_`：保存已分配的数组内存的末尾位置地址

`Alloc alloc_`：**内存分配器实例**，vector 通过它申请数组内存，它从 malloc / 自定义memory pool / arena……中申请内存。可以自定义内存分配器并通过 vector 的模板参数传入，如`std::vector<T, MyAllocator<T>> v;`，这样就 vector 就可以从自定义的内存池申请内存。如果使用的是默认的`allocator`，就会走标准的动态内存分配路径（new / malloc）申请内存

## 核心函数

### 长度、容量

- `vec.size()`：有效元素的个数，由 `end_ - begin_` 得到
- `vec.capcity()`：数组容量，表示当前数组能容纳的最多有效元素的个数，由 `cap_ - begin_` 得到
- `vec.resize(N)`：在数组末尾填充元素直至有效元素个数至 N，如果 N > `vec.capcity()`就扩容数组再填充；如果 N <`vec.size()`，那么就析构多出的数组元素
- `vec.reserve(N)`：改变数组容量，若 N >`vec.capcity()`就触发扩容；若 N <`vec.capcity()`则直接返回

### 内存与元素管理

对于数组元素，有插入与删除操作，对应着内存分配/释放、元素构造/析构

插入元素时，通常先分配可用内存，再在内存上调用元素的构造函数构造出对象

删除元素时，通常先调用对象的析构函数，再回收对象原来占有的这片内存

当然不是每次和删除都伴随着内存的分配与回收，此处只是为了建立一个清晰的模型来介绍数组的机制

- `Traits::allocate(alloc_,new_capacity)`：通过`alloc_`申请内存，返回新内存的起址
- `Traits::construct(alloc_,new_end,std::move_if_noexcept(*p))`：在数组末尾构造出一个新对象。优先尝试调用移动构造函数来构造对象，通过`std::move_if_noexcept(*p)`判断该类的移动构造是否会抛异常，如果会抛异常就退化为拷贝构造
- `Traits::destroy(alloc_,new_end)`：主动调用数组最后一个有效元素的析构函数，移除对象
- `Traits::deallocate(alloc_,begin_,old_capacity)`：释放原数组申请的所有内存

`Traits`是提供统一接口的内存管理器，它调用`alloc_`提供的接口函数管理内存和对象。四个接口都要传入`alloc_`实例是因为——如果希望在操作内存或对象时增加一些自定义逻辑，比如统计构造次数、使用特殊内存、记录调试信息……就需要传入**自定义**`alloc_`。为了兼容这种需求，所以`Traits`的接口都要求传入`alloc_`实例

### 扩容 

数组元素个数即将超出容量——触发扩容，首先调用`grow()`

`grow()`：算出 `new_capacity`（通常扩充为原容量的 2 倍或 1.5 倍），然后调用`reallocate(new_capacity)`

`reallocate(new_capacity)`：通过静态方法`Traits::allocate`申请`new_capacity`大小的新内存，然后把原数组元素迁移过去，优先移动构造，其次拷贝构造。如果构造过程中抛出异常，**立即析构所有对象并释放新申请的内存**，然后将异常抛给上层；如果成功构造，就析构原有内存上的对象并释放所有内存，然后更新三个指针