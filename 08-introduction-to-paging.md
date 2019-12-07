---
title: 介绍分页
date: 2019-01-31 18:20:38
tags: [Memory Management]
summary: 这篇文章介绍了分页，这是一种非常常见的内存管理方案，我们也将将其用于我们的操作系统。 它解释了为什么需要内存隔离，分段如何工作，虚拟内存是什么，以及分页如何解决内存碎片问题。 它还探讨了x86_64架构上多级页表的布局。
---

## 内存保护

操作系统的一个主要任务是将程序彼此隔离。例如，您的Web浏览器不应该干扰您的文本编辑器。为实现此目标，操作系统利用硬件功能来确保一个进程的内存区域不能被其他进程访问。根据硬件和操作系统的实现，有不同的方法来做到这一点。

例如，某些ARM Cortex-M处理器（通常用于嵌入式系统）具有内存保护单元（MPU），它允许您定义少量（例如8个）内存区域具有的不同访问权限（例如，无访问权限，只读，读写）。在每次内存访问时，MPU确保该地址位于具有正确访问权限的区域中，否则会抛出一个异常。通过在每次切换进程的时候同时也切换内存区域和访问权限，操作系统可以确保每个进程只访问自己的内存，从而将进程彼此隔离。

在x86上，硬件支持两种不同的内存保护方法：分段和分页。

## 分段

分段在1978年就已经引入了，最初是为了增加可寻址内存的数量。当时的情况是CPU只有16位地址线，这将可寻址内存量限制为64KiB。为了访问这64KiB之外的内存，引入了额外的段寄存器，每个寄存器包含一个偏移地址。 CPU会在每次内存访问时自动添加此偏移量，以便访问高达1MiB的内存。

段寄存器由CPU自动选择，具体取决于存储器访问的类型：对于获取指令，使用代码段CS，对于堆栈操作（push/pop），使用堆栈段SS。其他指令使用数据段DS或额外段ES。后来增加了两个额外的段寄存器FS和GS，可以自由使用。

在最初的分段机制设计中，段寄存器直接存储偏移量，并且没有访问控制。后来随着保护模式的引入，这一点改变了。当CPU以此模式运行时，段描述符包含本地或全局描述符表的索引，该表除了包含偏移地址外，还包含段大小和访问权限。通过为每个进程加载单独的全局/本地描述符表来限制对进程自身内存区域的内存访问，操作系统可以将进程彼此隔离。

通过在实际访问之前修改存储器地址，分段机制已经采用了现在几乎在任何地方使用的技术：虚拟内存。

## 虚拟内存

虚拟内存背后的想法是从底层物理存储设备中抽象出内存地址。不是直接访问存储设备，而是首先执行转换步骤。对于分段，转换步骤是添加活动段的偏移地址。想象一个程序访问偏移量为`0x1111000`的段中的内存地址`0x1234000`：真正访问的地址是`0x2345000`。

为了区分这两种地址类型，转换前的地址称为虚拟地址，转换后的地址称为物理地址。这两种地址之间的一个重要区别是物理地址是独一无二的的，并且始终指向相同的，唯一的存储位置。另一方面，虚拟地址取决于翻译功能。两个不同的虚拟地址完全可能指的是相同的物理地址。此外，相同的虚拟地址在使用不同的转换功能时可以指向不同的物理地址。

证明此属性有用的一个例子是并行运行相同的程序两次：

![Two virtual address spaces with address 0–150, one translated to 100–250, the other to 300–450](https://os.phil-opp.com/paging-introduction/segmentation-same-program-twice.svg)

这里相同的程序被运行了两次，但具有不同的地址映射。 第一个实例的段偏移量为100，因此其虚拟地址0-150被转换为物理地址100-250。 第二个实例具有偏移300，其将其虚拟地址0-150转换为物理地址300-450。 这允许两个程序运行相同的代码并使用相同的虚拟地址而不会相互干扰。

另一个优点是，程序现在可以被放置在任意物理内存位置，即使它们使用完全不同的虚拟地址。 因此，OS可以利用全部可用内存而无需重新编译程序。

## 碎片

虚拟地址和物理地址之间的区别使得分段功能非常强大。 但是，它存在碎片问题。 举个例子，假设我们要运行上面看到的程序的第三个副本：

![Three virtual address spaces, but there is not enough continuous space for the third](https://os.phil-opp.com/paging-introduction/segmentation-fragmentation.svg)

即使有足够的可用内存，也无法将程序的第三个实例映射到虚拟内存而不会重叠。 问题是我们需要连续的内存，不能使用小的空闲块。

解决掉这种碎片的一种方法是暂停执行，将存储器的已使用部分移近一些，更新地址映射，然后恢复执行：



![Three virtual address spaces after defragmentation](https://os.phil-opp.com/paging-introduction/segmentation-fragmentation-compacted.svg)

现在有足够的连续空间来启动我们程序的第三个实例了。

这种碎片整理过程的缺点是需要复制大量内存，这会降低系统性能。 在内存变得过于分散之前，还需要定期进行。 这使得系统的行为变得不可预测，因为程序可能在任何时间暂停并且可能无响应[^1]。

碎片问题是大多数操作系统不再使用分段的原因之一。 实际上，x86的64位模式甚至不再支持分段。 而是使用分页，这完全避免了碎片问题。

## 分页

我们的想法是将虚拟和物理内存空间分成小的固定大小的块。 虚拟存储器空间的块称为页面，物理地址空间的块称为帧。 每个页面可以单独映射到一个帧，这样就可以跨越非连续的物理帧分割更大的内存区域。

如果我们回顾碎片化内存空间的示例，但是这次使用分页而不是分段，这一点的优势变得明显：

![With paging the third program instance can be split across many smaller physical areas](https://os.phil-opp.com/paging-introduction/paging-fragmentation.svg)

在这个例子中，我们的页面大小为50字节，这意味着我们的每个内存区域分为三页。 每个页面都单独映射到一个帧，因此连续的虚拟内存区域可以映射到非连续的物理帧。 这允许我们在不执行任何碎片整理的情况下启动程序的第三个实例。

## 隐藏的碎片

与分段相比，分页使用许多小的固定大小的存储区域而不是几个大的可变大小的区域。由于每个帧具有相同的大小，因此没有太小的帧不能使用，因此不会发生碎片。

似乎没有碎片会出现。但是事实上仍然会存在一些隐藏的碎片，即所谓的内部碎片。发生内部碎片是因为并非每个内存区域都是页面大小的精确倍数。想象一下上面例子中一个大小为101的程序：它仍然需要三个大小为50的页面，因此它将比所需的多占用49个字节。为了区分这两种类型的碎片，使用分段时发生的碎片类型称为外部碎片。

出现内部碎片是很不幸的，但通常比使用分段时出现的外部碎片更好。它仍然浪费内存，但不需要进行碎片整理，并且可以预测碎片量（平均每个内存区域半页）。 

## 页表

我们看到可能有数百万个页面被映射到单独的一个帧。 此映射信息需要存储在某处。 分段对每个活动的内存区域使用单独的段选择器寄存器，这对于分页是不可能的，因为页面的数量远多于寄存器。 相反，分页使用称为页表的表结构来存储映射信息。

对于上面的示例，页表将如下所示：

![Three page tables, one for each program instance. For instance 1 the mapping is 0->100, 50->150, 100->200. For instance 2 it is 0->300, 50->350, 100->400. For instance 3 it is 0->250, 50->450, 100->500.](https://os.phil-opp.com/paging-introduction/paging-page-tables.svg)

我们看到每个程序实例都有自己的页表。 指向当前活跃的页表的指针存储在特殊CPU寄存器中。 在x86上，该寄存器称为CR3。 在运行每个程序实例之前，操作系统要将指向正确页表的指针加载到该寄存器。

在每次访问内存时，CPU从寄存器中读取页表指针，并在表中查找要访问页面的映射到的帧。 这完全由硬件完成，对运行的程序完全透明。 为了加快转换过程，许多CPU架构都有一个特殊的缓存，可以记住上次翻译的结果。

在一些体系结构上，页表条目还可以在标志字段中存储诸如访问权限之类的属性。 在上面的例子中，“r/w”标志表示页面既可读又可写。

## 多级页表

我们刚刚看到的简单页表在较大的地址空间中存在问题：它们本身要占用很多内存。 例如，假设一个程序使用四个虚拟页面0,1_000_000,1_000_050和1_000_100（我们使用_作为千位分隔符）：

![Page 0 mapped to frame 0 and pages 1_000_000–1_000_150 mapped to frames 100–250](https://os.phil-opp.com/paging-introduction/single-level-page-table.svg)

这个程序运行只需要4个物理帧，但页表中有超过一百万个条目。 我们不能省略空条目，因为那样的话CPU在翻译过程中不再能直接跳转到表中的正确条目（例如，不再保证第四页的数据在第四个条目）[^2]。

为了减少浪费的内存，我们可以使用**两级页表**。 我们的想法是我们为不同的地址区域使用不同的页表。 另一个称为2级页表的表包含虚拟地址区域和1级页表之间的映射。

最好用一个例子来解释。 让我们定义每个1级页面表负责一个大小为10_000的区域。 然后，上面的示例映射将存在以下表：

![Page 0 points to entry 0 of the level 2 page table, which points to the level 1 page table T1. The first entry of T1 points to frame 0, the other entries are empty. Pages 1_000_000–1_000_150 point to the 100th entry of the level 2 page table, which points to a different level 1 page table T2. The first three entries of T2 point to frames 100–250, the other entries are empty.](https://os.phil-opp.com/paging-introduction/multilevel-page-table.svg)

第0页在第一个10_000字节区域中，因此它使用2级页表的第一个条目。此条目指向1级页表T1，它指出页0指向的是第0帧。

页面1_000_000,1_000_050和1_000_100都属于第100个10_000字节区域，因此它们使用2级页面表的第100个条目。该条目指向另一个的1级页表T2，其将三个页面映射到帧100,150和200。注意，1级页表中的页面地址不包括区域偏移，因此例如，一级页表中第1_000_050页的条目就只是50（而非1_000_050）。

我们在2级页表中仍然有100个空条目，但比之前的百万个空条目少得多。这种节省的原因是我们不需要为10_000和1_000_000之间的未映射内存区域创建1级页面表。

两级页表的原理可以扩展到三级，四级或更多级。然后页表寄存器指向最高级别表，该表指向下一个较低级别的表，再指向下一个较低级别的表，依此类推。然后，1级页面表指向映射的帧。这整个原理被称为多级或分层页表。

现在我们知道分页和多级页表如何工作，我们可以看一下如何在x86_64架构中实现分页（我们假设CPU在64位模式下运行）。

## x86_64下的分页

x86_64体系结构使用4级页表，页面大小为4KiB。 每个页表，无论是哪级页表，具有固定大小512个条目。 每个条目的大小为8个字节，因此每个页表大小都为512 * 8B = 4KiB大，恰好能装入一个页面。

每个级别的页表索引可以直接从虚拟地址中读出：

![Bits 0–12 are the page offset, bits 12–21 the level 1 index, bits 21–30 the level 2 index, bits 30–39 the level 3 index, and bits 39–48 the level 4 index](https://os.phil-opp.com/paging-introduction/x86_64-table-indices-from-address.svg)

我们看到每个表索引由9位组成，这是因为每个表有2^9 = 512个条目。 最低的12位是4KiB页面中的偏移（2^12字节= 4KiB）。48到64位没用，这意味着x86_64实际上不是64位，因为它只支持48位地址。 有计划通过5级页表将地址大小扩展到57位，但是还没有支持此功能的处理器。

即使48到64位不被使用，也不能将它们设置为任意值。 相反，此范围内的所有位必须是第47位的副本，以保持地址的唯一性，并允许未来的扩展，如5级页表。 这称为符号扩展，因为它与二进制补码中的符号扩展非常相似。 如果地址未正确进行符号扩展，则CPU会抛出异常。

## 地址翻译的示例

让我们通过一个例子来了解地址翻译过程的详细工作原理：

![An example 4-level page hierarchy with each page table shown in physical memory](https://os.phil-opp.com/paging-introduction/x86_64-page-table-translation.svg)

当前活跃的的4级页表的物理地址（这个4级页表的基地址）存储在CR3寄存器中。 然后，每个页表条目指向下一级表的物理帧。 然后，1级页表的条目指向映射到的帧。 请注意，页表中的所有地址都是物理的而不是虚拟的，否则CPU也需要转换这些地址，导致永无止境的递归。

上面的页表层次结构映射了两个页面（用蓝色表示）。 从页表索引我们可以推断出这两个页面的虚拟地址是`0x803FE7F000`和`0x803FE00000`。 让我们看看当程序试图从地址`0x803FE7F5CE`读取时会发生什么。 首先，我们将地址转换为二进制，并确定页表索引和地址的页面偏移量：

![The sign extension bits are all 0, the level 4 index is 1, the level 3 index is 0, the level 2 index is 511, the level 1 index is 127, and the page offset is 0x5ce](https://os.phil-opp.com/paging-introduction/x86_64-page-table-translation-addresses.png)

使用这些页表索引，我们现在可以遍历页表层次结构以确定地址的映射帧：

- 我们首先从CR3寄存器中读取4级页表的地址。
- 4级索引是1，所以我们查看该表的索引1的条目，它告诉我们3级页表存储在物理地址16KiB处。
- 我们从该地址加载3级表，并查看索引为0的条目，它将我们指向物理地址24KiB处的2级页表。
- 2级索引是511，因此我们查看该页面的最后一个条目以找出1级页表的地址。
- 通过级别1表的索引127的条目，我们最终发现页面映射到物理地址为12KiB（或十六进制的0xc000）的帧。
- 最后一步是将页面偏移量加到获得的帧的基地址中，以获得最终的物理地址0xc000 + 0x5ce = 0xc5ce。

![The same example 4-level page hierarchy with 5 additional arrows: "Step 0" from the CR3 register to the level 4 table, "Step 1" from the level 4 entry to the level 3 table, "Step 2" from the level 3 entry to the level 2 table, "Step 3" from the level 2 entry to the level 1 table, and "Step 4" from the level 1 table to the mapped frames.](https://os.phil-opp.com/paging-introduction/x86_64-page-table-translation-steps.svg)

1级页表中页面的权限是`r`，表示只读。 硬件会强制保证这些权限，如果我们尝试写入该页面，则会抛出异常。 更高级别页面中的权限限制了较低级别的权限，因此如果我们将3级页表中的条目设置为只读，则使用此条目的页面都不可写，即使较低级别指定读/写权限也是如此。

重要的是要注意，尽管此示例仅使用每个表的单个实例，但每个地址空间中通常存在多个每个级别页表的实例。 最多，有：

- 一个4级页表
- 512个3级页表（因为4级页表有512个条目）
- 512*512个2级页表（因为512个3级页表中的每一个都有512个条目）
- 512 * 512 * 512个1级页表（每个2级页表512个条目）

## 页表格式

x86_64体系结构上的页表基本上是512个条目的数组。 在Rust语法中：

```rust
#[repr(align(4096))]
pub struct PageTable {
    entries: [PageTableEntry; 512],
}
```

如repr属性所示，页表需要页面对齐，即在4KiB边界上进行内存对齐。 此要求可确保页表始终填充整个页面，并允许进行优化，使条目非常紧凑。

每个条目都是8字节（64位）大，具有以下格式：

| Bit(s) | Name                  | Meaning                                                      |
| ------ | --------------------- | ------------------------------------------------------------ |
| 0      | present               | 这个页面是否正在内存中                                       |
| 1      | writable              | 这个页面是否可写                                             |
| 2      | user accessible       | 这个页面是否可以被用户态访问                                 |
| 3      | write through caching | 对这个页面的写入是否直接进入内存（不经过cache）              |
| 4      | disable cache         | 是否完全禁止使用cache                                        |
| 5      | accessed              | 当这个页面正在被使用时，这个位会被CPU自动设置                |
| 6      | dirty                 | 当这个页面有被写入时，CPU会自动被CPU设置                     |
| 7      | huge page/null        | 在 1级和4级页表中必须为0，在3级页表中会创建1GiB的内存页，在2级页表中会创建2MiB的内存页 |
| 8      | global                | 地址空间切换时，页面不会被换出cache ( CR4 中的PGE 位必须被设置) |
| 9-11   | available             | OS可以随意使用                                               |
| 12-51  | physical address      | 物理帧地址或下一个页表的地址                                 |
| 52-62  | available             | OS可以随意使用                                               |
| 63     | no execute            | 禁止将这个页面上的数据当作代码执行 (EFER寄存器中的NXE位必须被设置) |

我们看到只有12-51位用于存储物理帧地址，其余位用作标志或可由操作系统自由使用。 这是因为我们总是指向一个4096字节的对齐地址，可以是页面对齐的页表，也可以是映射到的帧的开头。 这意味着0-11位始终为零，因此没有理由存储这些位，因为硬件可以在使用地址之前将它们设置为零。 对于位52-63也是如此，因为x86_64架构仅支持52位物理地址（类似于它仅支持48位虚拟地址）。

让我们仔细看看可用的标志位：

- `present`标志将映射过的页面与未映射页面区分开来。当主内存已满时，它可用于临时将页面换出到磁盘。随后访问页面时，会发生一个称为缺页异常的特殊异常，操作系统可以从磁盘重新加载缺少的页面然后继续执行该程序。
- `writable`和`no execute`标志分别控制页面内容是否可写和是否包含可执行指令。
- 当对页面进行读取或写入时，CPU会自动设置`accessed`和`dirty`标志。该信息可以被操作系统所利用，例如确定自上次保存到磁盘后要换出的页面或页面内容是否被修改。
- 通过`write through caching`和 `disable cache` 标志的写入允许单独控制每个页面的缓存。
-  `user accessible` 标志使页面可用于用户态的代码，否则只有在CPU处于内核态时才可访问该页面。通过在用户空间程序运行时保持内核映射，此功能可用于更快地进行系统调用。但是，Spectre漏洞可以允许用户空间程序读取这些页面。
- `global`标志告诉硬件这个页在所有地址空间中都可用，因此不需要从地址空间切换的高速缓存中删除（请参阅下面有关TLB的部分）。该标志通常与用户可访问标志（设置为0）一起使用，以将内核代码映射到所有地址空间。
-  `huge page` 标志允许通过让级别2或级别3页表的条目直接指向映射的帧来创建更大尺寸的页面。设置此位后，对于2级条目，页面大小增加512倍至2MiB = 512 * 4KiB，对于3级条目，页面大小甚至增加到了1GiB = 512 * 2MiB。使用较大页面的优点是需要更少的地址切换缓存行和更少的页表。

`x86_64` crate为页表及其条目提供了类型，因此我们不需要自己创建这些结构。

## 转译后备缓冲器[^3]

4级页表使得虚拟地址的转换变得昂贵，因为每次地址翻译需要4次内存访问。为了提高性能，x86_64架构将最后几个转换缓存在所谓的转译后备缓冲器（TLB）中。这允许在仍然某个地址翻译仍然在缓存中时跳过翻译。

与其他CPU缓存不同，TLB不是完全透明的，并且在页表的内容发生更改时不会更新或删除缓存的转换规则。这意味着内核必须在修改页表时手动更新TLB。为此，有一个名为`invlpg`的特殊CPU指令（“invalidate page”），用于从TLB中删除指定页面的转换规则，下次访问时这个转换规则将从页表中从新加载。通过重新设置CR3寄存器，假装进行一次地址空间转换，也可以完全刷新TLB。 `x86_64` crate在tlb模块中提供了实现这两个功能的Rust函数。

重要的是要记住在每个页表修改时也要同时刷新TLB，否则CPU可能会继续使用旧的转换规则，这可能导致不确定的错误，这些错误很难调试。

## 实现

我们还没有提到的一件事：**我们的内核已经有分页机制了**。 我们在“A minimal Rust Kernel”那一篇文章中添加的引导加载程序已经设置了一个4级分页层次结构，它将我们内核的每个页面映射到一个物理帧。 bootloader程序执行此操作是因为在x86_64上的64位模式下必须进行分页。

这意味着我们在内核中使用的每个内存地址都是一个虚拟地址。 访问地址为0xb8000的VGA缓冲区能用，这是因为bootloader程序已经将该内存页映射到本身了，这意味着它将虚拟页0xb8000映射到物理帧0xb8000。

分页使我们的内核已经相对安全，因为每个超出范围的内存访问都会导致页面错误异常，而不是写入随机的物理内存。 引导加载程序甚至为每个页面设置了正确的访问权限，这意味着只有包含代码的页面是可执行的，只有数据页面是可写的。

## 页面错误

让我们尝试通过访问内核之外的一些内存来导致页面错误。 首先，我们创建一个页面错误处理程序并在我们的IDT中注册它，以便我们看到page fault exception而不是通用的double fault：

```rust
// in src/interrupts.rs

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();

        […]

        idt.page_fault.set_handler_fn(page_fault_handler); // new

        idt
    };
}

use x86_64::structures::idt::PageFaultErrorCode;

extern "x86-interrupt" fn page_fault_handler(
    stack_frame: &mut ExceptionStackFrame,
    _error_code: PageFaultErrorCode,
) {
    use crate::hlt_loop;
    use x86_64::registers::control::Cr2;

    println!("EXCEPTION: PAGE FAULT");
    println!("Accessed Address: {:?}", Cr2::read());
    println!("{:#?}", stack_frame);
    hlt_loop();
}
```

`CR2`寄存器由CPU在页面错误时自动设置，并包含导致页面错误的虚拟地址。 我们使用`x86_64` crate 的`Cr2 :: read`函数来读取和打印它。 通常，`PageFaultErrorCode`类型将提供有关导致页面错误的内存访问类型的更多信息，但目前有一个传递无效错误代码的LLVM bug[^4]，因此我们暂时忽略它。 我们无法在不解决页面错误的情况下继续执行程序，因此我们最后会进入一个`hlt_loop`。

现在我们可以尝试访问内核之外的一些内存：

```rust
// in src/main.rs

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    use blog_os::interrupts::PICS;

    println!("Hello World{}", "!");

    // set up the IDT first, otherwise we would enter a boot loop instead of
    // invoking our page fault handler
    blog_os::gdt::init();
    blog_os::interrupts::init_idt();
    unsafe { PICS.lock().initialize() };
    x86_64::instructions::interrupts::enable();

    // new
    let ptr = 0xdeadbeaf as *mut u32;
    unsafe { *ptr = 42; }

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

当我们运行这个程序，我们可以看到页面错误异常的回调函数被调用了：

![EXCEPTION: Page Fault, Accessed Address: VirtAddr(0xdeadbeaf), ExceptionStackFrame: {…}](https://os.phil-opp.com/paging-introduction/qemu-page-fault.png)

`CR2`寄存器确实包含`0xdeadbeaf`，我们试图访问的地址。

我们看到当前指令指针是`0x20430a`，所以我们可以知道这个地址指向一个代码页。 代码页由引导加载程序以只读方式映射，因此从该地址读取有效但写入会导致页面错误。 您可以通过将`0xdeadbeaf`指针更改为`0x20430a`来尝试此操作：

```rust
// Note: The actual address might be different for you. Use the address that
// your page fault handler reports.
let ptr = 0x20430a as *mut u32;
// read from a code page -> works
unsafe { let x = *ptr; }
// write to a code page -> page fault
unsafe { *ptr = 42; }
```

注释掉最后一行，我们可以看出读操作成功了，但写操作会导致一个页面异常。

## 读取页表

让我们试着看看我们的内核运行时用的页面表：

```rust
// in src/main.rs

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    […] // initialize GDT, IDT, PICS

    use x86_64::registers::control::Cr3;

    let (level_4_page_table, _) = Cr3::read();
    println!("Level 4 page table at: {:?}", level_4_page_table.start_address());

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

`x86_64`的`Cr3::read`函数从`CR3`寄存器返回当前活动的4级页表。 它返回`PhysFrame`和`Cr3Flags`类型的元组。 我们只对`PhysFrame`感兴趣，所以我们忽略了元组的第二个元素。

当我们运行它时，我们看到以下输出：

```shell
Level 4 page table at: PhysAddr(0x1000)
```

因此，当前活动的4级页表存储在物理内存中的地址0x1000处，如`PhysAddr`包装器类型所示。现在的问题是：我们如何从内核访问该表？

当分页处于活动状态时，无法直接访问物理内存，因为程序可以轻松地绕过内存保护并访问其他程序的内存。因此，访问该表的唯一方法是通过一些映射到地址`0x1000`处的物理帧的虚拟页面。为页表帧创建映射的这个问题是一个普遍问题，因为内核需要定期访问页表，例如在为新线程分配堆栈时。

下一篇文章将详细解释此问题的解决方案。现在，只需要知道引导加载程序使用称为递归页表的技术将虚拟地址空间的最后一页映射到4级页表的物理帧就足够了。虚拟地址空间的最后一页是`0xffff_ffff_ffff_f000`，所以让我们用它来读取该表的一些条目：

```rust
// in src/main.rs

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    […] // initialize GDT, IDT, PICS

    let level_4_table_pointer = 0xffff_ffff_ffff_f000 as *const u64;
    for i in 0..10 {
        let entry = unsafe { *level_4_table_pointer.offset(i) };
        println!("Entry {}: {:#x}", i, entry);
    }

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

我们将最后一个虚拟页面的地址转换为指向`u64`类型的指针。 正如我们在上一节中看到的，每个页表项都是8个字节（64位），因此一个`u64`只代表一个条目。 我们使用`for`循环打印表的前10个条目。 在循环内部，我们使用`unsafe`块来读取原始指针和`offset	`方法来执行指针运算。

当我们运行它时，我们看到以下输出：

![Entry 0: 0x2023, Entry 1: 0x6e2063, Entry 2-9: 0x0](https://os.phil-opp.com/paging-introduction/qemu-print-p4-entries.png)

当我们查看页表条目的格式时，我们看到条目0的值`0x2023`意味着该条目`present`，`writable`，由CPU `accessed`，并映射到帧`0x2000`。 条目1映射到帧`0x6e2000`并且具有与条目0相同的标志，并添加了表示页面已写入的`dirty`标志。 条目2-9不`present`，因此这些虚拟地址范围不会映射到任何物理地址。

我们可以使用`x86_64`crate 的`PageTable`类型，而不是使用不安全的原始指针：

```rust
// in src/main.rs

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    […] // initialize GDT, IDT, PICS

    use x86_64::structures::paging::PageTable;

    let level_4_table_ptr = 0xffff_ffff_ffff_f000 as *const PageTable;
    let level_4_table = unsafe {&*level_4_table_ptr};
    for i in 0..10 {
        println!("Entry {}: {:?}", i, level_4_table[i]);
    }

    println!("It did not crash!");
    blog_os::hlt_loop();
}
```

这里我们首先将`0xffff_ffff_ffff_f000`指针转换为原始指针，然后将其转换为Rust引用。 此操作仍然需要`unsafe`，因为编译器无法知道访问此地址的有效性。 但是在转换之后，我们有一个安全的`PageTable`类型，它允许我们通过安全的，有边界检查的索引操作来访问各个条目。

crate还为各个条目提供了一些抽象，以便我们在打印它们时直接看到设置了哪些标志：

![ Entry 0: PageTableEntry { addr: PhysAddr(0x2000), flags: PRESENT | WRITABLE | ACCCESSED } Entry 1: PageTableEntry { addr: PhysAddr(0x6e5000), flags: PRESENT | WRITABLE | ACCESSED | DIRTY } Entry 2: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 3: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 4: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 5: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 6: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 7: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 8: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)} Entry 9: PageTableEntry { addr: PhysAddr(0x0), flags: (empty)}](https://os.phil-opp.com/paging-introduction/qemu-print-p4-entries-abstraction.png)

下一步是遵循条目0或条目1中的指针到3级页表。 但我们现在将再次遇到`0x2000`和`0x6e5000`是物理地址的问题，因此我们无法直接访问它们。 这个问题将在下一篇文章中解决。

## 总结

这个帖子介绍了两种内存保护技术：分段和分页。 前者使用可变大小的内存区域并且受到外部碎片的影响，后者使用固定大小的页面，并允许对访问权限进行更细粒度的控制。

分页存储具有一个或多个级别的页表中的页面的映射信息。 x86_64体系结构使用4级页表，页面大小为4KiB。 硬件自动遍历页表并在转译后备缓冲区（TLB）中缓存生成的转译规则。 此缓冲区不是透明的，需要在页表更改时手动刷新。

我们了解到我们的内核已经在分页之上运行，并且非法内存访问会导致页面错误异常。 我们尝试访问当前活动的页表，但我们只能访问4级表，因为页表存储了我们无法直接从内核访问的物理地址。

## 接下来是什么？

下一篇文章建立在我们在这篇文章中学到的基础知识的基础上。 它引入了一种称为递归页表的高级技术，以解决从我们的内核访问页表的问题。 这允许我们遍历页表层次结构并实现基于软件的翻译功能。 该帖子还解释了如何在页表中创建新映射。

[^1]: @各大GC，尤其是某个会Stop the world 的GC
[^2]: 即地址翻译的过程不再是O(1)，对于一个每条指令的运行都要进行（甚至进行多次）的工作来说，O(1)真的很重要！
[^3]: 这是维基百科上的译名（还是想吐槽：这什么鬼……），常见的译名是“页表缓存”
[^4]: 我想这就是为啥作者拖更了那么多时间