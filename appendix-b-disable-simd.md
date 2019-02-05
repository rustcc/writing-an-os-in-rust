>原文：https://os.phil-opp.com/disable-simd/
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust创造操作系统（附录二）：禁用SIMD

**单指令多数据流**（[Single Instruction Multiple Data，SIMD](https://en.wikipedia.org/wiki/SIMD)）指令能够同时对多个**数据字**（data word）执行同一个操作，这能显著地加快程序运行的速度。`x86_64`架构支持下面的SIMD标准：

1. **MMX**（[Multi Media Extension](https://en.wikipedia.org/wiki/MMX_(instruction_set))）。MMX指令集发布于1997年，它定义了8个64位的寄存器，从`mm0`到`mm7`。这些寄存器只是**x87浮点数单元**（[x87 floating point unit](https://en.wikipedia.org/wiki/X87)）所用寄存器的别称；
2. **SSE**（[Streaming SIMD Extensions](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)）。SSE指令集于1999年发布。它定义了一些全新的寄存器集合，而不是重复使用已有的浮点数单元寄存器。从`xmm0`到`xmm15`，SSE定义了16个全新的寄存器，每个寄存器有128位长；
3. **AVX**（[Advanced Vector Extensions](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions)）。2008年发布的AVX又一次扩展了多媒体相关的寄存器，新的寄存器被称作`ymm0`到`ymm15`，长度均为256位。这些寄存器只是`xmm`寄存器的拓展，所以例如`xmm0`只是`ymm0`的低128位。

通过使用这些SIMD标准，程序通常能极大地提升速度。一些优秀的编译器拥有**自动矢量化编译技术**（[auto-vectorization](https://en.wikipedia.org/wiki/Automatic_vectorization)），能够自动将普通的循环代码转变为使用SIMD指令集的二进制码。

然而，庞大的SIMD寄存器可能在操作系统内核层面造成问题。当硬件中断发生时，操作系统内核不得不备份所有它使用的寄存器：因为在程序继续时，内核依然需要它原来的数据。所以如果内核使用了SIMD寄存器，它就不得不额外备份大量的数据，这将很显著地降低效率。为了避免效率损失，我们在开发内核时，常常禁用`sse`和`mmx`这两个**CPU特征**（CPU feature）。（`avx`特征是默认被禁用的。）

要禁用这两个特征，我们可以修改**目标配置清单**（target specification）的`features`配置项。使用`-`号，我们可以禁用`sse`和`mmx`两个CPU特征：

```json
"features": "-mmx,-sse"
```

## 浮点数

关于浮点数，我们有一个好消息和一个坏消息。坏消息是，`x86_64`架构也使用SSE寄存器做一些浮点数运算。因此，禁用SSE环境下的浮点数运算，都将导致LLVM发生错误。Rust的core库基于浮点数运算实现——比如它实现了f32和f64类型——这样的特点，让我们无法通过避免使用浮点数而绕开这个错误。而好消息是，LLVM支持一个称作`soft-float`的特征，它能够基于普通的整数运算软件模拟浮点运算；无需使用SSE寄存器，这让我们在内核中使用浮点数变为可能——只是性能上会慢一点点。

为了启用这个特征，我们把`+`号开头的特征名称加入配置项：

```json
"features": "-mmx,-sse,+soft-float"
```

