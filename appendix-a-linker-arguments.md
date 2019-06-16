>原文：https://os.phil-opp.com/freestanding-rust-binary/#linker-arguments
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust编写操作系统（附录一）：链接器参数

用Rust编写操作系统时，我们可能遇到一些链接器错误。这篇文章中，我们不将更换编译目标，而传送特定的链接器参数，尝试修复错误。我们将在常用系统Linux、Windows和macOS下，举例编写裸机应用时，可能出现的一些链接器错误；我们将逐个处理它们，还将讨论这种方式开发的必要性。

要注意的是，可执行程序在不同操作系统下格式各异；所以在不同平台下，参数和错误信息可能略有不同。

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

如果使用macOS系统开发，我们可能遇到这样的链接器错误：

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: entry point (_main) undefined. for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

这个错误消息告诉我们，链接器不能找到默认的入口点函数，它被命名为`main`——出于一些原因，macOS的所有函数名都被加以下划线`_`前缀。要设置入口点函数到`_start`，我们传送链接器参数`-e`：

```
cargo rustc -- -C link-args="-e __start"
```

`-e`参数指定了入口点的名称。由于每个macOS下的函数都有下划线`_`前缀，我们应该命名入口点函数为`__start`，而不是`_start`。

运行这行命令，现在出现了这样的链接器错误：

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: dynamic main executables must link with libSystem.dylib
          for architecture x86_64
          clang: error: linker command failed with exit code 1 […]
```

这个错误的原因是，macOS[并不官方支持静态链接的二进制库](https://developer.apple.com/library/content/qa/qa1118/_index.html)，而要求程序默认链接到`libSystem`库。要链接到静态二进制库，我们把`-static`标签传送到链接器：

```
cargo rustc -- -C link-args="-e __start -static"
```

运行修改后的命令。链接器似乎并不满意，又给我们抛出新的错误：

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" […]
  = note: ld: library not found for -lcrt0.o
          clang: error: linker command failed with exit code 1 […]
```

出现这个错误的原因是，macOS上的程序默认链接到`crt0`（C runtime zero）库。这和Linux系统上遇到的问题相似，我们可以添加一个`-nostartfiles`链接器参数：

```
cargo rustc -- -C link-args="-e __start -static -nostartfiles"
```

现在，我们的程序应该能够在macOS上成功编译了。

## 统一所有的编译命令

我们的裸机程序已经可以在多个平台上编译，但对每个平台，我们不得不记忆和使用不同的编译命令。为了避免这么做，我们创建`.cargo/config`文件，为每个平台填写对应的命令：

```toml
# in .cargo/config

[target.'cfg(target_os = "linux")']
rustflags = ["-C", "link-arg=-nostartfiles"]

[target.'cfg(target_os = "windows")']
rustflags = ["-C", "link-args=/ENTRY:_start /SUBSYSTEM:console"]

[target.'cfg(target_os = "macos")']
rustflags = ["-C", "link-args=-e __start -static -nostartfiles"]
```

这里，`rustflags`参数包含的内容，将被自动添加到每次`rustc`调用中。我们可以在[官方文档](https://doc.rust-lang.org/cargo/reference/config.html)中找到更多关于`.cargo/config`文件的说明。

做完这一步后，我们使用简单的一行指令——

```
cargo build
```

——就能在三个不同的平台上编译裸机程序了。

## 我们应该这么做吗？

虽然通过上文的方式，的确可以面向多个系统编译独立式可执行程序，但这可能不是一个好的途径。这么描述的原因是，我们的可执行程序仍然需要其它准备，比如在`_start`函数调用前一个加载完毕的栈。不使用C语言运行环境的前提下，这些准备可能并没有全部完成——这可能导致程序运行失败，比如说会抛出臭名昭著的段错误。

如果我们要为给定的操作系统创建最小的二进制程序，可以试着使用`libc`库并设定`#[start]`标记。[有一篇官方文档](https://doc.rust-lang.org/1.16.0/book/no-stdlib.html)给出了较好的建议。
