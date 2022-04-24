---
title: 内存分配器设计
date: 2020-01-22 18:31:30
tags: [Memory Management]
summary: 这篇文章介绍了如何从头开始实现堆分配器。它提出并讨论了不同的分配器设计，包括Bump分配，基于链表的分配和固定大小的块分配。 对于这三种设计中的每一种，我们将创建一个可用于我们的内核的基本实现。
---

## 介绍

在[上一篇文章](https://os.phil-opp.com/heap-allocation)中我们向内核添加了对堆内存分配的基本支持。 为此，我们在页表中[创建了一个新的内存区域](https://os.phil-opp.com/heap-allocation#creating-a-kernel-heap) ，并[使用`linked_list_allocator` crate](https://os.phil-opp.com/heap-allocation#using-an-allocator-crate)管理该部分内存。 现在我们有了一个可以工作的堆，但大部分工作都是这个crate做的，而我们没有了解它是如何工作的。

在本文中，我们将展示如何从头开始创建自己的堆分配器，而不是依赖现有的分配器crate。 我们将讨论不同的分配器设计，包括简单的*Bump分配器*和基本的*固定大小的块分配器* ，并使用此过程中得到的知识来实现（相比`linked_list_allocator` crate）性能更好的分配器。

### 设计目标

分配器的职责是管理可用的堆内存。 它需要在`alloc`调用中返回未使用的内存，并跟踪由`dealloc`释放的内存，以便可以再次重用它。 最重要的是，它绝不能重复分配已经在其他地方使用的内存，因为这会导致不确定的行为。

除了正确性之外，还有许多次要设计目标。 例如，分配器应有效地利用可用内存并使[*碎片*](https://en.wikipedia.org/wiki/Fragmentation_(computing))减少。此外，它对于并发应用程序应能很好地工作，并可扩展到任意数量的处理器。 为了获得最佳性能，它甚至可以针对CPU缓存优化内存布局，以提高[缓存亲和性](http://docs.cray.com/books/S-2315-50/html-S-2315-50/qmeblljm.html)并避免[False Sharing](http://mechanical-sympathy.blogspot.de/2011/07/false-sharing.html) 。

这些要求会使好的分配器非常复杂。 例如， [jemalloc](http://jemalloc.net/)具有超过30,000行代码。 通常我们不希望内核代码中的分配器如此复杂，因为其中的单个错误会就会导致严重的安全漏洞。 幸运的是，与用户空间代码相比，内核代码的分配模式通常要简单得多，因此相对简单的分配器设计通常就足够了。

在下文中，我们介绍了三种可能的内核分配器设计，并说明了它们的优缺点。

## Bump分配器

最简单的分配器设计是*Bump分配器* 。 它线性分配内存，并且仅记录分配的字节数和分配次数。 它仅在非常特定的用例中有用，因为它有一个严格的限制：它只能一次释放所有内存。

### 基本想法

Bump分配器背后的思想是通过增加（ *“Bump”* ） `next`变量来线性分配内存，该变量指向未使用的内存的开头。 一开始， `next`等于堆的起始地址。 每次分配时， `next`都会增加，因此它始终指向已用和未用内存之间的边界：

![在三个时间点的堆内存区域：1：在堆的开始处存在一个分配；下一个指针指向其结尾 2：在第一个分配之后立即添加了第二个分配；下一个指针指向第二个分配的末尾3：在第二个分配之后立即添加了第三个分配；`下一个指针指向第三分配的末尾](https://os.phil-opp.com/allocator-designs/bump-allocation.svg)

`next`指针单向移动，因此永远不会两次分配相同的存储区域。 当`next`到达堆末尾时，无法再分配更多的内存，从而导致下一次分配出现内存不足错误。

Bump分配器通常带有一个分配计数器，分配计数器在每个`alloc`调用中增加1，在每个`dealloc`调用中减少1。 当分配计数器达到零时，表示堆上分配的所有内存都已释放。 在这种情况下，可以将`next`指针重置为堆的起始地址，以便完整的堆内存可再次用于分配。

### 实现

我们通过声明一个新的`allocator::bump`子模块开始我们的实现：

```rust
// in src/allocator.rs

pub mod bump;
```

子模块的内容位于新的`src/allocator/bump.rs`文件中，如下：

```rust
// in src/allocator/bump.rs

pub struct BumpAllocator {
    heap_start: usize,
    heap_end: usize,
    next: usize,
    allocations: usize,
}

impl BumpAllocator {
    /// Creates a new empty bump allocator.
    pub const fn new() -> Self {
        BumpAllocator {
            heap_start: 0,
            heap_end: 0,
            next: 0,
            allocations: 0,
        }
    }

    /// Initializes the bump allocator with the given heap bounds.
    ///
    /// This method is unsafe because the caller must ensure that the given
    /// memory range is unused. Also, this method must be called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.heap_start = heap_start;
        self.heap_end = heap_start + heap_size;
        self.next = heap_start;
    }
}
```

`heap_start`和`heap_end`字段跟踪堆内存区域的下限和上限。 调用者需要确保这些地址有效，否则分配器将返回无效的内存。 因此， `init`函数是`unsafe`的。

`next`字段的目的是始终指向堆的第一个未使用字节，即下一个分配的起始地址。 在`init`函数中将其设置为`heap_start` ，因为在开始时没有使用任何堆内存。 在每次分配时，此字段都会增加分配大小（ *“ bumped”* ），以确保我们不会两次返回相同的内存区域。

`allocations`字段是记录分配数的简单计数器，目的是在释放最后一个分配后重置分配器。 初始化为0。

我们选择创建一个单独的`init`函数，而不是直接在`new`中直接执行初始化，以使接口与`linked_list_allocator` crate提供的分配器相同。 这样，无需更改其他代码即可切换分配器。

### 实现`GlobalAlloc`

如[前](https://os.phil-opp.com/heap-allocation#the-allocator-interface)一篇[文章所述](https://os.phil-opp.com/heap-allocation#the-allocator-interface) ，所有堆分配器都需要实现[`GlobalAlloc trait`](https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html)，其定义如下：

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

仅需要`alloc`和`dealloc`方法，其他两个方法具有默认实现，可以省略。

#### 首次尝试实现

让我们尝试为`BumpAllocator`实现`alloc`方法：

```rust
// in src/allocator/bump.rs

use alloc::alloc::{GlobalAlloc, Layout};

unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // TODO alignment and bounds check
        let alloc_start = self.next;
        self.next = alloc_start + layout.size();
        self.allocations += 1;
        alloc_start as *mut u8
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        todo!();
    }
}
```

首先，我们使用`next`字段作为分配的起始地址。 然后，我们更新`next`字段以指向分配的结束地址，这是堆上的下一个未使用的地址。 在以`*mut u8`指针返回分配的开始地址之前，我们将`allocations`计数器增加1。

请注意，我们不执行任何边界检查或对齐调整，因此此实现尚不安全。 这并不要紧，因为无论如何它都无法编译并出现以下错误：

```
error[E0594]: cannot assign to `self.next` which is behind a `&` reference
  --> src/allocator/bump.rs:29:9
   |
26 |     unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
...
29 |         self.next = alloc_start + layout.size();
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be written
```

（ `self.allocations += 1`行也发生相同的错误。为简洁起见，在此省略。）

发生该错误是因为`GlobalAlloc` trait的[`alloc`](https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.alloc)和[`dealloc`](https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html#tymethod.dealloc)方法仅在不可变的`&self`引用上运行，因此无法更新`next`和`allocations`字段。 这是有问题的，因为在每个分配上`next`进行更新是Bump分配器的基本原理。

注意，在方法声明中将`&self`更改为`&mut self`的编译器建议在这里不起作用。 原因是方法签名是由`GlobalAlloc` trait定义的，不能在实现端进行更改。 （我在Rust 的repo开了一个关于此类不合法的编译期建议的[issue](https://github.com/rust-lang/rust/issues/68049)。）

#### `GlobalAlloc`和可变性

在我们研究此可变性问题的可能解决方案之前，让我们尝试理解为什么使用`&self`参数定义`GlobalAlloc` trait方法的：正如我们在上一篇文章中看到[的那样](https://os.phil-opp.com/heap-allocation#the-global-allocator-attribute) ，全局堆分配器是通过将`#[global_allocator]`属性添加到实现`GlobalAlloc`特性的`static` 。静态变量在Rust中是不可变的，因此无法调用在静态分配器上采用`&mut self`的方法。 因此， `GlobalAlloc`所有方法仅采用不可变的`&self`引用。

幸运的是，有一种方法可以从`&self`引用中获取`&mut self`引用：通过将分配器包装在[`spin::Mutex`](https://docs.rs/spin/0.5.0/spin/struct.Mutex.html)自旋锁中，我们可以使用同步[内部可变性](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) 。 这种类型提供了执行[互斥](https://en.wikipedia.org/wiki/Mutual_exclusion)的`lock`方法，因此可以安全地将`&self`引用转换为`&mut self`引用。 我们已经在内核中多次使用了包装器类型，例如[VGA文本缓冲区](https://os.phil-opp.com/vga-text-mode#spinlocks) 。

#### `Locked`包装类型

借助`spin::Mutex`包装器类型，我们可以为Bump分配器实现`GlobalAlloc`特性。 诀窍是不是直接为`BumpAllocator`实现特征，而是为包装的`spin::Mutex`类型实现特征：

```rust
unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {…}
```

不幸的是，这仍然不起作用，因为Rust编译器不允许实现其他crate中定义trait：

```
error[E0117]: only traits defined in the current crate can be implemented for arbitrary types
  --> src/allocator/bump.rs:28:1
   |
28 | unsafe impl GlobalAlloc for spin::Mutex<BumpAllocator> {
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^--------------------------
   | |                           |
   | |                           `spin::mutex::Mutex` is not defined in the current crate
   | impl doesn't use only types from inside the current crate
   |
   = note: define and implement a trait or new type instead
```

为了解决这个问题，我们需要围绕`spin::Mutex`创建我们自己的包装器类型：

```rust
// in src/allocator.rs

/// A wrapper around spin::Mutex to permit trait implementations.
pub struct Locked<A> {
    inner: spin::Mutex<A>,
}

impl<A> Locked<A> {
    pub const fn new(inner: A) -> Self {
        Locked {
            inner: spin::Mutex::new(inner),
        }
    }

    pub fn lock(&self) -> spin::MutexGuard<A> {
        self.inner.lock()
    }
}
```

该类型是围绕`spin::Mutex<A>`的通用包装。 它对包装的类型`A`没有任何限制，因此可以用于包装所有类型，而不仅仅是分配器。 它提供了一个简单的`new`构造函数，该函数将给定值包装了起来。 为了方便起见，它还提供了一个`lock` 函数，该函数调用被包装的`Mutex`上的`lock` 。 由于`Locked`类型足够通用，因此也可用于其他分配器实现，因此我们将其放在父`allocator`模块中。

####  `Locked<BumpAllocator>`

`Locked`类型是在我们自己的crate中定义的（与`spin::Mutex`相反），因此我们可以使用它为我们的Bump分配器实现`GlobalAlloc` 。 完整的实现如下所示：

```rust
// in src/allocator/bump.rs

use super::{align_up, Locked};
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<BumpAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let mut bump = self.lock(); // get a mutable reference

        let alloc_start = align_up(bump.next, layout.align());
        let alloc_end = alloc_start + layout.size();

        if alloc_end > bump.heap_end {
            ptr::null_mut() // out of memory
        } else {
            bump.next = alloc_end;
            bump.allocations += 1;
            alloc_start as *mut u8
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        let mut bump = self.lock(); // get a mutable reference

        bump.allocations -= 1;
        if bump.allocations == 0 {
            bump.next = bump.heap_start;
        }
    }
}
```

`alloc`和`dealloc`第一步都是通过`inner`字段调用[`Mutex::lock`](https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock)方法，以获取对包装后的分配器类型的可变引用。 该实例将保持锁定状态，直到方法结束，以便在多线程上下文中不会发生数据竞争（我们将很快添加线程支持）。

与以前的原型相比， `alloc`实现现在遵守对齐要求并执行边界检查以确保分配保留在堆内存区域内。 第一步是将`next`地址四舍五入为`Layout`参数指定的对齐方式。 `align_up`显示`align_up`函数的代码。 像以前一样，我们然后将请求的分配大小添加到`alloc_start`以获得分配的结束地址。 如果它大于堆的结束地址，我们将返回一个空指针，以指示内存不足的情况。否则，我们将更新`next`地址，并像以前一样将`allocations`计数器增加1。 最后，我们返回转换为`*mut u8`指针的`alloc_start`地址。

`dealloc`函数将忽略给定的指针和`Layout`参数。 相反，它只是减少了`allocations`计数器。如果计数器再次达到`0` ，则意味着所有分配都被再次释放。 在这种情况下，它将`next`地址重置为`heap_start`地址，以使完整的堆内存再次可用。

`align_up`函数足够通用，可以将其放入父`allocator`模块中。 看起来像这样：

```rust
// in src/allocator.rs

fn align_up(addr: usize, align: usize) -> usize {
    let remainder = addr % align;
    if remainder == 0 {
        addr // addr already aligned
    } else {
        addr - remainder + align
    }
}
```

该函数首先计算`align`除`addr`除法的[余数](https://en.wikipedia.org/wiki/Euclidean_division) 。 如果余数为`0` ，则地址已与给定的对齐方式对齐。 否则，我们通过减去余数（这样新的余数为0）然后加上`align`（以便地址不会变得小于原始地址）来对齐地址。

### 使用

要使用Bump分配器而不是`linked_list_allocator` crate，我们需要更新`allocator.rs`的`ALLOCATOR`静态值：

```rust
// in src/allocator.rs

use bump::BumpAllocator;

#[global_allocator]
static ALLOCATOR: Locked<BumpAllocator> = Locked::new(BumpAllocator::new());
```

在这里，重要的是，我们将`BumpAllocator::new`和`Locked::new` 定义为[`const`函数](https://doc.rust-lang.org/reference/items/functions.html#const-functions) 。 如果它们是普通函数，则将发生编译错误，因为`static`的初始化表达式必须在编译时求值。

我们不需要在`init_heap`函数中更改`ALLOCATOR.lock().init(HEAP_START, HEAP_SIZE)`调用，因为凹凸分配器提供的接口与`linked_list_allocator`提供的分配器相同。

现在我们的内核使用我们的凹凸分配器！ 一切都应该仍然有效，包括我们在上[`heap_allocation`](https://os.phil-opp.com/heap-allocation#adding-a-test)文章中创建的[`heap_allocation`测试](https://os.phil-opp.com/heap-allocation#adding-a-test) ：

```
> cargo xtest --test heap_allocation
[…]
Running 3 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
```

### 讨论

Bump分配的最大优点是它非常快。 与需要主动寻找合适的内存块并在`alloc`和`dealloc`时执行各种记录任务的其他分配器设计（请参见下文）相比， 可以将Bump分配器优化为仅使用数条汇编指令。 这使得凹凸分配器对于优化分配性能很有用，例如在创建[虚拟DOM库时](https://hacks.mozilla.org/2019/03/fast-bump-allocated-virtual-doms-with-rust-and-wasm) 。

虽然很少使用Bump分配器作为全局分配器，但是凹凸分配的原理通常以竞技场分配的形式应用，[竞技场分配](https://mgravell.github.io/Pipelines.Sockets.Unofficial/docs/arenas.html)基本上是将多次分配集中在一起以提高性能。 Rust的竞技场分配器的一个示例是[`toolshed`](https://docs.rs/toolshed/0.8.1/toolshed/index.html) 。

#### Bump分配器的缺点

Bump分配器的主要局限性在于，只有释放所有分配后，它才能重新使用释放的内存。 这意味着单个长期分配足以防止内存重用。 当我们添加`many_boxes`测试的变体时，我们可以看到这一点：

```rust
// in tests/heap_allocation.rs

#[test_case]
fn many_boxes_long_lived() {
    serial_print!("many_boxes_long_lived... ");
    let long_lived = Box::new(1); // new
    for i in 0..HEAP_SIZE {
        let x = Box::new(i);
        assert_eq!(*x, i);
    }
    assert_eq!(*long_lived, 1); // new
    serial_println!("[ok]");
}
```

像`many_boxes`测试一样，如果分配器不重新使用释放的内存，此测试会创建大量分配以引发内存不足故障。 此外，测试会创建一个`long_lived`分配，该分配对于整个循环执行有效。

当我们尝试运行新测试时，我们发现它确实失败了：

```
> cargo xtest --test heap_allocation
Running 4 tests
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [failed]

Error: panicked at 'allocation error: Layout { size_: 8, align_: 8 }', src/lib.rs:86:5
```

让我们尝试详细了解为什么会发生此故障：首先，在堆的开头分配了`long_lived`，从而将`allocations`计数器增加1。对于循环的每次迭代，都会临时分配一次内存并在下一次迭代开始之前释放它。 这意味着`allocations`计数器在迭代开始时临时增加到2，在迭代结束时减少到1。 现在的问题是，凹凸分配器仅在释放*所有*分配后才可以重用内存，即`allocations`计数器降为0。由于在循环结束之前不会发生这种情况，因此每次循环迭代都会分配一个新的内存区域，多次迭代后导致了内存不足错误。

#### 重用释放的内存？

问题是：我们可以通过某种方式扩展凹凸分配器以消除此限制吗？

正如我们在上一篇文章中了解到的那样，分配可以生存任意长的时间，并且可以按任意顺序释放。 这意味着我们需要跟踪数量可能无限的非连续未使用内存区域，如以下示例所示：

![img](https://os.phil-opp.com/allocator-designs/allocation-fragmentation.svg)

该图显示了一段时间内的堆。 一开始，整个堆还没有被使用， `next`地址等于`heap_start` （第1行）。 然后发生第一个分配（第2行）。 在第3行中，分配了第二个存储块，并释放了第一次分配的内润。 在第4行中进行了更多分配。其中一半寿命很短，在第5行中已被释放，在该行中还进行了另一次新分配。

第5行显示了一个基本问题：我们共有五个未使用的内存区域，它们的大小各不相同，但是`next`指针只能指向最后一个区域的开始。 尽管对于本示例，我们可以将其他未使用的存储区域的起始地址和大小存储在大小为4的数组中，但这并不是一般的解决方案，因为我们可以轻松地创建一个包含8、16或1000个未使用的存储区域的示例。

通常，当我们有数量不定的项目时，我们可以使用在堆上分配的集合类型。 但此时这是不可能的，因为堆分配器不能依赖于自身（这将导致无限递归或死锁）。 因此，我们需要找到其他解决方案。

## 链表分配器

在实现分配器时，跟踪任意数量的空闲内存区域的常见技巧是将这些区域本身用作后备存储。 这利用了以下事实：区域仍被映射到虚拟地址并由物理帧支持，但是不再需要存储信息。 通过在已被释放的区域中存储有关的信息，我们可以跟踪无限制数量的已被释放的区域，而无需额外的内存。

最常见的实现方法是在释放的内存中构造一个链表，每个节点都是一个释放的内存区域：

![img](https://os.phil-opp.com/allocator-designs/linked-list-allocation.svg)

每个列表节点包含两个字段：内存区域的大小和指向下一个未使用的内存区域的指针。 使用这种方法，我们只需要一个指向第一个未使用区域（称为`head` ）的指针即可跟踪所有未使用区域，而与它们的数量无关。 产生的数据结构通常称为[*空闲列表*](https://en.wikipedia.org/wiki/Free_list) 。

正如您可能从名称中猜到的那样，这是`linked_list_allocator` crate使用的技术。

### 实现

在下面，我们将创建自己的简单`LinkedListAllocator`类型，该类型使用上述方法来跟踪已被释放的内存区域。 文章的其他部分不需要这一部分知识，因此您可以根据需要跳过这部分实现细节。

#### 分配器类型

我们首先在一个新的`allocator::linked_list`子模块中创建一个私有`ListNode`结构：

```rust
// in src/allocator.rs

pub mod linked_list;
```

```rust
// in src/allocator/linked_list.rs

struct ListNode {
    size: usize,
    next: Option<&'static mut ListNode>,
}
```

就像图中一样，列表节点具有`size`字段和指向下一个节点的可选指针，由`Option<&'static mut ListNode>`类型表示。 `&'static mut`类型在语义上描述了一个有所有权的，在指针后面的对象。 基本上，这是一个没有析构函数的[`Box`](https://doc.rust-lang.org/alloc/boxed/index.html) ，它在所在作用域的末尾释放对象。

我们为`ListNode`实现以下方法：

```rust
// in src/allocator/linked_list.rs

impl ListNode {
    const fn new(size: usize) -> Self {
        ListNode { size, next: None }
    }

    fn start_addr(&self) -> usize {
        self as *const Self as usize
    }

    fn end_addr(&self) -> usize {
        self.start_addr() + self.size
    }
}
```

该类型具有一个名为`new`的简单构造函数，以及用于计算所表示区域的开始和结束地址的方法。我们将`new`函数设为[const函数](https://doc.rust-lang.org/reference/items/functions.html#const-functions) ，稍后在构造静态链表分配器时将需要使用该函数。 请注意，在const函数中使用可变引用（包括将`next`字段设置为`None` ）仍然不稳定。 为了使其能够编译，我们需要在`lib.rs`的开头添加 **`#![feature(const_mut_refs)]`** 。

使用`ListNode`结构作为构建块，我们现在可以创建`LinkedListAllocator`结构：

```rust
// in src/allocator/linked_list.rs

pub struct LinkedListAllocator {
    head: ListNode,
}

impl LinkedListAllocator {
    /// Creates an empty LinkedListAllocator.
    pub const fn new() -> Self {
        Self {
            head: ListNode::new(0),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.add_free_region(heap_start, heap_size);
    }

    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        todo!();
    }
}
```

该结构包含一个指向第一个堆区域的`head`节点。 我们只对`next`指针的值感兴趣，因此我们在`ListNone::new`函数中将`size`设置为0。 我们让`head`为`ListNode`类型而非`&'static mut ListNode`类型，这样做的优点是， `alloc`方法的实现将更加简单。

像Bump分配器一样， `new`函数不会使用堆边界来初始化分配器。 除了保持API兼容性之外，原因还在于初始化函数需要将节点写入堆内存，这只能在运行时发生。 但是， `new`函数必须是可以在编译时求值的[`const`函数](https://doc.rust-lang.org/reference/items/functions.html#const-functions) ，因为它将用于初始化`ALLOCATOR`静态函数。 出于这个原因，我们再次单独提供了一个非const的`init`方法。

`init`方法使用`add_free_region`方法，稍后将显示其实现。 现在，我们使用[`todo!`](https://doc.rust-lang.org/core/macro.todo.html) 宏，以提供一个总是panic的占位符实现。

#### `add_free_region`方法

`add_free_region`方法提供对链表的基本*push*操作。 当前，我们仅从`init`调用此方法，但它也将成为我们`dealloc`实现中的中心方法。 请记住，当再次释放分配的内存区域时，将调用`dealloc`方法。 为了跟踪此释放的内存区域，我们希望将其push到链表。

`add_free_region`方法的实现如下所示：

```rust
// in src/allocator/linked_list.rs

use super::align_up;
use core::mem;

impl LinkedListAllocator {
    /// Adds the given memory region to the front of the list.
    unsafe fn add_free_region(&mut self, addr: usize, size: usize) {
        // ensure that the freed region is capable of holding ListNode
        assert!(align_up(addr, mem::align_of::<ListNode>()) == addr);
        assert!(size >= mem::size_of::<ListNode>());

        // create a new list node and append it at the start of the list
        let mut node = ListNode::new(size);
        node.next = self.head.next.take();
        let node_ptr = addr as *mut ListNode;
        node_ptr.write(node);
        self.head.next = Some(&mut *node_ptr)
    }
}
```

该方法以地址和大小表示的内存区域作为参数，并将其添加到列表的前面。 首先，它确保给定区域具有存储`ListNode`所需的大小和对齐方式。 然后，它通过以下步骤创建节点并将其插入到列表中：

![img](https://os.phil-opp.com/allocator-designs/linked-list-allocator-push.svg)

步骤0显示了在`add_free_region`之前堆的状态。 在步骤1中，使用图中标记为已`freed`的内存区域调用该方法。 初步检查后，该方法在其堆栈上创建一个具有被释放区域大小的新`node` 。 然后，它使用[`Option::take`](https://doc.rust-lang.org/core/option/enum.Option.html#method.take)方法将节点的`next`指针设置为当前的`head`指针，从而将`head`指针重置为`None` 。

在步骤2中，该方法通过[`write`](https://doc.rust-lang.org/std/primitive.pointer.html#method.write)方法将新创建的`node`写入到被释放的内存区域的开头。 然后将`head`指针指向新节点。 最终的指针结构看起来有些混乱，因为被释放区域总是插入在列表的开头，但是如果我们跟随指针，我们会看到每个空闲区域仍然可以从`head`指针到达。

#### `find_region`方法

对链表的第二项基本操作是查找条目并将其从列表中删除。 这是实现`alloc`方法所需的中心操作。 我们通过以下方式将操作实现为`find_region`方法：

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Looks for a free region with the given size and alignment and removes
    /// it from the list.
    ///
    /// Returns a tuple of the list node and the start address of the allocation.
    fn find_region(&mut self, size: usize, align: usize)
        -> Option<(&'static mut ListNode, usize)>
    {
        // reference to current list node, updated for each iteration
        let mut current = &mut self.head;
        // look for a large enough memory region in linked list
        while let Some(ref mut region) = current.next {
            if let Ok(alloc_start) = Self::alloc_from_region(&region, size, align) {
                // region suitable for allocation -> remove node from list
                let next = region.next.take();
                let ret = Some((current.next.take().unwrap(), alloc_start));
                current.next = next;
                return ret;
            } else {
                // region not suitable -> continue with next region
                current = current.next.as_mut().unwrap();
            }
        }

        // no suitable region found
        None
    }
}
```

该方法使用一个`current`变量和一个[`while let`循环](https://doc.rust-lang.org/reference/expressions/loop-expr.html#predicate-pattern-loops)来遍历列表元素。 首先，将`current`设置到（虚拟） `head`节点。 然后在每次迭代中，将其更新到当前节点的`next`字段（在`else`块中）。 如果该区域适合于具有给定大小和对齐方式的分配，则将该区域从列表中删除，并与`alloc_start`地址一起返回。

当`current.next`指针变为`None` ，循环退出。 这意味着我们遍历了整个列表，但是没有找到适合分配的区域。 在这种情况下，我们返回`None` 。 检查区域是否合适是通过`alloc_from_region`函数来进行的，稍后将展示其实现。

让我们更详细地研究如何从列表中删除合适的区域：

![img](https://os.phil-opp.com/allocator-designs/linked-list-allocator-remove-region.svg)

步骤0显示了任何指针调整之前的情况。 图中标记出了`region`和`current`区域以及`region.next`和`current.next`指针。 在步骤1中，可以使用[`Option::take`](https://doc.rust-lang.org/core/option/enum.Option.html#method.take)方法将`region.next`和`current.next`指针都重置为`None` 。 原始指针存储在名为`next`和`ret`局部变量中。

在步骤2中，将`current.next`指针设置为本地的`next`指针，它是原来的`region.next`指针。结果是`current`直接指向region之后的`region` ，因此`region`不再是链表的元素。 然后，该函数将指针返回到存储在本地`ret`变量中的区域。

##### `alloc_from_region`函数

`alloc_from_region`函数返回区域是否适合于具有给定大小和对齐方式的分配。 定义如下：

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Try to use the given region for an allocation with given size and
    /// alignment.
    ///
    /// Returns the allocation start address on success.
    fn alloc_from_region(region: &ListNode, size: usize, align: usize)
        -> Result<usize, ()>
    {
        let alloc_start = align_up(region.start_addr(), align);
        let alloc_end = alloc_start + size;

        if alloc_end > region.end_addr() {
            // region too small
            return Err(());
        }

        let excess_size = region.end_addr() - alloc_end;
        if excess_size > 0 && excess_size < mem::size_of::<ListNode>() {
            // rest of region too small to hold a ListNode (required because the
            // allocation splits the region in a used and a free part)
            return Err(());
        }

        // region suitable for allocation
        Ok(alloc_start)
    }
}
```

首先，该函数使用我们之前定义的`align_up`函数来计算潜在分配的开始和结束地址。 如果要求的结束地址在该区域的结束地址之后，则该分配不适合该区域，并且我们返回错误。

此后，该函数执行了一个作用不太显然的检查。 进行此检查是必要的，因为在大多数情况下，分配并不能完全适合某个合适的区域，因此分配后，该区域的一部分仍然可用。 该区域的这一部分必须在分配后存储自己的`ListNode` ，因此它必须足够大才能存储。该检查准确地验证了这一点：分配是否完全适合（`excess_size == 0`）或多余的大小足以存储a `ListNode`。

#### 实现`GlobalAlloc`

通过`add_free_region`和`find_region`方法提供的基本操作，我们现在可以最终实现`GlobalAlloc` trait。与Bump分配器一样，我们不直接针对`LinkedListAllocator`，而是仅针对包装过的`Locked<LinkedListAllocator>`来实现trait。该[`Locked`包装](https://os.phil-opp.com/allocator-designs#a-locked-wrapper-type)通过自旋锁向分配器实例添加了内部可变性，这使得我们可以修改它，即使`alloc`和`dealloc`方法的参数是不可变的`&self`引用。

实现看起来像这样：

```rust
// in src/allocator/linked_list.rs

use super::Locked;
use alloc::alloc::{GlobalAlloc, Layout};
use core::ptr;

unsafe impl GlobalAlloc for Locked<LinkedListAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // perform layout adjustments
        let (size, align) = LinkedListAllocator::size_align(layout);
        let mut allocator = self.inner.lock();

        if let Some((region, alloc_start)) = allocator.find_region(size, align) {
            let alloc_end = alloc_start + size;
            let excess_size = region.end_addr() - alloc_end;
            if excess_size > 0 {
                allocator.add_free_region(alloc_end, excess_size);
            }
            alloc_start as *mut u8
        } else {
            ptr::null_mut()
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // perform layout adjustments
        let (size, _) = LinkedListAllocator::size_align(layout);

        self.inner.lock().add_free_region(ptr as usize, size)
    }
}
```

让我们从该`dealloc`方法开始，因为它更简单：首先，它执行一些布局调整，我们将在稍后进行解释，并通过调用[Locked wrapper](https://os.phil-opp.com/allocator-designs#a-locked-wrapper-type)的[`Mutex::lock`](https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock)函数来获取`&mut LinkedListAllocator`。然后，它调用`add_free_region`函数以将释放区域添加到空闲列表中。

该`alloc`方法有点复杂。它以相同的布局调整开始，并且还调用该[`Mutex::lock`](https://docs.rs/spin/0.5.0/spin/struct.Mutex.html#method.lock)函数以接收可变的分配器引用。然后，它使用`find_region`方法找到适合分配的存储区域，并将其从列表中删除。如果此操作失败并返回`None`，则返回`null_mut`，告知用户没有合适的存储区域。

在成功的情况下，该`find_region`方法返回合适区域（不再在列表中）的元组和分配的起始地址。使用`alloc_start`、分配大小和区域的结束地址，它再次计算分配的结束地址和多余的大小。如果多余的大小不为null，则调用`add_free_region`将内存区域的多余的大小添加回空闲列表。最后，它返回转换为`*mut u8`指针的`alloc_start`。

#### 布局调整

那么我们在`alloc`和`dealloc`的一开始做的这些布局的调整，都是什么？它们确保每个分配的块都能够存储一个`ListNode`。这一点很重要，因为在某个时候我们要释放这一块内存并向其中写一个`ListNode`。如果块小于一个`ListNode`的大小或没有正确的对齐方式，则会发生未定义的行为。

布局调整由一个`size_align`函数执行，该函数定义如下：

```rust
// in src/allocator/linked_list.rs

impl LinkedListAllocator {
    /// Adjust the given layout so that the resulting allocated memory
    /// region is also capable of storing a `ListNode`.
    ///
    /// Returns the adjusted size and alignment as a (size, align) tuple.
    fn size_align(layout: Layout) -> (usize, usize) {
        let layout = layout
            .align_to(mem::align_of::<ListNode>())
            .expect("adjusting alignment failed")
            .pad_to_align();
        let size = layout.size().max(mem::size_of::<ListNode>());
        (size, layout.align())
    }
}
```

首先，如果需要，函数对[`Layout`](https://doc.rust-lang.org/alloc/alloc/struct.Layout.html)参数应用`align_to`方法来将内存对齐到能够容纳一个 `ListNode` 。然后，它使用该[`pad_to_align`](https://doc.rust-lang.org/core/alloc/struct.Layout.html#method.pad_to_align)方法将大小四舍五入为对齐方式的倍数，以确保下一个存储块的起始地址也将具有正确的对齐方式以存储`ListNode`。在第二步中，它使用[`max`](https://doc.rust-lang.org/std/cmp/trait.Ord.html#method.max)方法来强制最小分配大小为`mem::size_of::<ListNode>`。这样，`dealloc`函数可以安全地将`ListNode`写入已释放的内存块。

无论是`align_to`和`pad_to_align`方法都使用了不稳定的特性。要启用该功能，我们需要将添加**`#![feature(alloc_layout_extra)]`**到`lib.rs`的开头。

### 使用

现在，我们可以更新 `allocator`模块中的`ALLOCATOR`静态变量以使用我们的新`LinkedListAllocator`：

```rust
// in src/allocator.rs

use linked_list::LinkedListAllocator;

#[global_allocator]
static ALLOCATOR: Locked<LinkedListAllocator> =
    Locked::new(LinkedListAllocator::new());
```

由于`init`函数对于Bump和链表分配器的行为相同，因此我们不需要修改`init_heap`中的`init`调用。

现在，当我们再次运行`heap_allocation`测试时，我们看到所有测试现在都通过了，包括Bump分配器没有通过的`many_boxes_long_lived`：

```
> cargo xtest --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

这表明我们的链表分配器能够将释放的内存重新用于后续分配。 

### 讨论

与Bump分配器相比，链表分配器更适合用作通用分配器，主要是因为它能够直接重用释放的内存。但是，它也有一些缺点。其中一些是由我们的实现引起的，但是链表分配器这种设计本身也存在一些缺点。

#### 合并释放的块

我们实现的主要问题是，它只会将堆拆分为较小的块，而不会将它们重新合并在一起。考虑以下示例：

![img](https://os.phil-opp.com/allocator-designs/linked-list-allocator-fragmentation-on-dealloc.svg)

在第一行中，在堆上进行了三次分配。它们中的两个在第2行中再次被释放，而第三个在第3行中被释放。现在整个堆都不再被使用，但仍被分成四个单独的块。此时，可能无法再进行大分配，因为四个块都不足够大。随着时间的流逝，该过程将继续进行，并将堆分成越来越小的块。这样下去总有一天，堆中会充满此类碎片，以至于正常大小的分配都将失败。

要解决此问题，我们需要将相邻的释放块重新合并在一起。对于上面的示例，这意味着：

![img](https://os.phil-opp.com/allocator-designs/linked-list-allocator-merge-on-dealloc.svg)

像以前一样，三个分配中的两个在第`2`行中被释放。现在，我们不执行保留碎片的堆的操作，而是直接执行另外一个步骤，`2a`行将两个最右边的块合并在一起。在`3`第3 行中，释放了第三个分配（如之前一样），从而导致完全未使用的堆由三个不同的块组成。然后，在附加的合并步骤中（`3a`行）我们将三个相邻的块合并在一起。

`linked_list_allocator` crate以下列方式这一合并策略：它始终保持列表按照起始地址排序，而不是直接在链表的开头插入被`deallocate`释放的内存块。这样，`deallocate`调用时可以通过检查列表中两个相邻块的地址和大小来进行合并。当然，这种方式的释放操作比较慢，但是可以防止我们在上面看到的堆碎片。

#### 性能

正如我们在上面学到的，Bump分配器非常快，可以优化为仅需几个汇编指令操作。链表分配器的性能要差得多。其问题是分配请求可能需要遍历完整的链表，直到找到合适的块为止。

由于列表长度取决于未使用的存储块的数量，因此对于不同的程序，其性能可能会发生极大的变化。仅创建几个分配的程序将有相对较快的分配性能。但是，对于一个会进行很多次分配，导致堆变得碎片化的程序，分配性能将非常糟糕，因为链表将非常长，并且其中大多数包含的块都非常小。

值得注意的是，这类性能问题不是由我们的实现引起的问题，而是链表方法的根本问题。由于分配性能对于内核级代码非常重要，因此我们在下面探讨了第三种分配器设计，该设计以降低内存利用率代价来提高性能。

## 固定大小的块分配器

下面，我们介绍一种分配器设计，该设计使用固定大小的内存块来满足分配请求。这样，分配器通常返回的块大于分配所需的块，这会由于[内部碎片](https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation)而导致内存浪费。另一方面，它大大减少了找到合适块所需的时间（与链表分配器相比），从而有更好的分配性能。

### 介绍

*固定大小的块分配器*背后的思想是：并不恰好分配大小等于所需的内存，而是固定一些块大小并将每个分配舍入到能装下它的最小块大小。例如，对于16、64和512字节的块大小，分配4个字节将返回一个16字节的块，分配一个48字节将返回一个64字节的块，而分配128个字节将返回一个512字节的块。 。

像链表分配器一样，我们通过在未使用的内存中创建链表来跟踪未使用的内存。但是，我们不使用具有不同块大小的单个列表，而是为每个大小类创建一个单独的列表。然后，每个链表仅存储单个大小的块。例如，对于块大小为16、64和512的内存，将有三个单独的链表：

![img](https://os.phil-opp.com/allocator-designs/fixed-size-block-example.svg)

我们有三个头指针`head_16`，`head_64`以及`head_512`而不是一个单一的`head`指针，每个都指到到对应的大小的第一个未使用块。单个列表中的所有节点都具有相同的大小。例如，由`head_16`指针开始的列表仅包含16字节的块。这意味着我们不再需要在每个列表节点中存储大小，因为它已经由头指针的名称指定了。

由于列表中的每个元素都具有相同的大小，因此每个列表元素都同样适用于分配请求。这意味着我们可以使用以下步骤非常有效地执行分配：

- 将请求的分配大小四舍五入到下一个块大小。例如，当请求分配12个字节时，在上面的示例中，我们将选择16的块大小。
- 检索列表的头部指针，例如从一个头指针数组中检索。对于块大小16，我们需要使用`head_16`。
- 从列表中删除第一个块并返回它。 

最值得注意的是，我们总是可以返回链表的第一个元素，而不再需要遍历整个链表。因此，分配比使用链表分配器快得多。

#### 块大小和浪费的内存

根据块大小，我们通过舍入会损失很多内存。例如，当返回一个512字节的块以进行128字节的分配时，分配的内存中有四分之三未被使用。通过定义合理的块大小，可以在某种程度上限制浪费的内存量。例如，当使用2的幂（4、8、16、32、64、128、…）作为块大小时，在最坏的情况下，我们可以将内存浪费限制为分配大小的一半，平均情况下为分配的四分之一。

通常也会基于程序中的常用的分配大小来优化块大小。例如，我们可以额外增加块大小24，以提高经常执行24字节分配的程序的内存使用率。这样，通常可以减少浪费的内存量，而不会损失性能优势。

#### 释放

像分配一样，释放也非常高效。它涉及以下步骤：

- 将释放的内存大小四舍五入到下一个块大小。这是必需的，因为编译器仅将请求的分配大小传递给`dealloc`，而不传递`alloc`所返回的块的大小。通过在`alloc`和`dealloc`中使用相同的大小调整函数我们可以确保始终释放正确的内存量。
- 检索列表的头部指针，例如从一个头指针数组中检索。 
- 通过更新头指针，将释放的块添加到列表的开头。 

最值得注意的是，也无需遍历列表即可进行释放。这意味着`dealloc`无论列表长度如何，所需的时间都保持不变。

#### 后备分配器

鉴于大容量分配（> 2KB）通常是很少见的，尤其是在操作系统内核中，因此有可能需要退回到其他分配器进行这些分配。例如，为了减少内存浪费，我们可以退回到链表分配器中进行大于2048字节的分配。由于预期该大小的分配非常少，因此这个链表将保持较小状态，因此分配和释放仍会相当快。

#### 创建新块

上面，我们始终假定列表中始终有足够的特定大小的块来满足所有分配请求。但是，在某个时候，块大小的链接列表将为空。此时，有两种方法可以创建特定大小的未使用的新块来满足分配请求：

- 从后备分配器分配一个新块（如果有）。 
- 从其他列表中拆分较大的块。如果块大小为2的幂，则此方法最有效。例如，一个32字节的块可以分为两个16字节的块。

对于我们的实现，为了简单考虑，因此我们将从后备分配器分配新块。 

### 实现

现在我们知道了固定大小的块分配器是如何工作的，我们可以开始实现它了。我们将不依赖于上一节中创建的链表分配器的实现，因此即使您跳过了链表分配器的实现，也可以看这一部分。

#### 列表节点

我们通过在新`allocator::fixed_size_block`模块中创建`ListNode`类型来开始实现：

```rust
// in src/allocator.rs

pub mod fixed_size_block;
```

```rust
// in src/allocator/fixed_size_block.rs

struct ListNode {
    next: Option<&'static mut ListNode>,
}
```

这种类型类似于我们的[链表分配器](https://os.phil-opp.com/allocator-designs#the-allocator-type)中的`ListNode`类型，不同之处在于我们没有第二个`size`字段。不需要该字段是因为每个列表中的每个块都具有相同的大小。

#### 块大小

接下来，我们定义一个常量`BLOCK_SIZES`切片，并使用用于实现的块大小：

```rust
// in src/allocator/fixed_size_block.rs

/// The block sizes to use.
///
/// The sizes must each be power of 2 because they are also used as
/// the block alignment (alignments must be always powers of 2).
const BLOCK_SIZES: &[usize] = &[8, 16, 32, 64, 128, 256, 512, 1024, 2048];
```

我们使用从8到2048的2的幂作为块大小。我们不定义任何小于8的块大小，因为每个块在释放时必须能够存储指向下一个块的64位指针。对于大于2048字节的分配，我们将退回到链表分配器。

为了简化实现，我们定义块的大小也是其在内存中所需的对齐方式。因此，一个16字节的块始终在16字节的边界上对齐，而512字节的块始终在512字节的边界上对齐。由于对齐始终需要为2的幂，因此排除了任何其他块大小。如果将来需要的块大小不是2的幂，我们仍然可以为此调整实现（例如，通过定义第二个`BLOCK_ALIGNMENTS`数组）。

#### 分配器类型

使用`ListNode`类型和`BLOCK_SIZES`切片，我们现在可以定义分配器类型：

```rust
// in src/allocator/fixed_size_block.rs

pub struct FixedSizeBlockAllocator {
    list_heads: [Option<&'static mut ListNode>; BLOCK_SIZES.len()],
    fallback_allocator: linked_list_allocator::Heap,
}
```

该`list_heads`字段是一个`head`指针数组，每个块大小一个。这是通过`len()`将`BLOCK_SIZES`slice的用作数组长度来实现的。作为大于最大块大小的分配的后备分配器，我们使用由`linked_list_allocator`提供的分配器。我们也可以使用我们自己实现的`LinkedListAllocator`，但是它的缺点是它不[合并被释放的块](https://os.phil-opp.com/allocator-designs#merging-freed-blocks)。

对于`FixedSizeBlockAllocator`，我们提供了与其他分配器类型相同的`new`和`init`函数：

```rust
// in src/allocator/fixed_size_block.rs

impl FixedSizeBlockAllocator {
    /// Creates an empty FixedSizeBlockAllocator.
    pub const fn new() -> Self {
        FixedSizeBlockAllocator {
            list_heads: [None; BLOCK_SIZES.len()],
            fallback_allocator: linked_list_allocator::Heap::empty(),
        }
    }

    /// Initialize the allocator with the given heap bounds.
    ///
    /// This function is unsafe because the caller must guarantee that the given
    /// heap bounds are valid and that the heap is unused. This method must be
    /// called only once.
    pub unsafe fn init(&mut self, heap_start: usize, heap_size: usize) {
        self.fallback_allocator.init(heap_start, heap_size);
    }
}
```

`new`函数只是使用空节点初始化`list_heads`数组，并创建一个空链表分配器作为`fallback_allocator`。 由于初始化非`Copy`类型的数组仍然是不稳定特性，因此我们需要在lib.rs的开头添加`#![feature(const_in_array_repeat_expressions)]`。 在这种情况下，`None`不是`Copy`的原因是`ListNode`没有实现`Copy`。 因此，`Option`包装及其`None`变体也不是`Copy`。

不安全的init函数仅调用`fallback_allocator`的init函数，而无需对`list_heads`数组进行任何其他初始化。 相反，我们将在`alloc`和`dealloc`调用时lazy地初始化列表。

为了方便起见，我们还创建了一个私有`fallback_alloc`方法，该方法使用`fallback_allocator`分配空间：

```rust
// in src/allocator/fixed_size_block.rs

use alloc::alloc::Layout;
use core::ptr;

impl FixedSizeBlockAllocator {
    /// Allocates using the fallback allocator.
    fn fallback_alloc(&mut self, layout: Layout) -> *mut u8 {
        match self.fallback_allocator.allocate_first_fit(layout) {
            Ok(ptr) => ptr.as_ptr(),
            Err(_) => ptr::null_mut(),
        }
    }
}
```

由于`linked_list_allocator`crate的[`Heap`](https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html)类型无法实现[`GlobalAlloc`](https://doc.rust-lang.org/alloc/alloc/trait.GlobalAlloc.html)（因为如果没有锁，这是不可能的）。而是提供了[`allocate_first_fit`](https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html#method.allocate_first_fit)这样一个接口稍有不同的方法。它返回`Result<NonNull<u8>, AllocErr>`，而不是返回 `*mut u8`并使用空指针来表示错误。该[`NonNull`](https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html)类型是保证非空的原始指针的抽象。[`AllocErr`](https://doc.rust-lang.org/nightly/core/alloc/struct.AllocErr.html)是用于标记分配错误的类型。通过将`Ok`case 映射到[`NonNull::as_ptr`](https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html#method.as_ptr)方法并将`Err`case 映射到空指针，我们可以轻松地将其转换回`*mut u8`类型。

#### 计算链表索引

在实现`GlobalAlloc`特征之前，我们定义一个`list_index`辅助函数，该函数对一个[`Layout`](https://doc.rust-lang.org/alloc/alloc/struct.Layout.html)返回给定的最小可能块大小：

```rust
// in src/allocator/fixed_size_block.rs

/// Choose an appropriate block size for the given layout.
///
/// Returns an index into the `BLOCK_SIZES` array.
fn list_index(layout: &Layout) -> Option<usize> {
    let required_block_size = layout.size().max(layout.align());
    BLOCK_SIZES.iter().position(|&s| s >= required_block_size)
}
```

被分配的块必须至少具有给定`Layout`所需的大小和对齐方式。由于我们让块的大小等于其对齐到的大小，因此这意味着`required_block_size`是`size()`和`align()`属性的[最大值](https://doc.rust-lang.org/core/cmp/trait.Ord.html#method.max)。要查找`BLOCK_SIZES`切片中首个比所要求的更大的块，我们首先使用`iter()`方法来获得迭代器，然后使用`position`方法来找到第一个大小大于等于要求的块的索引。

请注意，我们不返回块大小本身，而是返回`BLOCK_SIZES`切片的索引。原因是我们要使用返回的索引作为`list_heads`数组的索引。

#### 实现`GlobalAlloc`

最后一步是实现`GlobalAlloc` trait：

```rust
// in src/allocator/fixed_size_block.rs

use super::Locked;
use alloc::alloc::GlobalAlloc;

unsafe impl GlobalAlloc for Locked<FixedSizeBlockAllocator> {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        todo!();
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        todo!();
    }
}
```

像其他分配器一样，我们不直接为`GlobalAlloc`分配器类型实现trait，而是使用[`Locked`包装器](https://os.phil-opp.com/allocator-designs#a-locked-wrapper-type)添加同步的内部可变性。由于`alloc`和的`dealloc`的实现相对较大，因此我们在下面逐一介绍它们。

##### `alloc`

该`alloc`方法的实现如下所示：

```rust
// in `impl` block in src/allocator/fixed_size_block.rs

unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            match allocator.list_heads[index].take() {
                Some(node) => {
                    allocator.list_heads[index] = node.next.take();
                    node as *mut ListNode as *mut u8
                }
                None => {
                    // no block exists in list => allocate new block
                    let block_size = BLOCK_SIZES[index];
                    // only works if all block sizes are a power of 2
                    let block_align = block_size;
                    let layout = Layout::from_size_align(block_size, block_align)
                        .unwrap();
                    allocator.fallback_alloc(layout)
                }
            }
        }
        None => allocator.fallback_alloc(layout),
    }
}
```

让我们一步步看： 

首先，我们使用该`Locked::lock`方法来获取对包装后的分配器实例的可变引用。接下来，我们调用`list_index`刚刚定义的函数，以计算给定布局的适当块大小，并将相应的索引获取到`list_heads`数组中。如果此索引为`None`，则不适合该分配的块大小，因此我们使用`fallback_allocator`using `fallback_alloc`函数。

如果列表索引为`Some`，则尝试`list_heads[index]`使用该[`Option::take`](https://doc.rust-lang.org/core/option/enum.Option.html#method.take)方法删除相应列表中的第一个节点。如果列表不为空，则进入语句的`Some(node)`分支，在`match`该处将列表的顶部指针指向弹出的后继对象`node`（[`take`](https://doc.rust-lang.org/core/option/enum.Option.html#method.take)再次使用）。最后，我们将弹出的`node`指针作为返回`*mut u8`。

如果列表头为`None`，则表示块列表为空。这意味着我们需要[如上所述](https://os.phil-opp.com/allocator-designs#creating-new-blocks)构建一个新块。为此，我们首先从`BLOCK_SIZES`切片中获取当前块的大小，并将其用作新块的大小和对齐方式。然后，我们`Layout`从中创建一个新对象，并调用该`fallback_alloc`方法来执行分配。调整布局和对齐方式的原因是，该块将在取消分配时添加到块列表中。

#### `dealloc`

`dealloc`方法的实现如下所示：

```rust
// in src/allocator/fixed_size_block.rs

use core::{mem, ptr::NonNull};

// inside the `unsafe impl GlobalAlloc` block

unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
    let mut allocator = self.lock();
    match list_index(&layout) {
        Some(index) => {
            let new_node = ListNode {
                next: allocator.list_heads[index].take(),
            };
            // verify that block has size and alignment required for storing node
            assert!(mem::size_of::<ListNode>() <= BLOCK_SIZES[index]);
            assert!(mem::align_of::<ListNode>() <= BLOCK_SIZES[index]);
            let new_node_ptr = ptr as *mut ListNode;
            new_node_ptr.write(new_node);
            allocator.list_heads[index] = Some(&mut *new_node_ptr);
        }
        None => {
            let ptr = NonNull::new(ptr).unwrap();
            allocator.fallback_allocator.deallocate(ptr, layout);
        }
    }
}
```

像`alloc`中一样，我们首先使用`lock`方法获取可变的分配器引用，然后使用`list_index`函数获取与给定`Layout`对应的块列表。如果index为`None`，则不存在合适的块大小`BLOCK_SIZES`，这表明该分配是由后备分配器创建的。因此，我们使用后备分配器的[`deallocate`](https://docs.rs/linked_list_allocator/0.6.4/linked_list_allocator/struct.Heap.html#method.deallocate)释放内存。该方法期望使用 [`NonNull`](https://doc.rust-lang.org/nightly/core/ptr/struct.NonNull.html)而不是 `*mut u8`，因此我们需要先转换指针。 （`unwrap`仅在指针为null时调用失败，这在编译器调用`dealloc`时永远不会发生。）

如果`list_index`返回了块索引，则需要将释放的内存块添加到列表中。为此，我们首先创建一个`ListNode`指向当前列表头的新内容（通过再次使用[`Option::take`](https://doc.rust-lang.org/core/option/enum.Option.html#method.take)）。在将新节点写入释放的内存块之前，我们首先要断言由指定的当前块大小`index`具有存储`ListNode`大小和对齐方式所需的空间。然后，我们通过将给定的`*mut u8`指针转换为`*mut ListNode`指针，然后[`write`](https://doc.rust-lang.org/std/primitive.pointer.html#method.write)在其上调用unsafe 方法来执行写操作。最后一步是将列表的起始指针设置为新创建的结点。为此，我们将原始指针`new_node_ptr`转换为可变引用。

有几件事值得注意： 

- 我们不区分从块列表分配的块和从后备分配器分配的块。这意味着在`alloc`中创建的新块将添加到上的块列表中`dealloc`，从而增加了该大小的块数。
- 该`alloc`方法是在我们的实现中唯一创建新块的地方。这意味着我们最初从空块列表开始，仅在执行针对该块大小的分配时才懒惰地填充列表。
- 即使我们执行了某些`unsafe`操作，也不需要在`alloc`和`dealloc`中添加`unsafe`块。原因是Rust目前将`unsafe`函数的全部视为一个大`unsafe`块。由于使用显式块具有明显的优点，即哪些操作不安全，哪些操作不安全，因此[有一个RFC](https://github.com/rust-lang/rfcs/pull/2585)建议更改此行为。

### 使用

要使用新的 `FixedSizeBlockAllocator`，我们需要更新`allocator`模块中的`ALLOCATOR`静态变量：

```rust
// in src/allocator.rs

use fixed_size_block::FixedSizeBlockAllocator;

#[global_allocator]
static ALLOCATOR: Locked<FixedSizeBlockAllocator> = Locked::new(
    FixedSizeBlockAllocator::new());
```

由于`init`函数对于我们实现的所有分配器的行为均相同，因此无需修改`init_heap`中的`init`调用。

现在，当我们再次运行`heap_allocation`测试时，所有测试仍应通过：

```
> cargo xtest --test heap_allocation
simple_allocation... [ok]
large_vec... [ok]
many_boxes... [ok]
many_boxes_long_lived... [ok]
```

我们的新分配器似乎可以正常工作！ 

### 讨论

尽管固定大小的块方法比链表方法具有更好的性能，但是当使用2的幂作为块大小时，它最差情况下会浪费多达一半的内存。这种权衡是否值得，在很大程度上取决于应用程序类型。对于性能至关重要的操作系统内核，固定大小块方法似乎是更好的选择。

在实现方面，我们可以在当前的实现中进行很多改进： 

- 与其仅使用后备分配器Lazy地分配块，还不如预先填充列表以提高初始分配的性能。 
- 为了简化实现，我们只允许块大小为2的幂，以便我们也可以将它们用作块对齐。通过以不同的方式存储（或计算）对齐方式，我们还可以允许任意其他块大小。这样，我们可以增加更多的块大小，例如常见的分配大小，以最大程度地减少浪费的内存。
- 目前，我们仅创建新的块，但不再释放它们。这会导致碎片，最终可能导致大块内存的分配请求失败。为每种块大小列表强制设置最大长度可能是一个好的选择。当达到最大长度时，后续的释放使用后备分配器进行，而不是将其添加到列表中。
- 我们可以使用一个特殊的分配器来分配大于4KiB的内存，而不是使用链表分配器。这个想法利用了在4KiB页面上运行的[分页](https://os.phil-opp.com/paging-introduction)可以将连续的虚拟内存块映射到非连续的物理帧。这样，未分配内存的碎片对于大分配而言不再是问题。
- 使用这样的页面分配器，可能有必要添加最大4KiB的块大小并完全删除链表分配器。这样做的主要优点是减少了碎片并提高了性能可预测性，即更好的最坏情况性能。

请务必注意，上面概述的实现上的改进仅是建议。操作系统内核中使用的分配器通常会针对内核的特定工作负载进行高度优化，这只有通过广泛的性能分析才能实现。

### 变化

固定大小的块分配器设计也有很多变体。*slab分配器*和*伙伴分配器*是两个流行的示例，它们也用在诸如Linux之类的流行内核中。下面，我们对这两种设计进行简短介绍。

#### slab分配器

[*slab*分配器](https://en.wikipedia.org/wiki/Slab_allocation)的思想是使用与内核中选定类型直接对应的块大小。这样，那些类型的分配恰好适合块大小，并且不会浪费内存。有时，甚至有可能在未使用的块中预先初始化好类型实例，以进一步提高性能。

slab分配通常与其他分配器结合使用。例如，它可以与固定大小的块分配器一起使用，以进一步拆分分配的块，以减少内存浪费。它还经常用于在一次分配大块内存，然后在这块内存上实现[对象池模式](https://en.wikipedia.org/wiki/Object_pool_pattern)。

#### 伙伴分配器

[伙伴分配器](https://en.wikipedia.org/wiki/Buddy_memory_allocation)设计不是使用链表来管理释放的块，而是使用[二叉树](https://en.wikipedia.org/wiki/Binary_tree)数据结构以及2的幂次方块大小。当需要一定大小的新块时，它将一个较大的块分成两半，从而在树中创建两个子节点。每当再次释放一个块时，就会分析树中的相邻块。如果邻居也是空的，则将两个块重新连接在一起，成为一个大小两倍的块。

此合并过程的优点是减少了[外部碎片](https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation)，因此可以将较小的释放块重新用于较大的分配。它还不使用后备分配器，因此性能更可预测。最大的缺点是只能使用2的幂次的块大小，这可能会由于[内部碎片](https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation)而导致大量的内存浪费。因此，伙伴分配器通常与slab分配器结合使用，以将分配的块进一步拆分为多个较小的块。

## 总结

这篇文章概述了不同的分配器设计。我们学习了如何实现基本的[Bump分配器](https://os.phil-opp.com/allocator-designs#bump-allocator)，该[分配器](https://os.phil-opp.com/allocator-designs#bump-allocator)通过增加单个`next`指针来线性[分配](https://os.phil-opp.com/allocator-designs#bump-allocator)内存。虽然Bump分配非常快，但是只有释放所有分配后，它才能重新使用内存。因此，它很少用作全局分配器。

接下来，我们创建了一个[链表分配器](https://os.phil-opp.com/allocator-designs#linked-list-allocator)，该[分配器](https://os.phil-opp.com/allocator-designs#linked-list-allocator)使用空闲的内存块本身来创建链表，即所谓的[链表](https://en.wikipedia.org/wiki/Free_list)。该列表可以存储任意数量的大小不同的已释放块。尽管不会发生内存浪费，但是由于分配请求可能需要完整遍历列表，因此该方法的性能很差。我们的实现还遭受[外部碎片](https://en.wikipedia.org/wiki/Fragmentation_(computing)#External_fragmentation)的困扰，因为它不会将相邻的释放块重新合并在一起。

为了解决链表方法的性能问题，我们创建了一个[固定大小块分配器](https://os.phil-opp.com/allocator-designs#fixed-size-block-allocator)，该[分配器](https://os.phil-opp.com/allocator-designs#fixed-size-block-allocator)预定义了一组固定的块大小。对于每个块大小，都存在一个单独的[空闲列表](https://en.wikipedia.org/wiki/Free_list)，因此分配和取消分配只需要在[列表的](https://en.wikipedia.org/wiki/Free_list)开头插入/弹出，因此非常快。由于每个分配都向上舍入到下一个更大的块大小，因此由于[内部碎片](https://en.wikipedia.org/wiki/Fragmentation_(computing)#Internal_fragmentation)而浪费了一些内存。

还有更多具有不同权衡取舍的分配器设计。[slab分配](https://os.phil-opp.com/allocator-designs#slab-allocator)可以很好地优化常见固定大小结构的分配，但并非在所有情况下都适用。[伙伴分配](https://os.phil-opp.com/allocator-designs#buddy-allocator)使用二叉树将释放的块合并回去，但浪费了大量内存，因为它仅支持2次幂的块大小。同样重要的是要记住，每个内核实现都有独特的工作负载，因此没有适合所有情况的“最佳”分配器设计。

## 接下来是什么？

通过这篇文章，我们现在就结束了我们的内存管理部分。接下来，我们将开始探索[*多任务处理*](https://en.wikipedia.org/wiki/Computer_multitasking&)，从[*线程*](https://en.wikipedia.org/wiki/Thread_(computing))开始。在随后的文章中，我们将探索多处理器、进程以及基于async/await的协作式多任务处理。
