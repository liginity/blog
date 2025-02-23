+++
title = "C++ std::vector 的 libc++ 源码解读"
date = 2025-02-21T20:00:00+08:00
draft = true
tags = ['c++']
+++


<!-- # C++ `std::vector` 的 libc++ 源码解读 -->

这里将解读 libc++ 14 中的 `std::vector` 源码。github 上 libc++ 的源码在 [这里][1]。
首先介绍 `vector` 实现的基本框架，再解读一些实现细节。

<!--more-->

注意，在 linux 上，g++ 和 clang++ 一般使用 libstdc++ 这个标准库实现。
如果要使用 libc++ 作为标准库实现，需要安装 libc++ 开发所需文件，并为 g++ 或 clang++ 添加命令行参数 `-stdlib=libc++`。

[1]: https://github.com/llvm/llvm-project/blob/llvmorg-14.0.6/libcxx/include/vector

## `std::vector` 实现的基本框架

下面是简化和修改后的 libc++ 14 `std::vector` 模板类的定义。

``` c++
namespace std
{

template <class T, class Allocator = allocator<T> >
class vector
{
public:
    typedef vector                                          __self;
    typedef T                                               value_type;
    typedef Allocator                                       allocator_type;
    typedef allocator_traits<allocator_type>                __alloc_traits;
    typedef value_type&                                     reference;
    typedef const value_type&                               const_reference;
    typedef typename __alloc_traits::size_type              size_type;
    typedef typename __alloc_traits::difference_type        difference_type;
    typedef typename __alloc_traits::pointer                pointer;
    typedef typename __alloc_traits::const_pointer          const_pointer;

private:
    pointer begin_ = nullptr;
    pointer end_ = nullptr;
    pointer end_cap_ = nullptr;
    allocator_type alloc_;
};

}
```

`std::vector` 模板有两个模板参数，一个是元素类型 `T`，一个是空间配置器（allocator）类型 `Allocator`。后者有默认值 `std::allocator<T>`，这是一种无状态的空间配置器。

一个 `std::vector` 对象有 4 个私有数据成员，它们用于管理 vector 私有的数组，以及 vector 所使用的内存配置器。

- `begin_`：指针，指向私有数组的首地址。
- `end_`：指针，指向当前已构造过的数组成员的下一个位置，有 `size() == begin_ - end_`。
- `end_cap_`：指针，指向 vector 对象私有数组的末尾，对应 `capacity() == end_cap_ - begin_`。
- `alloc_`：保存空间配置器。注意，在 libc++ 中，`end_cap_` 和 `alloc_` 两个成员合并存放在一个 `__compressed_pair` 类中，以便在情况允许时消除 `alloc_` 成员所需的空间。

使用无状态的空间配置器时，常见的 C++ 实现都会把存放它所需的数据成员的空间优化掉。
这时有 `sizeof(std::vector<int>) == 24`，即只存放 3 个必需的 8 字节的指针变量，省略掉空间配置器这个成员。
当需要使用空间配置器对象时，所用方法类似于直接默认构造出一个新对象。
<!-- 这是 Empty Base Optimization (EBO)，需要配合模板元编程的效果 -->
