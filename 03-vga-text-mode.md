> 原文：https://os.phil-opp.com/vga-text-mode/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust创造操作系统（三）：VGA字符模式

**VGA字符模式**（[VGA text mode](https://en.wikipedia.org/wiki/VGA-compatible_text_mode)）是打印字符到屏幕的一种简单方式。在这篇文章中，为了包装这个模式为一个安全而简单的接口，我们包装unsafe代码到独立的模块。我们还将实现对Rust语言**格式化宏**（[formatting macros](https://doc.rust-lang.org/std/fmt/#related-macros)）的支持。

## VGA字符缓冲区

为了在VGA字符模式向屏幕打印字符，我们必须将它写入硬件提供的**VGA字符缓冲区**（VGA text buffer）。通常状况下，VGA字符缓冲区是一个25行、80列的二维数组，它的内容将被实时渲染到屏幕。这个数组的元素被称作**字符单元**（character cell），它使用下面的格式描述一个屏幕上的字符：



其中，前景色和背景色取值范围如下：



每个颜色的第四位是**加亮位**（bright bit），比如会将蓝色变为亮蓝色。

