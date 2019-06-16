>原文：https://os.phil-opp.com/freestanding-rust-binary/#linker-arguments
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust编写操作系统（附录一）：链接器参数

用Rust编写系统时，我们可能遇到特定的链接器错误。这篇文章中，我们将探讨常用系统Linux、Windows和macOS下的链接器错误，并传送链接器参数来解决它们。要注意的是，可执行程序在不同操作系统下格式各异，所以要求的参数可能略有不同。

## Linux

我们在Linux下尝试编写裸机程序，可能出现这样的链接器错误：

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

这里遇到的问题是，链接器将默认引用C语言运行时的启动流程，或者也被描述为`_start`函数。它将使用我们在`no_std`下被排除的C语言标准库实现库`libc`，因此链接器不能解析相关的引用，得到“undefined reference”问题。为解决这个问题，我们需要告诉链接器，它不应该引用C语言使用的启动流程——我们可以添加`-nostartfiles`标签来做到这一点。

要通过cargo添加参数到链接器，我们使用`cargo rustc`命令。这个命令的作用和`cargo build`相同，但允许开发者向下层的Rust编译器`rustc`传递参数。另外，`rustc`提供一个`-C link-arg`标签，它能够传递所需的参数到链接器。综上所述，我们可以编写下面的命令：

```
cargo rustc -- -C link-arg=-nostartfiles
```

这样之后，我们的包应该能成功编译，作为Linux系统下的独立式可执行程序了。这里我们没有显式指定入口点函数的名称，因为链接器将默认使用函数名`_start`。

## Windows

在Windows系统下，可能有不一样的链接器错误：

```
error: linking with `link.exe` failed: exit code: 1561
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1561: entry point must be defined
```

链接器错误提示“必须定义入口点”，意味着它没有找到入口点。在Windows系统下，默认的入口点函数名[由使用的子系统决定](https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol)。对`CONSOLE`子系统，链接器将寻找名为`mainCRTStartup`的函数；而对`WINDOWS`子系统，它将寻找`WinMainCRTStartup`。我们的`_start`函数并非这两个名称：为了使用它，我们可以向链接器传递`/ENTRY`参数：

```
cargo rustc -- -C link-arg=/ENTRY:_start
```

我们也能从这里的参数的格式中看到，Windows系统下的链接器在使用方法上，与Linux下的有较大不同。

运行命令，我们得到了另一个链接器错误：

```
error: linking with `link.exe` failed: exit code: 1221
  |
  = note: "C:\\Program Files (x86)\\…\\link.exe" […]
  = note: LINK : fatal error LNK1221: a subsystem can't be inferred and must be
          defined
```

产生这个错误，是由于Windows可执行程序可以使用不同的**子系统**（[subsystem](https://docs.microsoft.com/en-us/cpp/build/reference/entry-entry-point-symbol)）。对一般的Windows程序，使用的子系统将由入口点的函数名推断而来：如果入口点是`main`函数，将使用`CONSOLE`子系统；如果是`WinMain`函数，则使用`WINDOWS`子系统。由于我们的`_start`函数名称与上两者不同，我们需要显式指定使用的子系统：

```
cargo rustc -- -C link-args="/ENTRY:_start /SUBSYSTEM:console"
```

这里我们使用`CONSOLE`子系统，但`WINDOWS`子系统也是可行的。这里，我们使用复数参数`link-args`代替多个`-C link-arg`，因为后者要求把所有参数依次列出，比较占用空间。

使用这行命令后，我们的可执行程序应该能在Windows下运行了。

## macOS


