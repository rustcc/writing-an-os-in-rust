>原文：https://os.phil-opp.com/freestanding-rust-binary/#linker-arguments
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust编写操作系统（附录一）：链接器参数

用Rust编写系统时，我们可能遇到特定的链接器错误。这篇文章中，我们将探讨常用系统Linux、Windows和macOS下的链接器错误，并传送链接器参数来解决它们。要注意的是，可执行程序在不同操作系统下格式各异，所以要求的参数可能略有不同。

## Linux

我们从典型的链接器错误开始：

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status
```

我们遇到的问题是，链接器将默认引用C语言运行时的启动流程，或者也被称为`_start`函数。它将使用我们在`no_std`下被排除的C语言标准库实现库`libc`，因此链接器不能解析相关的引用，得到“undefined reference”。为了解决这个问题，我们告诉链接器，它不应该引用C语言启动流程——我们可以添加`-nostartfiles`标签来做到这一点。

要通过cargo添加参数到链接器，我们使用`cargo rustc`指令。这个指令的用途和`cargo build`相同，但允许开发者向下层的Rust编译器`rustc`传递参数。另外，`rustc`有一个`-C link-arg`标签，它将传递参数到链接器。综上所述，我们编写下面的指令：

```
cargo rustc -- -C link-arg=-nostartfiles
```

这样之后，我们的包可以编译为一个Linux系统下的独立式可执行程序了。在这里我们不需要显式指定入口点函数的名称，因为链接器默认使用函数名`_start`。

## Windows

在Windows系统下，我们有了不一样的链接器错误：

```
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

链接器错误提示“必须定义入口点”，意味着它没有找到入口点。在Windows系统下，默认的入口点名字[由使用的子系统决定](https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol)。对`CONSOLE`子系统，链接器将寻找名为`mainCRTStartup`的函数；而对`WINDOWS`子系统，它将寻找`WinMainCRTStartup`。为了使用我们的`_start`函数，我们向链接器传递`/ENTRY`参数：

```
cargo rustc -- -C link-arg=/ENTRY:_start
```

这里我们能从参数的格式中看到，Windows系统下的链接器与Linux下的有较大不同。

运行命令，我们得到不同的链接器错误：

```
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

产生这个错误，是因为Windows可执行程序可以使用不同的**子系统**（[subsystem](https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol)）。

todo