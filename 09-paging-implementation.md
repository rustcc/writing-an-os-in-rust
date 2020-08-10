---
title: 实现分页
date: 2019-09-25 07:43:38
tags: [Memory Management]
summary: 这篇文章展示了如何在我们的内核中实现分页支持。 首先探讨了使内核可以访问物理页表帧的各种技术，并讨论了它们各自的优缺点。 然后，它实现了地址转换功能和创建新地址映射的功能。
---

## 简介

上一篇文章介绍了分页的概念。通过与分段进行比较，它介绍了分页，解释了分页和页表的工作原理，然后介绍了x86_64的4级页表设计。我们发现引导加载程序已经为内核建立了页表层次结构，这意味着我们的内核已经在虚拟地址上运行。由于非法内存访问会导致页面错误异常，而不是修改任意物理内存，因此可以提高安全性。

帖子的结尾是我们无法从内核访问页表的问题，因为它们存储在物理内存中，并且内核已经在虚拟地址上运行。这篇文章在这一点上继续，并探讨使页表帧可供我们的内核访问的不同方法。我们将讨论每种方法的优缺点，然后为我们的内核决定一种方法。

要实现此方法，我们将需要引导加载程序的支持，因此我们将首先对其进行配置。之后，我们将实现遍历页表层次结构的功能，以将虚拟地址转换为物理地址。最后，我们学习如何在页表中创建新的映射以及如何找到未使用的内存帧来创建新的页表。

### 升级依赖

这篇文章需要x86_64依赖项的0.7.5版或更高版本。 您可以在Cargo.toml中更新依赖项：

```toml
[dependencies]
x86_64 = "0.7.5"
```

有关最新版本中的更改的概述，请查看 [`x86_64`changelog](https://github.com/rust-osdev/x86_64/blob/master/Changelog.md)。

## 访问页表

从我们的内核访问页表并不像看起来那样容易。 为了理解这个问题，让我们再次看一下前一篇文章的示例4级页面表层次结构：

![An example 4-level page hierarchy with each page table shown in physical memory](https://os.phil-opp.com/paging-introduction/x86_64-page-table-translation.svg)

这里重要的是每个页面条目都存储下一张表的 *物理地址*。 这避免了再次为这些地址运行地址转换，否则将对性能造成不利影响，并很容易导致无限的转换循环。

对我们来说，问题在于我们无法直接从内核访问物理地址，因为我们的内核也运行在虚拟地址之上。例如，当我们访问地址 `4 KiB` 时，我们访问的是 *虚拟* 地址 `4 KiB`，而不是存储第4级页表的 *物理* 地址 `4 KiB`。 当我们想访问 *物理* 地址 `4 KiB` 时，我们只能通过一些映射到它的虚拟地址来进行访问。

因此，为了访问页表帧，我们需要将一些虚拟页映射到它们。 创建这些映射的方法有很多，所有这些方法都允许我们访问任意页表帧。

### 恒等映射

一个简单的解决方案是**对所有页表进行恒等映射**：

![A virtual and a physical address space with various virtual pages mapped to the physical frame with the same address](https://os.phil-opp.com/paging-implementation/identity-mapped-page-tables.svg)

在此示例中，我们看到了很多被恒等映射的页表帧。这样一来，页表的物理地址也是有效的虚拟地址，因此我们可以轻松地从CR3寄存器访问所有级别的页表。

但是，它会使虚拟地址空间变得混乱，并使得找到大尺寸的连续内存区域变得更加困难。例如，假设我们要在上面的图形中创建一个大小为 1000 KiB 的虚拟内存区域，例如用于[内存映射文件](https://zh.wikipedia.org/wiki/内存映射文件)。我们无法在 `28 KiB` 处开始该区域，因为它会与 `1004KiB` 处已映射的页碰撞。因此，我们必须继续寻找，直到找到足够大的未映射区域，例如 `1008 KiB` 。这是与分段类似的碎片问题。

同样，这使创建新的页表变得更加困难，因为我们需要找到其对应的页尚未使用的物理帧。例如，假设我们为内存映射文件保留了从`1008 KiB` 开始的虚拟 1000 KiB 内存区域。现在我们不能再使用物理地址在`1000 KiB`和`2008 KiB`之间的任何帧，因为我们无法对其进行恒等映射。

### 以固定偏移量映射
为了避免弄乱虚拟地址空间，我们可以 **为页表映射使用单独的内存区域**。我们不再对页表帧进行恒等映射，而是将其从一个有固定偏移量的虚拟地址空间开始映射。例如，偏移量可以是 10 TiB：

![The same figure as for the identity mapping, but each mapped virtual page is offset by 10 TiB.](https://os.phil-opp.com/paging-implementation/page-tables-mapped-at-offset.svg)

通过将 `10TiB..(10TiB + 物理内存大小)` 范围内的虚拟地址专门用于页表映射，我们避免了恒等映射的冲突问题。但要保留虚拟地址空间中这么大的一块区域，只能在虚拟地址空间远大于物理内存大小时才可行。这在x86_64上不是问题，因为其 48 位地址空间大小为 256 TiB。

这种方法仍然有一个缺点，那就是每当我们创建一个新的页表时我们都需要创建一个新的映射。 另外，它不允许访问其他地址空间的页表，这在创建新进程时很有用。

### 映射整个物理内存

我们可以通过映射完整的物理内存——而不是仅映射页表帧——来解决这些问题：

![The same figure as for the offset mapping, but every physical frame has a mapping (at 10TiB + X) instead of only page table frames.](https://os.phil-opp.com/paging-implementation/map-complete-physical-memory.svg)

此方法允许我们的内核访问任意物理内存，包括其他地址空间的页表帧。 保留的虚拟内存范围具有与以前相同的大小，不同之处在于它不再包含未映射的页面。

这种方法的缺点是需要额外的页表来存储物理内存的映射。 这些页表需要存储在某个地方，因此它们会占用一部分物理内存，这在内存量较小的设备上可能会成为问题。

但是在 x86_64 上，我们可以使用大小为 2MiB 的大页 ([huge pages](https://en.wikipedia.org/wiki/Page_%28computer_memory%29#Multiple_page_sizes)) 来进行映射，而不是使用默认的 4KiB 页。 这样一来，由于只需要一个 3 级表和 32 个 2 级表，映射 32 GiB 物理内存仅需要 132 KiB 大小的页表。 大页还可以提高缓存效率，因为它们在转译后备缓冲器（TLB）中使用的条目更少。

### 临时映射

对于物理内存量很小的设备，我们只能在需要访问它们时才 **临时映射页表帧** 。 为了能够创建临时映射，我们只需要一个恒等映射的 1 级页表：

![A virtual and a physical address space with an identity mapped level 1 table, which maps its 0th entry to the level 2 table frame, thereby mapping that frame to page with address 0](https://os.phil-opp.com/paging-implementation/temporarily-mapped-page-tables.svg)

此图中的 1 级表控制虚拟地址空间的前2MiB。这是因为它可以通过从CR3寄存器开始，沿着4级，3级和2级页表中的第0个条目来最终访问到。索引为 8 的条目将地址 `32 KiB` 的虚拟页映射到地址`32 KiB`的物理帧，从而恒等映射了 1 级页表自身。该图通过 32 KiB 处的水平箭头显示了此恒等映射。

通过写这个恒等映射的 1 级页表，内核最多可以创建511个临时映射（512减去映射自身所需的条目）。在上面的示例中，内核创建了两个临时映射：

- 通过将 1 级表的第 0 个条目映射到地址为 `24 KiB` 的帧，它创建了一个临时映射，将 `0 KiB` 处的虚拟页映射到第 2 级页表的物理帧，如虚线箭头所示。
- 通过将 1 级表的第 9 个条目映射到地址为 `4 KiB` 的帧，它创建了一个虚拟映射，将 `36 KiB` 处的虚拟页映射到 4 级页表的物理帧，如虚线箭头所示。

现在内核可以通过写入 `0 KiB` 处的页面来访问2级页面表，并通过写入 `36 KiB` 处的页面来访问 4 级页面表。

使用临时映射访问任意页表帧的过程将是：

- 在恒等映射的第 1 级页表中搜索空闲条目。
- 将该条目映射到我们要访问的页表的物理帧。
- 通过映射到条目的虚拟页面访问目标帧。
- 将条目设置回未使用状态，从而删除临时映射。

这种方法重复使用相同的512个虚拟页来创建映射，因此仅需要4KiB的物理内存。缺点是它有点麻烦，特别是因为新映射可能需要修改多个表级别，这意味着我们将需要重复上述过程多次。

### 递归页表

另一个有趣的方法——根本不需要额外页表——是递归地映射页表。 这种方法的思想是将 4 级页面表的某些条目映射到 4 级表本身。 通过这样做，我们有效地保留了虚拟地址空间的一部分，并将所有当前和将来的页表帧映射到该空间。

让我们通过一个例子来理解这一切是如何工作的：

![An example 4-level page hierarchy with each page table shown in physical memory. Entry 511 of the level 4 page is mapped to frame 4KiB, the frame of the level 4 table itself.](https://os.phil-opp.com/paging-implementation/recursive-page-table.png)

与本文开头示例的唯一区别是，第 4 级表中索引为 `511` 的条目被映射到了物理帧`4 KiB`，也就是这个 4 级表它本身。

当 CPU 在翻译地址的过程中跟随这个条目，它不会到达一个 3 级表，而是又到达同一个 4 级表。这类似于一个调用自身的递归函数，因此这个表被称为 *递归页表* 。需要注意的是，CPU 假定 4 级表中的每个条目都指向一个 3 级表，因此现在 CPU 将这个 4 级表视为一个 3 级表。这是可行的，因为在x86_64上，所有级别的页表的布局都完全相同。

通过在开始实际转换之前进行一次或多次递归，我们可以有效地缩短CPU遍历的级别数。例如，如果我们只跟踪一次递归条目，然后进入 3 级表，则CPU认为 3 级表是 2 级表。更进一步，它将 2 级表视为 1 级表，1 级表视为映射的帧。这意味着我们现在可以读写1级页表，因为CPU认为它是映射的帧。下图说明了5个翻译步骤：

![The above example 4-level page hierarchy with 5 arrows: "Step 0" from CR4 to level 4 table, "Step 1" from level 4 table to level 4 table, "Step 2" from level 4 table to level 3 table, "Step 3" from level 3 table to level 2 table, and "Step 4" from level 2 table to level 1 table.](https://os.phil-opp.com/paging-implementation/recursive-page-table-access-level-1.png)

同样，在开始翻译之前，我们可以两次跟踪递归项，以将遍历级别的数量减少到两个：

![The same 4-level page hierarchy with the following 4 arrows: "Step 0" from CR4 to level 4 table, "Steps 1&2" from level 4 table to level 4 table, "Step 3" from level 4 table to level 3 table, and "Step 4" from level 3 table to level 2 table.](https://os.phil-opp.com/paging-implementation/recursive-page-table-access-level-2.png)

让我们一步步看：首先，CPU 根据 4 级表上的递归条目进行跳转，并认为它到达了一个 3 级表。然后，它再次进行递归，并认为它到达了一个 2 级表。但实际上，它仍然位于此 4 级表中。现在，CPU跟着另一个不同的条目跳转时，它将实际到达一个 3 级表，但 CPU 认为它已经到了一个 1 级表上。因此，当下一个条目指向2级表时，CPU 认为它指向一个被映射的帧。这使我们可以读写2级表。

访问3级和4级表的工作方式相同。为了访问3级表，我们跟随递归条目进行了 3 次跳转，使CPU认为它已经在1级表中。然后，我们跟随另一个条目并到达第 3 级表，CPU将其视为映射帧。要访问4级表本身，我们只需遵循递归项四次，直到CPU将4级表本身视为映射帧（下图中的蓝色）。

![The same 4-level page hierarchy with the following 3 arrows: "Step 0" from CR4 to level 4 table, "Steps 1,2,3" from level 4 table to level 4 table, and "Step 4" from level 4 table to level 3 table. In blue the alternative "Steps 1,2,3,4" arrow from level 4 table to level 4 table.](https://os.phil-opp.com/paging-implementation/recursive-page-table-access-level-3.png)

你可能需要一点时间理清楚这些概念，但是它在实际中很有用。

在下面的部分，我们将解释如何构建虚拟地址以方便进行一次或多次的递归。在我们实际的内核实现中我们不会使用递归页表，所以你可以不继续读下面这部分了。但如果你觉得有兴趣，可以点击“地址计算”以展开。

<details>
<summary> 地址计算 </summary>

我们看到我们可以通过在实际翻译之前递归一次或多次来访问所有级别的表。 由于四个级别的表中的索引直接来自虚拟地址，因此我们需要为此技术构建特殊的虚拟地址。 请记住，页表索引是通过以下方式从地址中得到的：

![Bits 0–12 are the page offset, bits 12–21 the level 1 index, bits 21–30 the level 2 index, bits 30–39 the level 3 index, and bits 39–48 the level 4 index](https://d33wubrfki0l68.cloudfront.net/55d00a7a89ddaf126f40bb1414de0d78fcde09e4/478a7/paging-introduction/x86_64-table-indices-from-address.svg)

假设我们想要访问映射特定页面的1级页表。 如上所述，这意味着在继续使用4级，3级和2级索引之前，我们必须递归一次。 为此，我们将地址的每个块向右移动一个块，并将原始的4级索引设置为递归条目的索引：

![Bits 0–12 are the offset into the level 1 table frame, bits 12–21 the level 2 index, bits 21–30 the level 3 index, bits 30–39 the level 4 index, and bits 39–48 the index of the recursive entry](https://os.phil-opp.com/advanced-paging/table-indices-from-address-recursive-level-1.svg)

为了访问该页面的2级页表，我们将每个索引块向右移动两个块，并将原来的4级索引和原来的3级索引的块都设置为递归条目的索引：

![Bits 0–12 are the offset into the level 2 table frame, bits 12–21 the level 3 index, bits 21–30 the level 4 index, and bits 30–39 and bits 39–48 are the index of the recursive entry](https://os.phil-opp.com/advanced-paging/table-indices-from-address-recursive-level-2.svg)

访问3级表的方法是将每个块向右移动三个块，并使用原来的4级，3级和2级地址块的递归索引：

![Bits 0–12 are the offset into the level 3 table frame, bits 12–21 the level 4 index, and bits 21–30, bits 30–39 and bits 39–48 are the index of the recursive entry](https://os.phil-opp.com/advanced-paging/table-indices-from-address-recursive-level-3.svg)

最后，我们可以通过向右移动每个块四个块并使用除偏移之外的所有地址块的递归索引来访问4级表：

![Bits 0–12 are the offset into the level l table frame and bits 12–21, bits 21–30, bits 30–39 and bits 39–48 are the index of the recursive entry](https://os.phil-opp.com/advanced-paging/table-indices-from-address-recursive-level-4.svg)

我们现在可以计算所有四个级别的页表的虚拟地址。 我们甚至可以通过将其索引乘以8（页表条目的大小）来计算精确指向特定页表条目的地址。

下表总结了访问不同类型帧的地址结构：

|           | 虚拟地址的结构(八进制)           |
| --------- | -------------------------------- |
| 页        | `0o_SSSSSS_AAA_BBB_CCC_DDD_EEEE` |
| 1级页表项 | `0o_SSSSSS_RRR_AAA_BBB_CCC_DDDD` |
| 2级页表项 | `0o_SSSSSS_RRR_RRR_AAA_BBB_CCCC` |
| 3级页表项 | `0o_SSSSSS_RRR_RRR_RRR_AAA_BBBB` |
| 4级页表项 | `0o_SSSSSS_RRR_RRR_RRR_RRR_AAAA` |

`AAA`是4级索引，`BBB`是3级索引，`CCC`是2级索引，`DDD`是映射帧的1级索引，而`EEEE`是偏移量。 `RRR`是递归条目的索引。 当索引（三位数）转换为偏移量（四位数）时，通过将其乘以8（页面表条目的大小）来完成。使用此偏移量，结果地址直接指向相应的页表条目。

`SSSSSS`是符号扩展位，这意味着它们都是第47位的副本。这是x86_64体系结构上对有效地址的特殊要求。 我们在上一篇文章中解释过它。

我们使用八进制数来表示地址，因为每个八进制字符代表三位，这使我们能够清楚地分离不同页表级别的9位索引。 对于每个字符代表四位的十六进制系统，这是不可能的。

#### Rust代码

要在Rust代码中构造这样的地址，可以使用按位操作：

```rust
// the virtual address whose corresponding page tables you want to access
let addr: usize = […];

let r = 0o777; // recursive index
let sign = 0o177777 << 48; // sign extension

// retrieve the page table indices of the address that we want to translate
let l4_idx = (addr >> 39) & 0o777; // level 4 index
let l3_idx = (addr >> 30) & 0o777; // level 3 index
let l2_idx = (addr >> 21) & 0o777; // level 2 index
let l1_idx = (addr >> 12) & 0o777; // level 1 index
let page_offset = addr & 0o7777;

// calculate the table addresses
let level_4_table_addr =
    sign | (r << 39) | (r << 30) | (r << 21) | (r << 12);
let level_3_table_addr =
    sign | (r << 39) | (r << 30) | (r << 21) | (l4_idx << 12);
let level_2_table_addr =
    sign | (r << 39) | (r << 30) | (l4_idx << 21) | (l3_idx << 12);
let level_1_table_addr =
    sign | (r << 39) | (l4_idx << 30) | (l3_idx << 21) | (l2_idx << 12);
```

上面的代码假定索引为`0o777`（511）的最后一个4级条目被递归映射。 目前情况并非如此，因此代码尚无法使用。 请参阅下文，了解如何告诉引导加载程序设置递归映射。

除了手动执行按位运算，还可以使用`x86_64` crate的[`RecursivePageTable`](https://docs.rs/x86_64/0.7.5/x86_64/structures/paging/mapper/struct.RecursivePageTable.html)类型，该类型为各种页表操作提供安全的抽象。 例如，以下代码显示了如何将虚拟地址转换为其映射的物理地址：

```rust
// in src/memory.rs

use x86_64::structures::paging::{Mapper, Page, PageTable, RecursivePageTable};
use x86_64::{VirtAddr, PhysAddr};

/// Creates a RecursivePageTable instance from the level 4 address.
let level_4_table_addr = […];
let level_4_table_ptr = level_4_table_addr as *mut PageTable;
let recursive_page_table = unsafe {
    let level_4_table = &mut *level_4_table_ptr;
    RecursivePageTable::new(level_4_table).unwrap();
}


/// Retrieve the physical address for the given virtual address
let addr: u64 = […]
let addr = VirtAddr::new(addr);
let page: Page = Page::containing_address(addr);

// perform the translation
let frame = recursive_page_table.translate_page(page);
frame.map(|frame| frame.start_address() + u64::from(addr.page_offset()))
```

同样，此代码需要有效的递归映射。 使用这种映射，可以像第一个代码示例中那样计算`level_4_table_addr`。

</details>

递归分页是一种有趣的技术，它向我们展示了一个页表中的映射可以非常有用。 它相对容易实现，只需要很少的设置（只需一个递归项），因此它是第一个分页实验的不错选择。

但是，它也有一些缺点：

- 它占用大量虚拟内存（512GiB）。 在较大的 48 位地址空间中，这不是一个大问题，但它可能导致非最优的缓存行为。
- 它仅允许轻松访问当前活动的地址空间。 通过更改递归项仍然可以访问其他地址空间，但是需要临时映射才能切换回去。 我们在（过时的）“[重新映射内核](https://os.phil-opp.com/remap-the-kernel/#overview)”一文中描述了如何执行此操作。
- 它在很大程度上依赖于 x86 的页表格式，可能无法在其他体系结构上使用。

## Bootloader 支持

所有这些方法都需要在初始化时对页表进行修改。 例如，需要创建物理内存的映射，或者需要递归映射一个 4 级表的条目。问题在于，如果没有现有的访问页表的方法，我们将无法创建这些必需的映射。

这意味着我们需要引导加载程序的帮助——它创建内核运行的页表。引导加载程序有权访问页表，因此它可以创建我们需要的任何映射。在当前的实现中，`bootloader` crate支持上述两种方法，并通过 cargo fratures 进行控制：

- `map_physical_memory` 功能将整个物理内存映射到虚拟地址空间中的某个位置。因此，内核可以访问所有物理内存，并且可以遵循 “映射完整物理内存” 方法。
- 借助 `recursive_page_table` 功能，引导加载程序将递归映射一个 4 级页面表的条目。这允许内核按照“递归页面表”部分中的描述访问页面表。

我们为内核选择第一种方法，因为它简单，平台无关且功能更强大（它还允许访问非页表框架）。为了启用所需的引导程序支持，我们将`map_physical_memory`功能添加到了引导程序依赖项中：

```toml
[dependencies]
bootloader = { version = "0.8.0", features = ["map_physical_memory"]}
```

启用此功能后，引导加载程序会将完整的物理内存映射到一些未使用的虚拟地址范围。 为了将虚拟地址范围传达给我们的内核，引导加载程序会传递一个 *引导信息* 结构。

### 引导信息

`bootloader` crate定义了一个[`BootInfo`](https://docs.rs/bootloader/0.3.11/bootloader/bootinfo/struct.BootInfo.html) 结构体，该结构包含传递给我们内核的所有信息。
这个结构体仍处于早期阶段，因此如果日后升级为 [语义版本号不兼容](https://doc.rust-lang.org/stable/cargo/reference/specifying-dependencies.html#caret-requirements) 的版本时，可能会出现不向后兼容的情况。
当启用 `map_physical_memory` 功能后，当前它具有两个字段 `memory_map` 和 `physical_memory_offset`：

- `memory_map` 字段包含可用物理内存的概述。这告诉我们内核系统中有多少可用物理内存，以及哪些内存区域为 VGA 硬件等设备保留。内存映射可以从 BIOS 或 UEFI 固件中查询，但是只能在启动过程中的早期进行查询。出于这个原因，它必须由引导加载程序提供，因为内核无法在之后获取它。在本文的后面，我们将需要内存映射。
- `physical_memory_offset` 告诉我们物理内存映射的虚拟起始地址。通过将此偏移量添加到物理地址，我们可以获得相应的虚拟地址。这使我们可以从内核访问任意物理内存。

引导加载程序以 `_start` 函数的 `&'static BootInfo` 参数的形式将 `BootInfo` 结构体传递给我们的内核。我们尚未在函数中声明此参数，因此让我们添加一下：

```rust
// in src/main.rs

use bootloader::BootInfo;

#[no_mangle]
pub extern "C" fn _start(boot_info: &'static BootInfo) -> ! { // new argument
    […]
}
```

在此之前我们不此参数不是问题，因为 x86_64 调用约定在 CPU 寄存器中传递第一个参数。 因此，如果不声明参数，这个参数会被忽略。 但是，如果我们不小心使用了错误的参数类型，那就有问题了，因为编译器不知道我们入口点函数的正确类型签名。

### `entry_point`宏
由于`_start`函数是从引导加载程序外部调用的，因此不会检查函数签名。 这意味着我们可以让它接受任意参数而没有任何编译错误，但是它将失败或在运行时导致未定义的行为。

为了确保入口点函数始终具有引导程序期望的正确签名，`bootloader` crate提供了`entry_point`宏，该宏提供了类型检查的方式来将Rust函数定义为入口点。 让我们重写我们的入口点函数以使用此宏：

```rust
// in src/main.rs

use bootloader::{BootInfo, entry_point};

entry_point!(kernel_main);

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
}
```

我们不再需要在我们的入口点上使用`extern "C"`或`no_mangle`，因为该宏为我们定义了真正的下层`_start`入口点。 现在，`kernel_main`函数是一个完全正常的Rust函数，因此我们可以为其选择任意名称。 重要的是对它进行类型检查，以便在我们使用错误的函数签名时（例如通过添加参数或更改参数类型）发生编译错误。

让我们在`lib.rs`中执行相同的更改：

```rust
// in src/lib.rs

#[cfg(test)]
use bootloader::{entry_point, BootInfo};

#[cfg(test)]
entry_point!(test_kernel_main);

/// Entry point for `cargo xtest`
#[cfg(test)]
fn test_kernel_main(_boot_info: &'static BootInfo) -> ! {
    // like before
    init();
    test_main();
    hlt_loop();
}
```

由于入口点仅在测试模式下使用，因此我们将`＃[cfg(test)]`属性添加到所有项。 我们为测试入口点指定不同的名称`test_kernel_main`，以避免与main.rs的`kernel_main`混淆。 我们暂时不使用`BootInfo`参数，因此我们在参数名称前添加`_`以禁用"未使用的变量"警告。

## 实现

现在，我们可以访问物理内存了，我们终于可以开始实现页表代码了。 首先，我们来看看运行内核的当前活动页表。 在第二步中，我们将创建一个转换函数，该函数返回给定虚拟地址映射到的物理地址。 作为最后一步，我们将尝试修改页表以创建新的映射。

在开始之前，我们为代码创建一个新的内存模块：

```rust
// in src/lib.rs

pub mod memory;
```

### 访问页表
在上一篇文章的末尾，我们试图查看内核运行的页表，但是由于无法访问CR3寄存器指向的物理帧而失败。 现在，我们可以通过创建一个 `active_level_4_table` 函数来返回对活动4级页面表的引用，从而继续：

```rust
// in src/memory.rs

use x86_64::{
    structures::paging::PageTable,
    VirtAddr,
};

/// Returns a mutable reference to the active level 4 table.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`. Also, this function must be only called once
/// to avoid aliasing `&mut` references (which is undefined behavior).
pub unsafe fn active_level_4_table(physical_memory_offset: VirtAddr)
    -> &'static mut PageTable
{
    use x86_64::registers::control::Cr3;

    let (level_4_table_frame, _) = Cr3::read();

    let phys = level_4_table_frame.start_address();
    let virt = physical_memory_offset + phys.as_u64();
    let page_table_ptr: *mut PageTable = virt.as_mut_ptr();

    &mut *page_table_ptr // unsafe
}
```

首先，我们从CR3寄存器中读取活动的4级表的物理帧。 然后，我们获取其物理起始地址，将其转换为u64，并将其与 `physical_memory_offset` 相加，以获取页表帧所映射的虚拟地址。 最后，我们通过 `as_mut_ptr` 方法将虚拟地址转换为`*mut PageTable` 原始指针，然后从中安全地创建 `&mut PageTable` 引用。 我们创建一个 `&mut` 引用而不是 `&` 引用，因为我们将在本文后面中对此页表进行可变操作。

我们在这里不需要使用 unsafe 块，因为 Rust 将 `unsafe fn` 整个函数体都视作一个大的 `unsafe` 块。 这使我们的代码更加危险，因为我们可能在不注意的情况下意外引入了不安全的操作。 这也使发现不安全操作变得更加困难。 有一个 [RFC提案](https://github.com/rust-lang/rfcs/pull/2585) 希望可以更改此行为。

现在，我们可以使用此函数来打印 4 级页表的条目：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory::active_level_4_table;
    use x86_64::VirtAddr;

    println!("Hello World{}", "!");
    blog_os::init();

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let l4_table = unsafe { active_level_4_table(phys_mem_offset) };

    for (i, entry) in l4_table.iter().enumerate() {
        if !entry.is_unused() {
            println!("L4 Entry {}: {:?}", i, entry);
        }
    }

    // as before
    #[cfg(test)]
    test_main();

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

首先，我们 将`BootInfo` 结构的`physical_memory_offset` 转换为 `VirtAddr`，并将其传递给`active_level_4_table` 函数。 然后，我们使用 `iter` 函数对页表条目进行迭代，并使用 `enumerate` 组合子为每个元素添加索引 `i`。 我们仅打印非空条目，因为所有512个条目均无法显示在屏幕上。

运行它时，我们看到以下输出：

![QEMU printing entry 0 (0x2000, PRESENT, WRITABLE, ACCESSED), entry 1 (0x894000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 31 (0x88e000, PRESENT, WRITABLE, ACCESSED, DIRTY), entry 175 (0x891000, PRESENT, WRITABLE, ACCESSED, DIRTY), and entry 504 (0x897000, PRESENT, WRITABLE, ACCESSED, DIRTY)](https://os.phil-opp.com/paging-implementation/qemu-print-level-4-table.png)


我们看到有各种非空条目，它们都映射到不同的3级表。 有这么多区域是因为内核代码，内核堆栈，物理内存映射和引导信息都使用隔开的内存区域。

为了进一步遍历页表并查看 3 级表，我们可以将一个条目的映射到的帧再次转换为虚拟地址：

```rust
// in the `for` loop in src/main.rs

use x86_64::structures::paging::PageTable;

if !entry.is_unused() {
    println!("L4 Entry {}: {:?}", i, entry);

    // get the physical address from the entry and convert it
    let phys = entry.frame().unwrap().start_address();
    let virt = phys.as_u64() + boot_info.physical_memory_offset;
    let ptr = VirtAddr::new(virt).as_mut_ptr();
    let l3_table: &PageTable = unsafe { &*ptr };

    // print non-empty entries of the level 3 table
    for (i, entry) in l3_table.iter().enumerate() {
        if !entry.is_unused() {
            println!("  L3 Entry {}: {:?}", i, entry);
        }
    }
}
```

为了查看2级和1级表，我们对 3 级和 2 级条目重复该过程。 您可以想象，这很快就会变得非常冗长，因此我们在这里不显示完整的代码。

手动遍历页表很有趣，因为它有助于了解CPU如何执行转换。 但是，大多数时候我们只对给定虚拟地址的映射物理地址感兴趣，因此让我们为其创建一个函数。

### 地址转换
为了将虚拟地址转换为物理地址，我们必须遍历四级页表，直到到达映射的帧。 让我们创建一个执行此转换的函数：

为了将虚拟地址转换为物理地址，我们必须遍历四级页表，直到到达映射的帧。 让我们创建一个执行此转换的函数：

```rust
// in src/memory.rs

use x86_64::PhysAddr;

/// Translates the given virtual address to the mapped physical address, or
/// `None` if the address is not mapped.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`.
pub unsafe fn translate_addr(addr: VirtAddr, physical_memory_offset: VirtAddr)
    -> Option<PhysAddr>
{
    translate_addr_inner(addr, physical_memory_offset)
}
```

我们将该函数转发给安全的`translate_addr_inner`函数，以限制不安全的范围。 如上所述，Rust将unsafe fn的整个主体视为一个大的unsafe块。 通过调用私有safe函数，我们使每个unsafe操作再次明确。

内部私有函数包含实际的实现：

```rust
// in src/memory.rs

/// Private function that is called by `translate_addr`.
///
/// This function is safe to limit the scope of `unsafe` because Rust treats
/// the whole body of unsafe functions as an unsafe block. This function must
/// only be reachable through `unsafe fn` from outside of this module.
fn translate_addr_inner(addr: VirtAddr, physical_memory_offset: VirtAddr)
    -> Option<PhysAddr>
{
    use x86_64::structures::paging::page_table::FrameError;
    use x86_64::registers::control::Cr3;

    // read the active level 4 frame from the CR3 register
    let (level_4_table_frame, _) = Cr3::read();

    let table_indexes = [
        addr.p4_index(), addr.p3_index(), addr.p2_index(), addr.p1_index()
    ];
    let mut frame = level_4_table_frame;

    // traverse the multi-level page table
    for &index in &table_indexes {
        // convert the frame into a page table reference
        let virt = physical_memory_offset + frame.start_address().as_u64();
        let table_ptr: *const PageTable = virt.as_ptr();
        let table = unsafe {&*table_ptr};

        // read the page table entry and update `frame`
        let entry = &table[index];
        frame = match entry.frame() {
            Ok(frame) => frame,
            Err(FrameError::FrameNotPresent) => return None,
            Err(FrameError::HugeFrame) => panic!("huge pages not supported"),
        };
    }

    // calculate the physical address by adding the page offset
    Some(frame.start_address() + u64::from(addr.page_offset()))
}
```

我们不再重用我们的`active_level_4_table`函数，而是再次从`CR3`寄存器读取4级帧。我们这样做是因为它简化了此原型的实现。不用担心，我们稍后会创建一个更好的解决方案。

`VirtAddr`结构已经提供了将索引计算到四个级别的页表中的方法。我们将这些索引存储在一个小的数组中，因为它允许我们使用for循环遍历页表。在循环之外，我们记得最后访问的帧，以便稍后计算物理地址。该框架在迭代时指向页表框架，并在最后一次迭代后（即在跟随1级条目之后）指向映射的框架。

在循环内部，我们再次使用`physical_memory_offset`将帧转换为页表引用。 然后，我们读取当前页表的条目，并使用`PageTableEntry::frame`函数检索映射的帧。 如果条目未映射到帧，则返回`None`。 如果条目映射了一个`2MiB`或`1GiB`的huge页面，我们现在会panic。

让我们通过翻译一些地址来测试我们的翻译功能：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // new imports
    use blog_os::memory::translate_addr;
    use x86_64::VirtAddr;

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);

    let addresses = [
        // the identity-mapped vga buffer page
        0xb8000,
        // some code page
        0x201008,
        // some stack page
        0x0100_0020_1a10,
        // virtual address mapped to physical address 0
        boot_info.physical_memory_offset,
    ];

    for &address in &addresses {
        let virt = VirtAddr::new(address);
        let phys = unsafe { translate_addr(virt, phys_mem_offset) };
        println!("{:?} -> {:?}", virt, phys);
    }

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

跑一下看看，我们可以看到如下输出：

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, "panicked at 'huge pages not supported'](https://os.phil-opp.com/paging-implementation/qemu-translate-addr.png)

如预期的那样，恒等映射的地址`0xb8000`转换为相同的物理地址。代码页和堆栈页转换为一些任意的物理地址，这取决于引导加载程序如何为内核创建初始映射。值得注意的是，转换后的最后12位始终保持不变，这是有道理的，因为这些位是页面偏移量，而不是转换的一部分。

由于可以通过添加`physical_memory_offset`来访问每个物理地址，因此`physical_memory_offset`地址本身的转换应指向物理地址0。但是，转换失败了，因为该映射使用huge页面来提高效率，这在我们的实现中尚不支持。

### 使用`OffsetPageTable`
将虚拟地址转换为物理地址是OS内核中的常见任务，因此`x86_64` crate为其提供了抽象。该实现已经支持huge 页面和除`translate_addr`之外的其他几个页面表功能，因此我们将在下面使用它，而不是向我们自己的实现添加huge页面支持。

抽象的基础是定义各种页表映射功能的两个trait：

- `Mapper` trait在页面大小上是通用的，并提供可在页面上运行的功能。例如：`translate_page`（将给定页面转换为相同大小的框架）和map_to（在页面表中创建新的映射）。
- `MapperAllSizes`特性意味着实现者为所有页面大小实现Mapper。此外，它提供了适用于多种页面大小的功能，例如`translate_addr`或常规的`translate`。

trait仅定义接口，它们不提供任何实现。当前，`x86_64` crate提供了三种类型，这些类型可实现具有不同要求的特征。 `OffsetPageTable`类型假定完整的物理内存以某个偏移量映射到虚拟地址空间。 `MappedPageTable`稍微灵活一些：它只需要将每个页表帧映射到可计算地址处的虚拟地址空间即可。最后，可以使用`RecursivePageTable`类型通过递归页表访问页表框架。

在我们的例子中，引导加载程序将完整的物理内存映射到由`physical_memory_offset`变量指定的虚拟地址，因此我们可以使用`OffsetPageTable`类型。要初始化它，我们在内存模块中创建一个新的`init`函数：

```rust
use x86_64::structures::paging::OffsetPageTable;

/// Initialize a new OffsetPageTable.
///
/// This function is unsafe because the caller must guarantee that the
/// complete physical memory is mapped to virtual memory at the passed
/// `physical_memory_offset`. Also, this function must be only called once
/// to avoid aliasing `&mut` references (which is undefined behavior).
pub unsafe fn init(physical_memory_offset: VirtAddr) -> OffsetPageTable<'static> {
    let level_4_table = active_level_4_table(physical_memory_offset);
    OffsetPageTable::new(level_4_table, physical_memory_offset)
}

// make private
unsafe fn active_level_4_table(physical_memory_offset: VirtAddr)
    -> &'static mut PageTable
{…}
```

该函数将`physical_memory_offset`作为参数，并返回一个具有`'static`生命周期的新`OffsetPageTable`实例。这意味着实例对于我们的内核的完整运行时保持有效。在函数主体中，我们首先调用`active_level_4_table`函数以检索对第4级页表的可变引用。然后，我们使用此引用调用`OffsetPageTable::new`函数。作为第二个参数，新函数期望在物理内存的映射处开始的虚拟地址，该地址在`physical_memory_offset`变量中给出。

从现在开始，仅应从`init`函数调用`active_level_4_table`函数，因为当多次调用它时，它很容易使用可变的引用，这可能导致未定义的行为。因此，我们通过删除`pub`说明符来使函数私有。

现在，我们可以使用`MapperAllSizes::translate_addr`方法来代替我们自己的`memory::translate_addr`函数。我们只需要在`kernel_main`中更改几行：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // new: different imports
    use blog_os::memory;
    use x86_64::{structures::paging::MapperAllSizes, VirtAddr};

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    // new: initialize a mapper
    let mapper = unsafe { memory::init(phys_mem_offset) };

    let addresses = […]; // same as before

    for &address in &addresses {
        let virt = VirtAddr::new(address);
        // new: use the `mapper.translate_addr` method
        let phys = mapper.translate_addr(virt);
        println!("{:?} -> {:?}", virt, phys);
    }

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

我们需要导入`MapperAllSizes`trait以使用它提供的`translate_addr`方法。

现在运行它时，我们会看到与以前相同的翻译结果，不同之处在于huge页面翻译现在也可以工作：

![0xb8000 -> 0xb8000, 0x201008 -> 0x401008, 0x10000201a10 -> 0x279a10, 0x18000000000 -> 0x0](https://os.phil-opp.com/paging-implementation/qemu-mapper-translate-addr.png)

不出所料，`0xb8000`的转换以及代码和堆栈地址与我们自己的转换功能相同。 此外，我们现在看到虚拟地址`physical_memory_offset`映射到物理地址`0x0`。

通过使用`MappedPageTable`类型的转换功能，我们可以节省实施大型页面支持的工作。 我们还可以访问其他页面功能，例如`map_to`，我们将在下一部分中使用。

此时，我们不再需要`memory::translate_addr`函数，因此可以将其删除。

### 创建一个新映射

到目前为止，我们仅查看页面表，而没有进行任何修改。让我们通过为以前未映射的页面创建一个新的映射来更改它。

我们将使用`Mapper` trait 的`map_to`函数进行实现，因此让我们首先看一下该函数。文档告诉我们，它带有四个参数：我们要映射的页面，该页面应映射到的框架，页面表项的一组标志以及`frame_allocator`。我们需要一个帧分配器，因为映射给定页面可能需要创建其他页面表，这些页面表需要未使用的帧作为后备存储。

#### `create_example_mapping`函数

我们实现的第一步是创建一个新的`create_example_mapping`函数，该函数将给定的虚拟页面映射到`0xb8000`（VGA文本缓冲区的物理帧）。我们选择该帧是因为它使我们能够轻松测试映射是否正确创建：我们只需要向新映射的页面写入，然后查看是否看到写入内容出现在屏幕上。

`create_example_mapping`函数如下所示：

```rust
// in src/memory.rs

use x86_64::{
    PhysAddr,
    structures::paging::{Page, PhysFrame, Mapper, Size4KiB, FrameAllocator}
};

/// Creates an example mapping for the given page to frame `0xb8000`.
pub fn create_example_mapping(
    page: Page,
    mapper: &mut OffsetPageTable,
    frame_allocator: &mut impl FrameAllocator<Size4KiB>,
) {
    use x86_64::structures::paging::PageTableFlags as Flags;

    let frame = PhysFrame::containing_address(PhysAddr::new(0xb8000));
    let flags = Flags::PRESENT | Flags::WRITABLE;

    let map_to_result = unsafe {
        mapper.map_to(page, frame, flags, frame_allocator)
    };
    map_to_result.expect("map_to failed").flush();
}
```

除了应该映射的`page`之外，该函数还需要一个对`OffsetPageTable`实例的可变引用，和一个`frame_allocator`。 `frame_allocator`参数对实现了`FrameAllocator` trait 的所有类型通用。该特征在`PageSize` trait上具有通用性，可与标准4KiB页面和huge的2MiB/1GiB页面一起使用。我们只想创建4KiB映射，因此我们将通用参数设置为`Size4KiB`。

对于映射，我们设置`PRESENT`标志是因为所有有效条目都需要它，而`WRITABLE`标志则使映射的页面可写。调用`map_to`是不安全的，因为有可能使用无效的参数来破坏内存安全性，因此我们需要使用一个`unsafe`块。有关所有可能的标志的列表，请参见上一篇文章的“页面表格式”部分。

`map_to`函数可能会失败，因此它将返回`Result`。由于这只是一些示例代码，不需要鲁棒性，因此我们仅在发生错误时使用`expect`来引发一个panic。成功后，该函数将返回MapperFlush类型，该类型提供了一种使用其flush方法从转换后备缓冲区（TLB）中刷新新映射页面的简便方法。像`Result`一样，当我们意外忘记使用它时，由于使用了`#[must_use]`属性，会发出一个警告。

#### 一个虚拟的FrameAllocator

为了能够调用`create_example_mapping`，我们需要创建一个首先实现`FrameAllocator` Trait的类型。如上所述，如果`map_to`需要帧，则Trait负责为新页表分配帧。

让我们从简单的案例开始，并假设我们不需要创建新的页表。对于这种情况，始终返回`None`的帧分配器就足够了。我们创建了一个`EmptyFrameAllocator`来测试我们的映射功能：

```rust
// in src/memory.rs

/// A FrameAllocator that always returns `None`.
pub struct EmptyFrameAllocator;

unsafe impl FrameAllocator<Size4KiB> for EmptyFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> {
        None
    }
}
```

实现`FrameAllocator`是unsafe的，因为实现者必须保证分配器仅产生未使用的帧。 否则，可能会发生不确定的行为，例如，当两个虚拟页面映射到同一物理框架时。 我们的`EmptyFrameAllocator`只返回`None`，因此在这种情况下这不是问题。

#### 选择虚拟页面

现在，我们有了一个简单的帧分配器，可以将其传递给`create_example_mapping`函数。 但是，分配器始终返回`None`，因此只有在不需要其他页表框架来创建映射时，此分配器才起作用。 要了解何时需要其他页表框架以及何时不需要，请考虑以下示例：

![A virtual and a physical address space with a single mapped page and the page tables of all four levels](https://os.phil-opp.com/paging-implementation/required-page-frames-example.svg)

该图在左侧显示虚拟地址空间，在右侧显示物理地址空间，以及它们之间的页表。页表存储在物理存储帧中，由虚线表示。虚拟地址空间在地址`0x803fe00000`包含一个映射的页面，该页面以蓝色标记。为了将此页面转换为其框架，CPU遍历4级页面表，直到到达地址36 KiB的框架。

此外，该图以红色显示VGA文本缓冲区的物理帧。我们的目标是使用我们的`create_example_mapping`函数将先前未映射的虚拟页面映射到此帧。由于`EmptyFrameAllocator`始终返回`None`，因此我们要创建映射，以便不需要分配器中的其他帧。这取决于我们为映射选择的虚拟页面。

该图显示了虚拟地址空间中的两个候选页面，均以黄色标记。一页位于地址`0x803fdfd000`，即映射页之前的3页（蓝色）。尽管第4级和第3级页表索引与蓝页相同，但第2级和第1级索引却不同（请参阅上一篇文章）。级别2表中的索引不同，意味着此页面使用了不同的级别1表。由于此1级表尚不存在，因此如果我们为示例映射选择该页面，则需要创建该表，这将需要一个额外的未使用的物理框架。相反，位于地址`0x803fe02000`的第二个候选页面不存在此问题，因为它使用与蓝色页面相同的1级页面表。因此，所有必需的页表已经存在。

总而言之，创建新映射的难度取决于我们要映射的虚拟页面。在最简单的情况下，该页面的1级页面表已经存在，我们只需要编写一个条目即可。在最困难的情况下，该页面位于尚不存在第3级的内存区域中，因此我们需要首先创建新的第3级，第2级和第1级页表。

为了使用`EmptyFrameAllocator`调用`create_example_mapping`函数，我们需要选择一个页面，其所有页表均已存在。要找到这样的页面，我们可以利用引导加载程序将自身加载到虚拟地址空间的第一个兆字节中这一事实。这意味着该区域的所有页面都存在一个有效的1级表。因此，我们可以为示例映射选择该存储区域中任何未使用的页面，例如地址0的页面。通常，该页面应保持未使用状态，以确保取消引用空指针会导致页面错误，因此我们知道引导加载程序保持了该页面的未映射状态。

#### 创建映射

现在，我们有了用于调用`create_example_mapping`函数的所有必需参数，因此让我们修改`kernel_main`函数，以将页面映射到虚拟地址0。由于我们将页面映射到VGA文本缓冲区的帧，因此我们应该能够向屏幕写入。实现看起来像这样：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory;
    use x86_64::{structures::paging::Page, VirtAddr}; // new import

    […] // hello world and blog_os::init

    let phys_mem_offset = VirtAddr::new(boot_info.physical_memory_offset);
    let mut mapper = unsafe { memory::init(phys_mem_offset) };
    let mut frame_allocator = memory::EmptyFrameAllocator;

    // map an unused page
    let page = Page::containing_address(VirtAddr::new(0));
    memory::create_example_mapping(page, &mut mapper, &mut frame_allocator);

    // write the string `New!` to the screen through the new mapping
    let page_ptr: *mut u64 = page.start_address().as_mut_ptr();
    unsafe { page_ptr.offset(400).write_volatile(0x_f021_f077_f065_f04e)};

    […] // test_main(), "it did not crash" printing, and hlt_loop()
}
```

我们首先通过调用`create_example_mapping`函数来调用地址0处的页面的映射。 这会将页面映射到VGA文本缓冲区框架，因此我们应该在屏幕上看到对其进行的任何写入。

然后，我们将页面转换为原始指针，并向偏移量400写入一个值。我们不写入页面的开头，因为VGA缓冲区的第一行直接由下一个println移出屏幕。 我们写入值`0x_f021_f077_f065_f04e`，它表示字符串“ New！”。 在白色背景上。 正如我们在“ VGA Text Mode”（VGA文本模式）文章中所了解的那样，对VGA缓冲区的写入应该是易失的，因此我们使用`write_volatile`方法。

在QEMU中运行它时，将看到以下输出：

![QEMU printing "It did not crash!" with four completely white cells in the middle of the screen](https://os.phil-opp.com/paging-implementation/qemu-new-mapping.png)

屏幕上的 "New!" 是通过写入第`0`页来显示的，这意味着我们已在页表中成功创建了新映射。

仅因为负责地址0的页面的1级表已经存在，所以创建该映射才起作用。 当我们尝试为尚不存在1级表的页面进行映射时，`map_to`函数将失败，因为它试图从`EmptyFrameAllocator`分配帧以创建新的页表。 当我们尝试映射页面`0xdeadbeaf000`而不是`0`时，我们可以看到这种情况：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    […]
    let page = Page::containing_address(VirtAddr::new(0xdeadbeaf000));
    […]
}
```

当我们运行它时，会出现以下错误消息：

```rust
panicked at 'map_to failed: FrameAllocationFailed', /…/result.rs:999:5
```

要映射没有1级页面表的页面，我们需要创建一个适当的`FrameAllocator`。 但是我们如何知道哪些帧未使用以及有多少物理内存可用？

#### 分配帧
为了创建新的页表，我们需要创建一个适当帧分配器。 为此，我们使用由引导程序作为`BootInfo`结构的一部分传递的`memory_map`：

```rust
// in src/memory.rs

use bootloader::bootinfo::MemoryMap;

/// A FrameAllocator that returns usable frames from the bootloader's memory map.
pub struct BootInfoFrameAllocator {
    memory_map: &'static MemoryMap,
    next: usize,
}

impl BootInfoFrameAllocator {
    /// Create a FrameAllocator from the passed memory map.
    ///
    /// This function is unsafe because the caller must guarantee that the passed
    /// memory map is valid. The main requirement is that all frames that are marked
    /// as `USABLE` in it are really unused.
    pub unsafe fn init(memory_map: &'static MemoryMap) -> Self {
        BootInfoFrameAllocator {
            memory_map,
            next: 0,
        }
    }
}
```

该结构有两个字段：对引导加载程序传递的内存映射的`'static`引用，以及一个跟踪分配器应返回的下一帧的编号的`next`字段。

如我们在“引导信息”部分所述，内存映射由BIOS / UEFI固件提供。它只能在引导过程的早期被查询，因此引导加载程序已经为我们调用了相应的函数。存储器映射由`MemoryRegion`结构的列表组成，这些结构包含每个存储器区域的起始地址，长度和类型（例如未使用，保留等）。

初始化函数使用给定的内存映射初始化`BootInfoFrameAllocator`。下一个字段用0初始化，并且将在每次帧分配时递增，以避免两次返回同一帧。由于我们不知道内存映射的可用帧是否已在其他地方使用，因此我们的`init`函数必须unsafe才能要求调用者提供其他保证。

#### `usable_frames`方法

在实现`FrameAllocator`特性之前，我们添加了一个辅助方法，该方法将内存映射转换为可用帧的迭代器：

```rust
// in src/memory.rs

use bootloader::bootinfo::MemoryRegionType;

impl BootInfoFrameAllocator {
    /// Returns an iterator over the usable frames specified in the memory map.
    fn usable_frames(&self) -> impl Iterator<Item = PhysFrame> {
        // get usable regions from memory map
        let regions = self.memory_map.iter();
        let usable_regions = regions
            .filter(|r| r.region_type == MemoryRegionType::Usable);
        // map each region to its address range
        let addr_ranges = usable_regions
            .map(|r| r.range.start_addr()..r.range.end_addr());
        // transform to an iterator of frame start addresses
        let frame_addresses = addr_ranges.flat_map(|r| r.step_by(4096));
        // create `PhysFrame` types from the start addresses
        frame_addresses
            .map(|addr|PhysFrame::containing_address(PhysAddr::new(addr)))
    }
}

```

此函数使用迭代器组合子方法将初始`MemoryMap`转换为可用物理帧的迭代器：

- 首先，我们调用`ite`r方法将内存映射转换为MemoryRegions的迭代器。
- 然后，我们使用`filter`方法跳过任何保留的区域或其他不可用的区域。引导加载程序会为其创建的所有映射更新内存映射，因此内核使用的帧（代码，数据或堆栈）或用于存储引导信息的帧已被标记为InUse或类似的。因此，我们可以确定可用框架不会在其他地方使用。
- 之后，我们使用`map`组合子和Rust的range语法将内存区域的迭代器转换为地址范围的迭代器。
- 下一步是最复杂的：我们通过`into_iter`方法将每个范围转换为一个迭代器，然后使用`step_by`选择每个范围内的第4096个地址。由于页面大小为4096字节（= 4 KiB），因此我们获得了每个帧的起始地址。 Bootloader页面会对齐所有可用的内存区域，因此我们在此处不需要任何对齐或舍入代码。通过使用`flat_map`而不是`map`，我们得到了`Iterator <Item = u64>`而不是`Iterator <Item = Iterator <Item = u64 >>`。
- 最后，我们将起始地址转换为`PhysFrame`类型，以构造所需的`Iterator <Item = PhysFrame>`。然后，我们使用此迭代器创建并返回一个新的`BootInfoFrameAllocator`。
  该函数的返回类型使用`impl Trait`功能。这样，我们可以指定返回某种类型为`PhysFrame`的实现`Iterator `trait的类型，而无需命名具体的返回类型。这一点很重要，因为我们无法命名具体类型，因为它取决于不可命名的闭包类型。

#### 实现 `FrameAllocator` Trait

现在我们可以实现`FrameAllocator` trait：

```rust
// in src/memory.rs

unsafe impl FrameAllocator<Size4KiB> for BootInfoFrameAllocator {
    fn allocate_frame(&mut self) -> Option<PhysFrame> {
        let frame = self.usable_frames().nth(self.next);
        self.next += 1;
        frame
    }
}
```

我们首先使用`usable_frames`方法从内存映射中获取可用帧的迭代器。 然后，我们使用`Iterator::nth`函数获取索引为`self.next`的帧（从而跳过`(self.next-1)`帧）。 在返回该帧之前，我们将`self.next`增加一，以便在下一次调用时返回下一个帧。

这种实现方式并不是十分理想，因为它会在每次分配时重新创建`usable_frame`分配器。 最好直接将迭代器存储为`struct`字段。 然后，我们将不需要`nth`方法，而只需对每个分配调用`next`。 这种方法的问题在于，当前无法在`struct`字段中存储`impl Trait`类型。完全实现[具名存在性类型](https://github.com/rust-lang/rfcs/pull/2071)的某天，这个方法可能可以使用。

#### 使用`BootInfoFrameAllocator`

现在，我们可以修改`kernel_main`函数，以传递`BootInfoFrameAllocator`实例而不是`EmptyFrameAllocator`：

```rust
// in src/main.rs

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    use blog_os::memory::BootInfoFrameAllocator;
    […]
    let mut frame_allocator = unsafe {
        BootInfoFrameAllocator::init(&boot_info.memory_map)
    };
    […]
}
```

使用引导信息帧分配器，映射成功了，并且我们看到了黑白“ New！” 再次出现在屏幕上。 在后台，`map_to`方法通过以下方式创建缺少的页表：

- 从传递的`frame_allocator`中分配未使用的帧。
- 将帧内容全部设置为0以创建一个新的空页表。
- 将更高级别的表的条目映射到该框架。
- 继续下一个表格级别。

尽管我们的`create_example_mapping`函数只是一些示例代码，但我们现在能够为任意页面创建新的映射。 这对于在以后的帖子中分配内存或实现多线程至关重要。

## 总结

在这篇文章中，我们了解了访问页表物理帧的各种技术，包括恒等映射，完整物理内存的映射，临时映射和递归页表。 我们选择映射完整的物理内存，因为它简单，可移植且功能强大。

没有页表访问权限，我们无法映射内核中的物理内存，因此我们需要引导加载程序的支持。 引导加载程序板条箱支持通过可选的货物功能创建所需的映射。 它将所需信息以`&BootInfo`参数的形式传递给我们的内核，该参数传递给我们的入口点函数。

对于我们的实现，我们首先手动遍历页表以实现转换功能，然后使用`x86_64`crate 的`MappedPageTable`类型。 我们还学习了如何在页表中创建新的映射，以及如何在引导加载程序传递的内存映射之上创建必要的`FrameAllocator`。

## 接下来？

下一篇文章将为我们的内核创建一个堆内存区域，这将使我们能够分配内存并使用各种集合类型。