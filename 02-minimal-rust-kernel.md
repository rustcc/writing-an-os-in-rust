>原文：https://os.phil-opp.com/minimal-rust-kernel/
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust创造操作系统（二）：第一个操作系统内核

在这篇文章中，我们将基于**x86架构**（the x86 architecture），使用Rust语言，创造一个最小化的64位操作系统内核。我们将从上一章中构建的独立式可执行程序开始，构建自己的内核；它将向显示器打印字符串，并能被打包为一个能够引导启动的**磁盘映像**（disk image）。

## 引导启动

当我们启动电脑时，主板ROM内存储的**固件**（firmware）将会运行：它将负责电脑的**上电自检**（power-on self test），**可用内存**（available RAM）的检测，以及CPU和其它硬件的预加载。这之后，它将寻找一个**可引导的存储介质**（bootable disk），并开始引导启动其中的操作系统内核。

x86架构支持两种固件标准：**BIOS**（Basic Input/Output System）和**UEFI**（Unified Extensible Firmware Interface）。其中，BIOS标准显得陈旧而过时，但实现简单，并为1980年代后的所有x86设备所支持；相反地，UEFI更现代化，功能也更全面，但开发和构建更复杂（至少从我的角度看是如此）。

在这篇文章中，我们暂时只提供BIOS固件的引导启动方式。

## BIOS启动

几乎所有的x86硬件系统都支持BIOS启动，这也包含新式的、基于UEFI、用**模拟BIOS**（emulated BIOS）的方式向后兼容的硬件系统。这可以说是一件好事情，因为无论是上世纪还是现在的硬件系统，你都只需编写同样的引导启动逻辑；但这种兼容性有时也是BIOS引导启动最大的缺点，因为这意味着在系统启动前，你的CPU必须先进入一个16位系统兼容的**实模式**（real mode），这样1980年代古老的引导固件才能够继续使用。

让我们从头开始，理解一遍BIOS启动的过程。

当电脑启动时，主板上特殊的闪存中存储的BIOS固件将被加载。BIOS固件将会上电自检、初始化硬件，然后它将寻找一个可引导的存储介质。如果找到了，那电脑的控制权将被转交给**引导程序**（bootloader）：一段存储在存储介质的开头的、512字节长度的程序片段。大多数的引导程序长度都大于512字节——所以通常情况下，引导程序都被切分为一段优先启动、长度不超过512字节、存储在介质开头的**第一阶段引导程序**（first stage bootloader），和一段随后由其加载的、长度可能较长、存储在其它位置的**第二阶段引导程序**（second stage bootloader）。

引导程序必须决定操作系统内核的位置，并将内核加载到内存。引导程序还需要将CPU从16位的实模式，先切换到32位的**保护模式**（protected mode），最终切换到64位的**长模式**（long mode）：此时，所有的64位寄存器和整个**主内存**（main memory）才能被访问。引导程序的第三个作用，是从BIOS查询特定的信息，并将其传递到操作系统内核；如查询和传递**内存映射表**（memory map）。

编写一个引导程序并不是一个简单的任务，因为这需要使用汇编语言，而且必须经过许多意图并不显然的步骤——比如，把一些**魔术数字**（magic number）写入某个寄存器。因此，我们不会讲解如何编写自己的引导程序，而是推荐bootimage工具——它能够自动而方便地为你的内核准备一个引导程序。

## Multiboot标准

每个操作系统都实现自己的引导程序，而这只对单个操作系统有效。为了避免这样的僵局，1995年，**自由软件基金会**（Free Software Foundation）颁布了一个开源的引导程序标准——Multiboot。这个标准定义了引导程序和操作系统间的统一接口，所以任何适配Multiboot的引导程序，都能用来加载任何同样适配了Multiboot的操作系统。GNU GRUB是一个可供参考的Multiboot实现，它也是最热门的Linux系统引导程序之一。

要创造一款适配Multiboot的操作系统内核，我们只需要在内核文件开头，插入被称作**Multiboot头**（Multiboot header）的数据片段。这让GRUB很容易引导任何操作系统，但是，GRUB和Multiboot标准也有一些可预知的问题：

1. 它们只支持32位的保护模式。这意味着，在引导之后，你依然需要配置你的CPU，让它切换到64位的长模式；
2. 他们被设计为让引导程序最精简，而不是让操作系统系统内核最精简。举个栗子，操作系统内核需要以调整过的**默认页长度**（default page size）被链接，否则GRUB将无法找到内核的Multiboot头。另一个例子是**引导信息**（boot information），这个包含着大量与架构有关的数据，会在引导启动时，被直接传到操作系统，而不会经过一层清晰的抽象；
3. GRUB和Multiboot标准并没有被详细地注释，阅读相关文档需要一定经验；
4. 为了创建一个能够被引导的磁盘映像，我们在开发时必须安装GRUB：这加大了基于Windows或macOS开发操作系统内核的难度。

出于这些考虑，我们决定不使用GRUB或者Multiboot标准。然而，Multiboot支持功能也在bootimage工具的开发计划之中，所以从原理上讲，如果选用bootimage工具，在未来使用GRUB引导你的系统内核是可能的。

## 最小化的操作系统内核

现在我们已经明白电脑是如何启动的，那也是时候创造我们自己的操作系统内核了。我们的小目标是，创建一个内核的磁盘映像，它能够在启动时，向屏幕输出一行“Hello World!”。我们的工作将基于上一章构建的独立式可执行程序。

如果读者还有印象的话，在上一章，我们使用`cargo`构建了一个独立的二进制程序；但这个程序依然基于特定的操作系统平台：因平台而异，我们需要定义不同名称的函数，需要使用不同的编译指令。这是因为在默认情况下，`cargo`会为特定的**宿主系统**（host system）构建源码，比如为你正在运行的系统构建源码。这并不是我们想要的，因为我们的内核不应该基于另一个操作系统——我们想要创造的，就是这个操作系统。确切地说，我们想要的是，编译为一个特定的**目标系统**（target system）。

## 目标系统配置清单

通过`--target`参数，`cargo`支持不同的目标系统。这个目标系统可以使用一个**目标三元组**（target triple）来描述，它描述了CPU架构、平台供应者、操作系统和**应用程序二进制接口**（Application Binary Interface, ABI）。比方说，目标三元组`x86_64-unknown-linux-gnu`描述一个基于`x86_64`架构CPU的、没有明确的平台供应者的linux系统，它遵循GNU风格的ABI。Rust支持许多不同的目标三元组，包括安卓系统对应的`arm-linux-androideabi`和WebAssembly使用的`wasm32-unknown-unknown`。

为了编写我们的目标系统，鉴于我们需要做一些特殊的配置（比如没有依赖的底层操作系统），已经支持的目标三元组都不能满足我们的要求。幸运的是，只需使用一个JSON文件，Rust便允许我们定义自己的目标系统；这个文件常被称作**目标系统配置清单**（target specification）。比如，一个描述`x86_64-unknown-linux-gnu`目标系统的配置清单大概长这样：

```json
{
    "llvm-target": "x86_64-unknown-linux-gnu",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "linux",
    "executables": true,
    "linker-flavor": "gcc",
    "pre-link-args": ["-m64"],
    "morestack": false
}
```

一个配置清单中包含多个**配置项**（field）。大多数的配置项都是LLVM需求的，它们将配置为特定平台生成的代码。打个比方，`data-layout`配置项定义了不同的整数、浮点数、指针类型的长度；另外，还有一些Rust是用作条件变编译的配置项，如`target-pointer-width`。还有一些类型的配置项，定义了这个包该如何被编译，例如，`pre-link-args`配置项指定了该向链接器传入的参数。

我们将把我们的操作系统内核编译到`x86_64`架构，所以我们的配置清单将和上面的例子相似。现在，我们来创建一个名为`x86_64-blog_os.json`的文件——当然也可以选用自己喜欢的文件名——里面包含这样的内容：

```json
{
    "llvm-target": "x86_64-unknown-none",
    "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
    "arch": "x86_64",
    "target-endian": "little",
    "target-pointer-width": "64",
    "target-c-int-width": "32",
    "os": "none",
    "executables": true,
}
```

需要注意的是，因为我们要在**裸机**（bare metal）上运行内核，我们已经修改了`llvm-target`的内容，并将`os`配置项的值改为`none`。

我们还需要添加下面与编译相关的配置项：

```json
"linker-flavor": "ld.lld",
"linker": "rust-lld",
```

在这里，我们不使用平台默认提供的链接器，因为它可能不支持Linux目标系统。为了链接我们的内核，我们使用跨平台的**LLD链接器**（LLD linker），它是和Rust打包发布的。

```json
"panic-strategy": "abort",
```

这个配置项的意思是，我们的编译目标不支持panic时的**栈展开**（stack unwinding），所以我们选择直接**在panic时中止**（abort on panic）。这和在`Cargo.toml`文件中添加`panic = "abort"`选项的作用是相同的，所以我们可以不在这里的配置清单中填写这一项。

```json
"disable-redzone": true,
```

我们正在编写一个内核，所以我们应该同时处理中断。要安全地实现这一点，我们必须禁用一个称作红色区域（redzone）的栈指针优化