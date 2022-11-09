---
title: 分配堆内存
date: 2019-09-29 09:45:40
tags: [Memory Management]
summary: 这篇文章为我们的内核增加了对堆分配的支持。 首先，它介绍了动态内存，并展示了借用检查器如何防止常见的分配错误。 然后，它实现Rust的基本分配接口，创建一个堆内存区域，并设置一个分配器crate。 在这篇文章的结尾，内置分配crate的所有分配和收集类型将对我们的内核可用。
---

这篇文章为我们的内核增加了对堆分配的支持。 首先，它介绍了动态内存，并展示了借用检查器如何防止常见的分配错误。 然后，它实现Rust的基本分配接口，创建一个堆内存区域，并设置一个分配器crate。 在这篇文章的结尾，内置分配crate的所有分配和收集类型将对我们的内核可用。

该博客在[GitHub上](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/phil-opp/blog_os&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhh6KGm1hLx2QlIFubK_gEmxif3aHg)公开开发。 此文章的完整源代码可以在[`post-10`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/phil-opp/blog_os/tree/post-10&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhijyi0qw7SN36BGwlhAbAJ_nMofiA)分支中找到。

## 局部和静态变量

当前，我们在内核中使用两种类型的变量：局部变量和`static`变量。 局部变量存储在[调用堆栈](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Call_stack&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiDjrmAp1RMPlkX43yI6cMjfgQlEg)中，并且仅在所在的函数返回之前才有效。 静态变量存储在固定的内存位置，并且在程序的整个生命周期中始终有效。

### 局部变量

局部变量存储在[调用堆栈中](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Call_stack&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiDjrmAp1RMPlkX43yI6cMjfgQlEg) ， [调用堆栈](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Call_stack&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiDjrmAp1RMPlkX43yI6cMjfgQlEg)是支持`push`和`pop`操作的[堆栈数据结构](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) 。 在每个函数里，编译器会将被调用函数的参数，返回地址和局部变量压入栈中：

![An outer() and an inner(i: usize) function. Both have some local variables. Outer calls inner(1). The call stack contains the following slots: the local variables of outer, then the argument i = 1, then the return address, then the local variables of inner.](https://os.phil-opp.com/heap-allocation/call-stack.svg)

上面的示例显示了`inner`函数被`outer`函数调用之后的调用堆栈。 我们看到调用堆栈首先包含了`outer`的局部变量。 在`inner`调用中，参数`1`和函数的返回地址被压入栈中。 然后将控制权转移到`inner` ，从而开始压入其局部变量。

`inner`函数返回后，将弹出其调用堆栈的一部分，仅保留`outer`函数的局部变量：

我们看到`inner`的局部变量仅在函数返回之前有效。 当我们使用某个值太久时，例如当我们尝试返回对局部变量的引用时，Rust编译器会强制检查这些生命周期并引发错误：

```rust
fn inner(i: usize) -> &'static u32 {
    let z = [1, 2, 3];
    &z[i]
}
```

（[在Playground中运行示例](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2018%26gist%3D6186a0f3a54f468e1de8894996d12819&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhcWnAktOoWRK84k152ti4sf_VPhQ)）

虽然在此示例中返回引用毫无意义，但在某些情况下，我们希望变量的寿命比函数更长。 当我们尝试[加载中断描述符表](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/cpu-exceptions/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgwRvWr9gSPjeJgzsv7l7wShMNucg#loading-the-idt)并不得不使用`static`变量来延长生存期时，我们已经在内核中看到了这种情况。

### 静态变量

静态变量存储在与堆栈分开的固定内存位置。 链接器在编译时分配了此存储位置，并编码在了可执行文件中。 静态变量在程序的完整运行时中都有效，因此它们具有`'static`生命周期，并且始终可以从局部变量中进行引用：



![The same outer/inner example with the difference that inner has a static Z: [u32; 3] = [1,2,3]; and returns a &Z[i] reference](https://os.phil-opp.com/heap-allocation/call-stack-static.svg)

在上面的示例中，当`inner`函数返回时，它的部分调用堆栈被销毁。静态变量位于一个不会被销毁的单独的内存范围内，因此`&Z[1]`引用在返回后仍然有效。

除了`'static`生命周期之外，静态变量还具有有用的属性：它们的位置在编译时确定，因此不需要引用即可访问它。 我们为`println`宏利用了该属性：通过在内部使用[静态`Writer`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/vga-text-mode/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhjKVQ-rVdxu77flCvGVy5Fx-CA02w#a-global-interface) ，不需要`&mut Writer`引用即可调用该宏，这在我们无法访问任何其他变量的[异常处理程序](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/cpu-exceptions/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgwRvWr9gSPjeJgzsv7l7wShMNucg#implementation)中非常有用。

但是，静态变量的此属性带来一个关键的缺点：默认情况下，它们是只读的。 Rust之所以要这样做是因为，例如，如果两个线程同时修改一个静态变量，则会发生[数据争用](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nomicon/races.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhjYe_MLT6P-PIV2jWQnPjP6WbgrPQ) 。 修改静态变量的唯一方法是将其封装为[`Mutex`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/spin/0.5.2/spin/struct.Mutex.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiPigEeaETYxRVwZpm59ZJfQBqK7w)类型，以确保在任何时间点仅存在一个`&mut`引用。 我们已经为[静态VGA缓冲区`Writer`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/vga-text-mode/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhjKVQ-rVdxu77flCvGVy5Fx-CA02w#spinlocks)使用了`Mutex` 。

## 动态内存

局部变量和静态变量已经非常强大，可以满足大部分场合的要求。 但是，我们看到它们都有局限性：

- 局部变量仅在所在函数或块结束之前有效。 这是因为它们存在于调用堆栈中，并在所在的函数返回后销毁。
- 静态变量在程序运行时始终有效，因此无法在不再需要时回收和重用其内存。 而且，它们的所有权语义不明确，并且可以从所有函数中访问，因此，当我们要修改它们时，需要使用[`Mutex`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/spin/0.5.2/spin/struct.Mutex.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiPigEeaETYxRVwZpm59ZJfQBqK7w)进行保护。

局部和静态变量的另一个限制是它们具有固定的大小。 因此，当添加更多元素时，它们将无法存储动态增长的集合。 （Rust中有一些关于建议使用[未确定](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/rust-lang/rust/issues/48055&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhs9Pf2O2O5WPZ-Xvlj1DibDpm-Rg)大小的[右值](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/rust-lang/rust/issues/48055&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhs9Pf2O2O5WPZ-Xvlj1DibDpm-Rg)的建议，该值允许动态大小的局部变量，但它们仅在某些特定情况下有效。）

为了避免这些缺点，编程语言通常支持第三个内存区域：**堆 **来存储变量。 堆通过两个称为`allocate`和`deallocate`函数在运行时支持*动态内存分配* 。 它以以下方式工作： `allocate`函数返回指定大小的可用内存块，可用于存储变量。 然后，该变量将一直存在，直到通过调用对该变量的引用的`deallocate`函数将其`deallocate`为止。

让我们来看一个例子：

![The inner function calls allocate(size_of([u32; 3])), writes z.write([1,2,3]);, and returns (z as *mut u32).offset(i). The outer function does deallocate(y, size_of(u32)) on the returned value y.](https://os.phil-opp.com/heap-allocation/call-stack-heap.svg)

在这里， `inner`函数使用堆内存而不是静态变量来存储`z` 。 它首先分配所需大小的内存块，然后返回`*mut u8` [原始指针](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgxBivPXeHGd4Lv7ZRfhxcOwPp25w#dereferencing-a-raw-pointer) 。 然后，它使用[`ptr::write`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/ptr/fn.write.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhGBOZaaa9afSHJ_T9_KNkhBtR70w)方法将数组`[1,2,3]`写入其中。 在最后一步中，它使用[`offset`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/std/primitive.pointer.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhjoDpBUUAFPMkakB2aev8GBAmlz1A#method.offset)函数计算指向第`i`个元素的指针，然后将其返回。 （请注意，为简洁起见，我们在此示例函数中省略了一些必需的强制转换和unsafe块。）

分配的内存将一直存在，直到通过调用`deallocate`显式释放它为止。 因此，即使在`inner`返回并且调用栈的一部分被销毁之后，返回的指针仍然有效。 与静态内存相比，使用堆内存的优势在于，可以在释放内存后重用内存，这是通过`deallocate`中的`deallocate`调用实现的。 调用之后，情况如下：

![The call stack contains the local variables of outer, the heap contains z[0] and z[2], but no longer z[1].](https://os.phil-opp.com/heap-allocation/call-stack-heap-freed.svg)

我们看到`z[1]`的位置上又是空闲的了，可以重新用于下一个`allocate`调用。 但是，我们也看到`z[0]`和`z[2]`没有被释放，因为我们从未释放过它们。 这种错误称为*内存泄漏* ，通常是导致程序过度消耗内存的原因（试想一下，当我们在循环中重复调用`inner`时会发生什么）。 这看起来很糟糕，但是动态分配还可能会发生更多危险的错误类型。

### 常见错误

除了不幸的但不会使程序容易受到攻击的内存泄漏外，还有两种常见的错误类型，其后果更为严重：

- 当我们在调用`deallocate`后意外地继续使用变量时，我们有一个所谓的**use-after-free**漏洞。这样的错误会导致未定义的行为，并且攻击者经常会利用它执行任意代码。
- 当我们不小心两次释放变量时，我们就有一个**double-free**漏洞。它可能在第一次`deallocate`调用之后释放在同一位置分配的另一个变量。 因此，它可能导致use-after-free漏洞。

这些类型的漏洞是众所周知的，因此人们可能会期望大家现在已经学会了如何避免它们。 但是，他们并没有，仍然经常发现此类漏洞，例如Linux中最近发现的[use-after-free漏洞](https://securityboulevard.com/2019/02/linux-use-after-free-vulnerability-found-in-linux-2-6-through-4-20-11/)允许任意代码执行。 这表明即使是最好的程序员也不一定总是能够正确处理复杂项目中的动态内存。

为了避免这些问题，许多语言（例如Java或Python）都使用称为[*垃圾回收*](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))的技术自动管理动态内存。 这种方法让程序员永远不要手动调用`deallocate` 。 而是定期暂停程序并扫描未使用的堆变量，然后将它们自动释放。 因此，上述漏洞永远不会发生。 缺点是常规扫描的性能开销以及可能较长的暂停时间。

Rust针对此问题采用了不同的方法：它使用一种称为[*所有权*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiXg-xhyAHe5PuaeLsNN8Nsck3u9g)的概念，该概念能够在编译时检查动态内存操作的正确性。 因此，不需要垃圾收集来避免提到的漏洞，这意味着没有性能开销。 这种方法的另一个优点是，程序员仍然可以像使用C或C ++一样对动态内存的使用进行细粒度的控制。

### Rust中的动态内存分配

Rust标准库提供了抽象类型来隐式调用这些函数，而不是让程序员手动调用`allocate`和`deallocate` 。 最重要的类型是[**Box**](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/std/boxed/index.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgydQDRRu9l94xRKsKXfpC58IaGQQ) ，它是堆分配值的抽象。 它提供了一个[`Box::new`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/boxed/struct.Box.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiALFCmMq4SQmt-_lR3sbOud4Zh8g#method.new)构造函数，该函数接受一个值，使用该值的大小调用`allocate` ，然后将该值移动到堆上新分配的插槽中。 为了再次释放堆内存， `Box`类型实现了[`Drop`特性](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch15-03-drop.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiheLdRjZNyOmgfBOzki5tmpGGmEA)以在超出范围时调用`deallocate` ：

```rust
{
    let z = Box::new([1,2,3]);
    […]
} // z goes out of scope and `deallocate` is called
```

此模式有一个奇怪的名称， [*资源获得即初始化*](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization) （或简称*RAII* ）。 它起源于C++，用于实现类似的抽象类型[`std::unique_ptr`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.cppreference.com/w/cpp/memory/unique_ptr&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiqCXA9Qx4dEIWJLDL41o5wPIwCJg) 。

单靠这种类型不足以防止所有的use-after-free漏洞，因为在`Box`超出范围并释放相应的堆内存空间之后，程序员仍然可以保留对内容的引用：

```rust
let x = {
    let z = Box::new([1,2,3]);
    &z[1]
}; // z goes out of scope and `deallocate` is called
println!("{}", x);
```

这就是Rust所有权机制起作用的地方。它为每个引用分配了抽象[生命周期](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhheK8nR0_Zz4xow84YA4SkVmwPdbg) ，这是该引用有效的范围。 在上面的示例中， `x`引用是从`z`数组中获取的，因此在`z`超出范围后，它将变为无效。 在[Playgroud中运行上述示例时](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=28180d8de7b62c6b4a681a7b1f745a48)，您会看到Rust编译器确实报告了错误：

```shell
error[E0597]: `z[_]` does not live long enough
 --> src/main.rs:4:9
  |
2 |     let x = {
  |         - borrow later stored here
3 |         let z = Box::new([1,2,3]);
4 |         &z[1]
  |         ^^^^^ borrowed value does not live long enough
5 |     }; // z goes out of scope and `deallocate` is called
  |     - `z[_]` dropped here while still borrowed
```

一开始，这个错误提示可能会有些混乱。 创建一个值的引用被称为*借用*，因为它类似于现实生活中的借用：您可以临时访问某个对象，但需要在某个时候将其返回，并且不得销毁它。 通过检查所有借用在对象被销毁之前是否已结束，Rust编译器可以保证不会发生use-after-free。

Rust的所有权系统更进了一步，它不仅可以防止使用后使用的错误，而且可以像Java或Python这样的垃圾收集语言提供完全的[*内存安全性*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Memory_safety&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiUSIOCdxW0QA2oZubgSMT-wZKM7Q) 。 另外，它保证[*线程安全*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Thread_safety&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgl5oGZfFqc51TdoKgVrffZcKjSfg) ，Rust代码在多线程环境下比那些语言更安全。 最重要的是，所有这些检查都在编译时进行，因此与C中的手动内存管理相似，它没有运行时开销。

### 用例

现在我们知道Rust中动态内存分配的基础知识，但是应该什么时候使用它呢？ 没有动态内存分配的内核已经走得很远了，那么为什么现在需要它呢？

首先，动态内存分配总是会带来一些性能开销，因为我们需要为每个分配在堆上找到一个空闲空间。 因此，通常最好使用局部变量，尤其是在性能敏感的内核代码中。 但是，在某些情况下，动态内存分配是最佳选择。

基本规则是，具有动态生存期或可变大小的变量需要动态内存。 动态生存期最重要的类型是[**Rc**](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/rc/index.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhOvikyjnB-QTtPny0gSuSLpF9Tkw) ，它对其包装值的引用进行计数，并在所有引用离开作用域后将其释放。另外一些具有可变大小的类型的示例包括[**Vec**](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/vec/index.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhHCKQLyRM5nVe02J_5GfW_hJSj_g) ， [**String**](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/string/index.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhxFOZSUCPDiU1gZFAW_5Q5bksW7A)等[集合类型](https://doc.rust-lang.org/alloc/collections/index.html) ，这些[类型](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/index.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhihRW6JMQDicWS77B_50faSnHwSug)会在添加更多元素时动态增长。 这些类型的工作方式是在它们装满时重新分配一块更大的内存，将所有元素复制过来，然后取消掉旧的分配。

对于我们的内核，我们最需要的是集合类型，例如，在以后的帖子中实现多任务处理时用于存储活动任务的列表。

## 分配器接口

实现堆分配器的第一步是添加对内置`alloc`crate的依赖。 与[`core`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhiAG6dIvJgsOsusTaEqJeByp29mQA)crate一样，它是标准库的子集，另外还包含分配和集合类型。 为了添加对`alloc`的依赖，我们将以下内容添加到我们的`lib.rs`：

```rust
// in src/lib.rs

extern crate alloc;
```

与其他依赖不同，我们不需要修改`Cargo.toml` 。 原因是`alloc`crate与Rust编译器一起作为标准库的一部分提供，因此我们只需要启用它即可。 这就是这个`extern crate`语句的作用。（历史上，所有依赖项都需要一个`extern crate`语句，该语句现在是可选的）。

由于我们是在编译一个特定的目标，因此我们不能使用Rust安装时预编译版本的`alloc`。取而代之的是，我们可以把`alloc`加入到位于`.cargo/config.toml`文件的`unstable.build-std`数组中，实现使cargo通过源代码来重新编译crate。

```rust
# in .cargo/config.toml

[unstable]
build-std = ["core", "compiler_builtins", "alloc"]
```

现在，编译器将重新编译并在我们的内核中包含alloc crate。

`#[no_std]`默认禁用了`alloc`crate，其原因是它还有其他要求。 现在尝试编译项目时，我们可以从错误中提示看到这些要求：

```shell
error: no global memory allocator found but one is required; link to std or add
       #[global_allocator] to a static item that implements the GlobalAlloc trait.

error: `#[alloc_error_handler]` function required, but not found
```

发生第一个错误是因为分配箱需要堆分配器，该堆分配器是提供`allocate`和`deallocate`功能的对象。 在Rust中，堆分配器由[`GlobalAlloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg)特性描述，该特性在错误消息中提到。 要为crate设置堆分配器，必须将`#[global_allocator]`属性应用于实现`GlobalAlloc` trait的`static`变量。

发生第二个错误是因为`allocate`调用可能失败，通常是在没有更多可用内存时失败。 我们的程序必须能够对这种情况做出反应，这就是`#[alloc_error_handler]`函数的作用。

在以下各节中，我们将详细描述这些特征和属性。

### `GlobalAlloc`Trait

[`GlobalAlloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg) Trait 定义了堆分配器必须提供的功能。 该Trait很特殊，因为程序员几乎从不直接使用它。 相反，当使用`alloc`分配内存和使用集合类型时，编译器将自动向trait中的方法插入适当的调用。

由于我们将需要为我们的分配器类型实现trait，因此有必要仔细研究其声明：

```rust
pub unsafe trait GlobalAlloc {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);

    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 { ... }
    unsafe fn realloc(
        &self,
        ptr: *mut u8,
        layout: Layout,
        new_size: usize
    ) -> *mut u8 { ... }
}
```

它定义了两个必需的方法[`alloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhixPH85ItX6ssyPAbPLnxIX8ZIpXA)和[`dealloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#tymethod.dealloc) ，它们与我们在示例中使用的`allocate`和`deallocate`函数相对应：

- [`alloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhixPH85ItX6ssyPAbPLnxIX8ZIpXA)方法将[`Layout`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/struct.Layout.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgfbVGb4vhKLhix2Ocb5VA9y29OfA)实例作为参数，该实例描述分配的内存应具有的所需大小和对齐方式。 它返回[原始指针](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgxBivPXeHGd4Lv7ZRfhxcOwPp25w#dereferencing-a-raw-pointer) ， [指向](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhgxBivPXeHGd4Lv7ZRfhxcOwPp25w#dereferencing-a-raw-pointer)分配的内存块的第一个字节。 `alloc`方法返回空指针而非显式的错误值以指示分配错误。 这有点令人不习惯，但是它的优点是包装现有的系统分配器很容易，因为它们使用相同的调用约定。
- 对应的有[`dealloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#tymethod.dealloc)方法，负责释放内存块。 它接收两个参数，一个是`alloc`返回的指针， `alloc`是用于分配的`Layout` 。

该特征还使用默认实现定义了两个方法[`alloc_zeroed`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#method.alloc_zeroed)和[`realloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#method.realloc) ：

- [`alloc_zeroed`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#method.alloc_zeroed)方法等效于调用`alloc` ，然后将分配的内存块设置为零，这正是提供的默认实现所执行的。 如果可能的话，分配器实现可以使用更有效的自定义实现来覆盖默认实现。
- [`realloc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhja9Cla59NwkTNZAMz6PcSj4jN6wg#method.realloc)方法允许增加或减少分配。 默认实现分配一个具有所需大小的新内存块，并复制先前分配的所有内容。 同样，分配器实现可能可以提供此方法的更有效实现，例如，如果可能的话，通过就地扩展/缩小分配。

#### Unsafe

要注意的一件事是trait本身和所有trait方法都被声明为`unsafe` ：

- 将该特征声明为`unsafe`的原因是，程序员必须保证分配器类型的特征实现是正确的。 例如， `alloc`方法绝不能返回已在其他地方使用的内存块，因为这将导致未定义的行为。
- 同样，方法`unsafe`的原因是，调用方在调用方法时必须确保各种不变性，例如，传递给`alloc`的`Layout`指定一个非零大小。 在实践中，这实际上并不重要，因为方法通常是由编译器直接调用的，这可以确保满足要求。

### `DummyAllocator`

既然我们知道应该提供什么分配器类型，我们就可以创建一个简单的虚拟分配器。 为此，我们创建一个新的`allocator`模块：

```rust
// in src/lib.rs

pub mod allocator;
```

我们的DummyAllocator会尽最大的努力来实现Trait，并在调用`alloc`时始终返回错误。 看起来像这样：

```rust
// in src/allocator.rs

use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr::null_mut;

pub struct Dummy;

unsafe impl GlobalAlloc for Dummy {
    unsafe fn alloc(&self, _layout: Layout) -> *mut u8 {
        null_mut()
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        panic!("dealloc should be never called")
    }
}
```

该结构不需要任何字段，因此我们将其创建[为零大小类型](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nomicon/exotic-sizes.html&xid=25657,15700023,15700186,15700190,15700256,15700259,15700262,15700265,15700271&usg=ALkJrhhjDn4Dp1jvubRT6ctCKEw2rj_gYA#zero-sized-types-zsts) 。 如上所述，我们总是从`alloc`返回空指针，它对应于分配错误。 由于分配器从不返回任何内存，因此永远不会发生对`dealloc`的调用。 因此，我们在`dealloc`方法被调用时panic。 `alloc_zeroed`和`realloc`方法具有默认实现，因此我们无需为其提供实现。

现在我们有了一个简单的分配器，但是我们仍然必须告诉Rust编译器它应该使用这个分配器。这时`#[global_allocator]`就有用了。

### #`[global_allocator]`属性

`#[global_allocator]`属性告诉Rust编译器应该使用哪个分配器实例作为全局堆分配器。 该属性仅适用于实现`GlobalAlloc`特性的`static` 。 让我们将`Dummy`分配器的一个实例注册为全局分配器：

```rust
// in src/lib.rs

#[global_allocator]
static ALLOCATOR: allocator::Dummy = allocator::Dummy;
```

由于`Dummy`分配器是[零大小的类型](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nomicon/exotic-sizes.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgeU-TEc9SDEbOIraxGeQ8xYg4r3A#zero-sized-types-zsts) ，因此我们不需要在初始化表达式中指定任何字段。 请注意， `#[global_allocator]`模块[不能在子模块中使用](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/rust-lang/rust/pull/51335&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiRtm-6gsZO_3uOw-LJ9wMyI-a1kA) ，因此我们需要将其放入`lib.rs` 。

现在，当我们尝试编译它时，第一个错误应该消失了。 让我们修复剩余的第二个错误：

```rust
error: `#[alloc_error_handler]` function required, but not found
```

### `#[alloc_error_handler]`属性

正如我们在讨论`GlobalAlloc`特性时所了解的那样， `alloc`函数可以通过返回空指针来指示分配错误。 问题是：Rust运行时应如何应对这种分配失败？ 这是`#[alloc_error_handler]`属性的来源。它指定发生分配错误时调用的函数，类似于在发生panic时调用panic处理程序的方式。

让我们添加这样的函数来修复编译错误：

```rust
// in src/lib.rs

#![feature(alloc_error_handler)] // at the top of the file

#[alloc_error_handler]
fn alloc_error_handler(layout: alloc::alloc::Layout) -> ! {
    panic!("allocation error: {:?}", layout)
}
```

`alloc_error_handler`函数仍然不稳定，因此我们需要一个特性设置来启用它。 该函数接收一个参数：发生分配失败时传递给`alloc`的`Layout`实例。 我们对解决该错误无能为力，因此我们只是发送包含`Layout`实例的panic消息。

加上这个函数之后，编译错误应该已经被修复了。 现在我们可以使用`alloc`的分配和收集类型，例如，我们可以使用[`Box`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/boxed/struct.Box.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiosxpeNyAEHn8OqRYErq2bnWWKRw)在堆上分配一个值：

```rust
// in src/main.rs

extern crate alloc;

use alloc::boxed::Box;

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] print "Hello World!", call `init`, create `mapper` and `frame_allocator`

    let x = Box::new(41);

    // […] call `test_main` in test mode

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

请注意，我们也需要在`main.rs`指定`extern crate alloc`语句。 这是必需的，因为`main.rs`和`main.rs`部分被视为单独的crate。 但是，我们不需要创建另一个静态`#[global_allocator]`，因为全局分配器适用于项目中的所有crate。 实际上，在另一个crate中指定其他分配器将是错误的。

运行上面的代码时，我们看到我们的`alloc_error_handler`函数被调用：![QEMU printing "panicked at `allocation error: Layout { size_: 4, align_: 4 }, src/lib.rs:89:5"](https://os.phil-opp.com/heap-allocation/qemu-dummy-output.png)

调用错误处理程序是因为`Box::new`函数隐式调用了全局分配器的`alloc`函数。 我们的虚拟分配器始终返回空指针，因此每次分配都会失败。 为了解决这个问题，我们需要创建一个实际上返回可用内存的分配器。

## 创建内核堆

在创建合适的分配器之前，我们首先需要创建一个堆内存区域，分配器可以从中分配内存。 为此，我们需要为堆区域定义一个虚拟内存范围，然后将该区域映射到物理帧。 有关虚拟内存和页表的概述，请参见[*“分页简介”*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/paging-introduction/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhkanSS8TkxwpQrLCkMYDffTLWobg) 。

第一步是为堆定义虚拟内存区域。 我们可以选择所需的任何虚拟地址范围，只要它尚未用于其他内存区域即可。 让我们将其定义为从地址`0x_4444_4444_0000`开始的内存，以便稍后可以轻松识别堆指针：

```rust
// in src/allocator.rs

pub const HEAP_START: usize = 0x_4444_4444_0000;
pub const HEAP_SIZE: usize = 100 * 1024; // 100 KiB
```

我们现在将堆大小设置为100 KiB。 如果将来需要更多空间，我们可以简单地增加它。

如果我们现在尝试使用此堆区域，则会发生页面错误，因为虚拟内存区域尚未映射到物理内存。为了解决这个问题，我们创建了一个`init_heap`函数，该函数使用我们在[*“分页实现”一文中*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/paging-implementation/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhj_xC5St0tlJnPmBv9hvbAEPd6W5A)介绍的[`Mapper` API](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/paging-implementation/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhj_xC5St0tlJnPmBv9hvbAEPd6W5A#using-mappedpagetable)映射堆页面：

```rust
// in src/allocator.rs

use x86_64::{
    structures::paging::{
        mapper::MapToError, FrameAllocator, Mapper, Page, PageTableFlags, Size4KiB,
    },
    VirtAddr,
};

pub fn init_heap(
    mapper: &mut impl Mapper<Size4KiB>,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) -> Result<(), MapToError> {
    let page_range = {
        let heap_start = VirtAddr::new(HEAP_START as u64);
        let heap_end = heap_start + HEAP_SIZE - 1u64;
        let heap_start_page = Page::containing_address(heap_start);
        let heap_end_page = Page::containing_address(heap_end);
        Page::range_inclusive(heap_start_page, heap_end_page)
    };

    for page in page_range {
        let frame = frame_allocator
            .allocate_frame()
            .ok_or(MapToError::FrameAllocationFailed)?;
        let flags = PageTableFlags::PRESENT | PageTableFlags::WRITABLE;
        unsafe { mapper.map_to(page, frame, flags, frame_allocator)?.flush() };
    }

    Ok(())
}
```

该函数通过使用[`Size4KiB`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/page/enum.Size4KiB.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhg2M_G0Ol9na-mTvgYz9wcyhGSTxg)作为模版参数来获取对[`Mapper`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/trait.Mapper.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgVFv5BTMRwDVcKqwVIbnPTvsmTPg)和[`FrameAllocator`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/trait.FrameAllocator.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjggS52v2i2ScjsxhDb74n9oOyHIA)实例的可变引用，它们均限于4KiB页。 该函数的返回值是一个[`Result`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/result/enum.Result.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhce2hBUz9y9tfOupW--gctQJa0tQ) ，其[`Result`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/result/enum.Result.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhce2hBUz9y9tfOupW--gctQJa0tQ)成功时为单位类型`()`，而出错时为[`MapToError`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/enum.MapToError.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhicxFwlIHtXfyNzcvL79bpGKF_HjA)，这是[`Mapper::map_to`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/trait.Mapper.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgVFv5BTMRwDVcKqwVIbnPTvsmTPg#tymethod.map_to)方法返回的错误类型。 在这里重用错误类型是有意义的，因为`map_to`方法是此函数中错误的主要来源。

实现可以分为两部分：

- **创建页面范围：**要创建我们要映射的页面范围，我们将`HEAP_START`指针转换为[`VirtAddr`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/struct.VirtAddr.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhNOHf3zU7AgNVZhjYJkAVQ39Xufw)类型。 然后，我们通过添加`HEAP_SIZE`从中计算出堆结束地址。 我们需要一个包含性的边界（堆最后一个字节的地址），因此我们减去1。接下来，我们使用[`containing_address`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/page/struct.Page.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjSe7J0VwXNgWPo3i7TiwHf8vSwFw#method.containing_address)函数将地址转换为[`Page`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/page/struct.Page.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjSe7J0VwXNgWPo3i7TiwHf8vSwFw)类型。 最后，我们使用[`Page::range_inclusive`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/page/struct.Page.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjSe7J0VwXNgWPo3i7TiwHf8vSwFw#method.range_inclusive)函数从起始页面和结束页面创建页面范围。
- **映射页面：**第二步是映射我们刚刚创建的页面范围的所有页面。 为此，我们使用`for`循环遍历该范围内的页面。 对于每个页面，我们执行以下操作：
  - 该页应该被映射到我们使用[`FrameAllocator::allocate_frame`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/trait.FrameAllocator.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjggS52v2i2ScjsxhDb74n9oOyHIA#tymethod.allocate_frame)方法分配的物理帧上。 当没有剩余的帧时，此方法将返回[`None`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/option/enum.Option.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhaJ_Il3_-qAEhTV0CU_eroK_zIqA#variant.None) 。 我们通过使用[`Option::ok_or`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/option/enum.Option.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhaJ_Il3_-qAEhTV0CU_eroK_zIqA#method.ok_or)方法将其映射到[`MapToError::FrameAllocationFailed`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/enum.MapToError.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhicxFwlIHtXfyNzcvL79bpGKF_HjA#variant.FrameAllocationFailed)错误来处理没有剩余帧的情况，并应用[问号运算符](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhg1cg548fqAFT-Y_9h9-bsEfN4Qpw)以在出现错误的情况下尽早返回。
  - 我们为页面设置了必需的`PRESENT`标志和`WRITABLE`标志。 使用这些标志，允许读取和写入访问，这对于堆内存是有意义的。
  - 我们使用不安全的[`Mapper::map_to`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/trait.Mapper.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgVFv5BTMRwDVcKqwVIbnPTvsmTPg#tymethod.map_to)方法在活动页面表中创建映射。 该方法可能会失败，因此我们再次使用[问号运算符](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/edition-guide/rust-2018/error-handling-and-panics/the-question-mark-operator-for-easier-error-handling.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhg1cg548fqAFT-Y_9h9-bsEfN4Qpw)将错误转发给调用方。 成功后，该方法将返回一个[`MapperFlush`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/struct.MapperFlush.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhu0tpL-XJy7I4bFlQNnOuMtOYO7g)实例，我们可以使用该实例使用[`flush`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/struct.MapperFlush.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhu0tpL-XJy7I4bFlQNnOuMtOYO7g#method.flush)方法来更新[*转换后备缓冲区*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/paging-introduction/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhkanSS8TkxwpQrLCkMYDffTLWobg#the-translation-lookaside-buffer)。

最后一步是从我们的`kernel_main`调用此函数：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::allocator; // new import
    use blog_os::memory::{self, BootInfoFrameAllocator};

    println!("Hello World{}", "!");
    blog_os::init();

    let mut mapper = unsafe { memory::init(boot_info.physical_memory_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };

    // new
    allocator::init_heap(&mut mapper, &mut frame_allocator)
        .expect("heap initialization failed");

    let x = Box::new(41);

    // […] call `test_main` in test mode

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

我们在这里出示了完整的代码，为了便于参考上下文。 仅有的新行是`blog_os::allocator`导入和对`allocator::init_heap`函数的调用。 万一`init_heap`函数返回错误，我们会使用[`Result::expect`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/result/enum.Result.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhce2hBUz9y9tfOupW--gctQJa0tQ#method.expect)方法来引发panic，因为当前没有明智的方法来处理此错误。

现在，我们有一个准备使用的映射堆内存区域。 `Box::new`调用仍然使用我们旧的`Dummy`分配器，因此运行它时，您仍然会看到“内存不足”错误。 让我们通过使用适当的分配器来解决此问题。

## 使用分配器 crate

由于实现分配器有些复杂，因此我们首先使用外部分配器crate。 在下一篇文章中，我们将学习如何实现自己的分配器。

`no_std`应用程序的一个简单分配器箱是[`linked_list_allocator`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://github.com/phil-opp/linked-list-allocator/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjRQdwoIoMwukn1dxQxvPbEk9FaCw)箱子。 它起这个名称是因为：它使用链接列表数据结构来跟踪释放的内存区域。 有关此方法的详细说明，请参见下一篇文章。

要使用板条箱，我们首先需要在`Cargo.toml`添加对它的依赖：

```rust
# in Cargo.toml

[dependencies]
linked_list_allocator = "0.6.4"
```

然后，我们可以用crate提供的分配器替换我们的dummy分配器：

```rust
// in src/lib.rs

use linked_list_allocator::LockedHeap;

#[global_allocator]
static ALLOCATOR: LockedHeap = LockedHeap::empty();
```

该结构被命名为`LockedHeap`因为，它使用[`spin::Mutex`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/spin/0.5.2/spin/struct.Mutex.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiB-xAcS86h4-YyKqhs0T_npkoGBQ)进行同步。 这是必需的，因为多个线程可以同时访问`ALLOCATOR`静态对象。 一如往常，在使用`Mutex`时 ，我们需要注意不要意外导致死锁。 这意味着我们不应该在中断处理程序中执行任何分配，因为它们可以在任意时间运行，并且可能会中断正在进行的分配。

仅将`LockedHeap`设置为全局分配器是不够的。 原因是我们使用了[`empty`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.LockedHeap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjSImcHYQIFUsXh6pW843hrmoYKmQ#method.empty)构造函数，该函数创建了一个没有任何后备内存的分配器。 就像我们的虚拟分配器一样，它总是在`alloc`上返回错误。为了解决这个问题，我们需要在创建堆之后初始化分配器：

```rust
// in src/allocator.rs

pub fn init_heap(
    mapper: &mut impl Mapper<Size4KiB>,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) -> Result<(), MapToError<Size4KiB>> {
    // […] map all heap pages to physical frames

    // new
    unsafe {
        super::ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE);
    }

    Ok(())
}
```

我们使用[`LockedHeap::lock`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.LockedHeap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjSImcHYQIFUsXh6pW843hrmoYKmQ#method.lock)方法获取对包装后的[`Heap`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgOQD-NmzuyehtvmIr_FZz2Unvq2A)实例的排他性引用，然后在该实例上以堆边界作为参数调用[`init`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgOQD-NmzuyehtvmIr_FZz2Unvq2A#method.init)方法。 重要的是在映射堆页面*之后*初始化堆，因为[`init`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgOQD-NmzuyehtvmIr_FZz2Unvq2A#method.init)函数会尝试写入堆内存。

初始化堆之后，我们现在可以使用内置alloc crate中的的所有分配和集合类型，而不会出现错误：

```rust
// in src/main.rs

use alloc::{boxed::Box, vec, vec::Vec, rc::Rc};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialize interrupts, mapper, frame_allocator, heap

    // allocate a number on the heap
    let heap_value = Box::new(41);
    println!("heap_value at {:p}", heap_value);

    // create a dynamically sized vector
    let mut vec = Vec::new();
    for i in 0..500 {
        vec.push(i);
    }
    println!("vec at {:p}", vec.as_slice());

    // create a reference counted vector -> will be freed when count reaches 0
    let reference_counted = Rc::new(vec![1, 2, 3]);
    let cloned_reference = reference_counted.clone();
    println!("current reference count is {}", Rc::strong_count(&cloned_reference));
    core::mem::drop(reference_counted);
    println!("reference count is {} now", Rc::strong_count(&cloned_reference));

    // […] call `test_main` in test context
    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

此代码示例演示[`Box`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/boxed/struct.Box.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiosxpeNyAEHn8OqRYErq2bnWWKRw) ， [`Vec`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/vec/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhieY-F6KPuz5zi1aQC51lYwWTpocQ)和[`Rc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/rc/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhhRszVAQpT964_IqTwW-N23ixY3Og)类型的某些用法。对于`Box`和`Vec`类型，我们使用[`{:p}`格式说明符](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/fmt/trait.Pointer.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhi8JAgOHJJ7R4OMg1lddXKPPImIfg)打印基础堆指针。为了进行展示`Rc`，我们创建了一个引用计数的堆值，并使用该[`Rc::strong_count`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/rc/struct.Rc.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgdb8ySpOxHWtrwxFWLY2BRF6R_0Q#method.strong_count)函数在删除实例之前和之后（使用[`core::mem::drop`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/mem/fn.drop.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhj1lgzbQAChTDPbzBUXrB1rjwXYkQ)）打印当前引用计数。

运行它时，我们看到以下内容：

![QEMU在0x444444440000打印vec在0x4444444408000处的heap_value当前引用计数为2引用计数现在为1](https://os.phil-opp.com/heap-allocation/qemu-alloc-showcase.png)

如预期的那样，我们看到`Box` 和 `Vec`值存在于堆中，如以开头的指针所指示`0x_4444_4444`。引用计数值也表现出预期的效果，在`clone`调用之后，引用计数为2，在删除一个实例之后，引用计数再次为1。

向量从`0x800`开始的原因不是装箱的值是`0x800`字节大的，而是向量需要增加其容量时发生的[重新分配](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/vec/struct.Vec.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhggQTZld9-J9QlCghMnx6mS6Iv6Yw#capacity-and-reallocation)。例如，当向量的容量为32且我们尝试添加下一个元素时，向量将在幕后分配容量为64的新后备数组，并将所有元素复制到其后。然后释放旧分配。

当然`alloc`，我们现在可以在内核中使用所有更多的分配和收集类型，包括：

- 线程安全引用计数的指针 [`Arc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/sync/struct.Arc.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhj0W9GHTbExBe7ogcC-At7IwnbvzA)
- 有所有权的字符串类型[`String`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/string/struct.String.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjCj3aWKrmBGjI8tSiHyAR6AM07kw)和[`format!`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/macro.format.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgROMRJHZB_3JXG5v6BfGOFfyxgeQ)宏
- [`LinkedList`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/linked_list/struct.LinkedList.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgxe05cnRkWpSbYK8E95OQ9Dr16uQ)
- 可成长的环形缓冲区 [`VecDeque`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/vec_deque/struct.VecDeque.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhjMDKrGsPBTUoy8J-dzcNlJ9_icLA)
- [`BinaryHeap`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/binary_heap/struct.BinaryHeap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiLc9dwZt2-i1zsmCUyeP0MaFeI3Q)优先队列
- [`BTreeMap`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhg_SwcL9IDCpGKpEg1N6gV9Voruxw) 和 [`BTreeSet`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/btree_set/struct.BTreeSet.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhix7vWfraqRdLrcIm8JC7VWF8p5ZA)

当我们要实现线程列表，调度队列或支持async/await时，这些类型将变得非常有用。 

## 添加测试

为了确保我们不会意外破坏新的分配代码，我们应该为其添加集成测试。我们首先创建一个`tests/heap_allocation.rs`具有以下内容的新文件：

```rust
// in tests/heap_allocation.rs

#![no_std]
#![no_main]
#![feature(custom_test_frameworks)]
#![test_runner(blog_os::test_runner)]
#![reexport_test_harness_main = "test_main"]

extern crate alloc;

use bootloader::{entry_point, BootInfo};
use core::panic::PanicInfo;

entry_point!(main);

fn main(boot_info: &'static BootInfo) -> ! {
    unimplemented!();
}

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    blog_os::test_panic_handler(info)
}
```

我们重用了`lib.rs`中的`test_runner`和`test_panic_handler`。由于我们要测试内存分配，因此通过`extern crate alloc`语句启用了这个crate。有关测试样板的更多信息，请查看“ [*测试”*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/testing/&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhie76olh4ei1lGqtBZIyplpwxv8qg)文章。

该`main`函数的实现如下所示：

```rust
// in tests/heap_allocation.rs

fn main(boot_info: &'static BootInfo) -> ! {
    use blog_os::allocator;
    use blog_os::memory::{self, BootInfoFrameAllocator};
    use x86_64::VirtAddr;

    blog_os::init();
    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };
    allocator::init_heap(&mut mapper, &mut frame_allocator)
        .expect("heap initialization failed");

    test_main();
    loop {}
}
```

它与我们`main.rs`中的`kernel_main`函数非常相似，不同之处在于我们不调用`println`，不包含任何示例分配以及无条件地调用`test_main`。

现在我们准备添加一些测试用例。首先，我们添加一个测试，该测试使用进行简单分配[`Box`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/boxed/struct.Box.html&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhiosxpeNyAEHn8OqRYErq2bnWWKRw)并检查分配的值，以确保基本分配有效：

```rust
// in tests/heap_allocation.rs

use blog_os::{serial_print, serial_println};
use alloc::boxed::Box;

#[test_case]
fn simple_allocation() {
    serial_print!("simple_allocation... ");
    let heap_value = Box::new(41);
    assert_eq!(*heap_value, 41);
    serial_println!("[ok]");
}
```

最重要的是，此测试可验证没有发生分配错误。 

接下来，我们迭代构建一个大向量，以测试大量分配和多次分配（由于重新分配）：

```rust
// in tests/heap_allocation.rs

use alloc::vec::Vec;

#[test_case]
fn large_vec() {
    serial_print!("large_vec... ");
    let n = 1000;
    let mut vec = Vec::new();
    for i in 0..n {
        vec.push(i);
    }
    assert_eq!(vec.iter().sum::<u64>(), (n - 1) * n / 2);
    serial_println!("[ok]");
}
```

我们通过与[第n部分和](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/1_%2B_2_%2B_3_%2B_4_%2B_%E2%8B%AF&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgV_rmHr53cE-UooOstgDtXajdMwA#Partial_sums)的公式进行比较来验证[和](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/1_%2B_2_%2B_3_%2B_4_%2B_%E2%8B%AF&xid=17259,15700022,15700186,15700191,15700256,15700259,15700262,15700265&usg=ALkJrhgV_rmHr53cE-UooOstgDtXajdMwA#Partial_sums)。这使我们确信分配的值都是正确的。

作为第三项测试，我们依次创建一万个分配：

```rust
// in tests/heap_allocation.rs

#[test_case]
fn many_boxes() {
    serial_print!("many_boxes... ");
    for i in 0..10_000 {
        let x = Box::new(i);
        assert_eq!(*x, i);
    }
    serial_println!("[ok]");
}
```

此测试可确保分配器将释放的内存重新用于后续分配，因为否则分配器将耗尽内存。这似乎是对分配器的明显要求，但是有些分配器设计没有这样做。下一篇文章中将展示一个做不到这一点的不好的分配器设计。

让我们运行新的集成测试：

```rust
> cargo xtest --test heap_allocation
[…]
Running 3 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
```

## 总结

这篇文章介绍了动态内存，并解释了为什么以及在什么地方需要它。我们了解了Rust的借用检查器如何防止常见漏洞，并了解了Rust的内存分配API的工作方式。

在使用虚拟分配器创建了Rust分配器接口的最小实现之后，我们为内核创建了一个适当的堆内存区域。为此，我们定义的堆的虚拟地址范围，然后使用该映射范围内的物理帧的所有页面`Mapper`，并`FrameAllocator`从以前的帖子。

最后，我们添加了对`linked_list_allocator`板条箱的依赖，以向内核添加适当的分配器。有了这个分配器，我们能够使用`Box`，`Vec`并从其他分配和集合类型`alloc` crate。

## 接下来？

尽管我们已经在本文中添加了堆分配支持，但我们将大部分工作交给了`linked_list_allocator` crate。下一篇文章将详细显示如何从头开始实现分配器。它将介绍多种可能的分配器设计，展示如何实现它们的简单版本，并说明其优缺点。