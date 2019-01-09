> 原文：https://os.phil-opp.com/vga-text-mode/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust创造操作系统（三）：VGA字符模式

**VGA字符模式**（[VGA text mode](https://en.wikipedia.org/wiki/VGA-compatible_text_mode)）是打印字符到屏幕的一种简单方式。在这篇文章中，为了包装这个模式为一个安全而简单的接口，我们包装unsafe代码到独立的模块。我们还将实现对Rust语言**格式化宏**（[formatting macros](https://doc.rust-lang.org/std/fmt/#related-macros)）的支持。

## VGA字符缓冲区

为了在VGA字符模式向屏幕打印字符，我们必须将它写入硬件提供的**VGA字符缓冲区**（VGA text buffer）。通常状况下，VGA字符缓冲区是一个25行、80列的二维数组，它的内容将被实时渲染到屏幕。这个数组的元素被称作**字符单元**（character cell），它使用下面的格式描述一个屏幕上的字符：

| Bit(s)    | Value |
|-----|----------------|
| 0-7   | ASCII code point |
| 8-11  | Foreground color |
| 12-14 | Background color |
| 15    | Blink |

其中，前景色和背景色取值范围如下：

| Number  | Color  | Number + Bright Bit  | Bright Color |
|-----|----------|------|--------|
| 0x0  | Black  | 0x8  | Dark Gray |
| 0x1  | Blue  | 0x9  | Light Blue |
| 0x2  | Green  | 0xa  | Light Green |
| 0x3  | Cyan  | 0xb  | Light Cyan |
| 0x4  | Red  | 0xc  | Light Red |
| 0x5  | Magenta  | 0xd  | Pink |
| 0x6  | Brown  | 0xe  | Yellow |
| 0x7  | Light Gray  | 0xf  | White |

每个颜色的第四位称为**加亮位**（bright bit）。

要修改VGA字符缓冲区，我们可以通过**存储器映射输入输出**（[memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O)）的方式，读取或写入地址`0xb8000`；这意味着，我们可以像操作普通的内存区域一样操作这个地址。

需要主页的是，一些硬件虽然映射到存储器，却可能不会完全支持所有的内存操作：可能会有一些设备支持按`u8`字节读取，却在读取`u64`时返回无效的数据。幸运的是，字符缓冲区都[支持标准的读写操作](https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vgamem.htm#manip)，所以我们不需要用特殊的标准对待它。

## 包装到Rust模块

既然我们已经知道VGA文字缓冲区如何工作，也是时候创建一个Rust模块来处理文字打印了。我们输入这样的代码：

```rust
// in src/main.rs
mod vga_buffer;
```

~~这行代码定义了一个Rust模块，它的内容可以保存在`src/vga_buffer.rs`或`src/vga_buffer/mod.rs`中；后一种保存方式支持使用**子模块**（submodule），而前者不支持。~~

这行代码定义了一个Rust模块，它的内容应当保存在`src/vga_buffer.rs`文件中。使用**2018版次**（2018 edition）的Rust时，我们可以把模块的**子模块**（submodule）文件直接保存到`src/vga_buffer/`文件夹下，与`vga_buffer.rs`文件共存，而无需创建一个`mod.rs`文件。

除非另有说明，

