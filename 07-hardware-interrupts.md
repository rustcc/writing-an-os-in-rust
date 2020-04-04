> 原文：https://os.phil-opp.com/hardware-interrupts/
>
> 原作者：@phil-opp
>
> 译者：尚卓燃（@psiace） 华中农业大学

# 使用Rust编写操作系统（七）：硬件中断

在这一章中，我们将会设置可编程中断控制器（programmable interrupt controller），它的作用是正确地将硬件中断转发到 CPU。为了处理这些中断，需要向中断描述符表中添加新的条目，就像我们实现异常处理程序一样。通过对这一章的学习，你会了解到如何获取周期性定时器中断，以及如何获取键盘输入。

## 简介

中断为外部硬件设备提供了向 CPU 发送通知的方法。这样一来，内核不必定期检查键盘上是否产生了新字符（这一过程称作「[轮询]」），而是由键盘在出现按键事件时通知内核。采用这种方法有两个好处：一是中断处理更高效，因为内核只需要在硬件触发中断后进行响应；二是响应时间更短，因为内核可以即时作出响应，而不是在下一次轮询中进行处理。

[轮询]: https://en.wikipedia.org/wiki/Polling_(computer_science)

要将所有硬件设备都与 CPU 进行直接连接是不现实的。替代办法是使用一个单独的「中断控制器」（interrupt controller）来聚合所有中断控制器的中断，然后再通知 CPU：

```text
                                     ____________             _____
               定时器 ------------> |            |           |     |
               键盘 --------------> | 中断控制器 |---------> | CPU |
               其他硬件 ----------> |            |           |_____|
               更多... -----------> |____________|

```

大多数中断控制器都是可编程的，这意味着它们支持为中断分配不同的优先级。举个例子：如果需要保证计时准确，可以为定时器中断设置比键盘中断更高的优先级。

与异常不同的是，硬件中断是异步（asynchronously）发生的。这意味着它们完全独立于执行代码，并且可能在任何时候发生。因此，内核中需要一种并发机制来支撑，而且我们也需要面对所有与并发相关的潜在错误。Rust 严格的所有权模型会为我们提供一定帮助，因为它禁止使用可变的全局状态。然而，死锁仍然可能发生，我们在后面也会遇到这种情况。

## 8259 可编程中断控制器

[Intel 8259] 是一款在 1976 年推出的可编程中断控制器（PIC）。早已被新的「[高级可编程中断控制器（APIC）]」所取代，但由于 APIC 保持了较好的向后兼容，所以它的接口仍然在当前系统上得到较好的支持。8259 PIC 比 APIC 更容易设置，所以在我们切换到 APIC 之前，将先使用它来介绍我们自己的中断。

[高级可编程中断控制器（APIC）]: https://en.wikipedia.org/wiki/Intel_APIC_Architecture

8259 有 8 条中断控制线和几条与 CPU 通信的线。当时的典型系统配备了一主一从两个 8259 PIC 实例，其中从控制器连接到主控制器的一条中断控制线上。

[intel 8259]: https://en.wikipedia.org/wiki/Intel_8259

```text
                     ____________                          ____________
实时时钟 ---------> |            |   定时器 ------------> |            |
ACPI -------------> |            |   键盘 --------------> |            |      _____
可用 -------------> |            |----------------------> |            |     |     |
可用 -------------> | 从中断     |   串行端口 2 --------> | 主中断     |---> | CPU |
鼠标 -------------> |     控制器 |   串行端口 1 --------> |     控制器 |     |_____|
协处理器 ---------> |            |   并行端口 2/3 ------> |            |
主 ATA -----------> |            |   软盘 --------------> |            |
次 ATA -----------> |____________|   并行端口 1 --------> |____________|

```

上图显示了中断控制线的经典分配方案。我们看到剩下 15 条线中的大多数都对应有一个固定的映射，例如从中断控制器的第 4 条中断控制线被分配给了鼠标。

每个控制器可以通过两个 [I/O 端口] 进行配置，其中一个是「命令」端口，另一个是「数据」。 在主控制器中，这两个端口分别位于 0x20（命令）和 0x21（数据）。 而在从控制器中，分别是 0xa0（命令）和 0xa1（数据）。 如果你想要了解关于「如何配置中断控制器」的更多信息，可以参考 [osdev.org 上的文章]。

[I/O 端口]: ./04-testing.md#IO-端口
[osdev.org 上的文章]: https://wiki.osdev.org/8259_PIC

### 实现

不能使用默认的 PIC 配置，因为它将会发送 0-15 范围内的中断类型码到 CPU 。这些数字已经被 CPU 异常占用了，例如数字 8 对应二重错误。为了解决范围重叠的问题，我们需要将中断重新映射到不同的数字。实际的范围并不重要，只要不与异常重叠就可以，但通常会选择范围 32-47，因为这是 32 个异常槽之后的第一个空闲数字。

配置是通过向 PIC 的命令和数据端口写入特殊值来完成的。幸运的是，已经有一个名为 [`pic8259_simple`] 的包，所以我们不需要自己编写初始化序列。如果您对它的工作原理感兴趣，请查看[它的源代码][pic crate source]，它相当小并且有很好的文档说明。

[pic crate source]: https://docs.rs/crate/pic8259_simple/0.1.1/source/src/lib.rs

为了将包添加为依赖项，我们需要将以下内容添加到我们的项目中:

[`pic8259_simple`]: https://docs.rs/pic8259_simple/0.1.1/pic8259_simple/

```toml
# in Cargo.toml

[dependencies]
pic8259_simple = "0.1.1"
```

这个包提供的主要抽象是 [`ChainedPics`] 结构，它表示我们上面看到的主/从 PIC 布局。基于它的设计，我们可以按以下方式来使用它：

[`ChainedPics`]: https://docs.rs/pic8259_simple/0.1.1/pic8259_simple/struct.ChainedPics.html

```rust
// in src/interrupts.rs

use pic8259_simple::ChainedPics;
use spin;

pub const PIC_1_OFFSET: u8 = 32;
pub const PIC_2_OFFSET: u8 = PIC_1_OFFSET + 8;

pub static PICS: spin::Mutex<ChainedPics> =
    spin::Mutex::new(unsafe { ChainedPics::new(PIC_1_OFFSET, PIC_2_OFFSET) });
```

就像在前面提过的那样，我们将会为 PIC 设置偏移量，使得中断类型码范围为 32-47。 通过用 `Mutex` 包装 `ChainedPics` 结构，能够获得安全的可变访问（通过 [`lock` 方法][spin mutex lock]），这是我们在下一步所需要的。`ChainedPics::new` 函数不安全，因为错误的偏移量可能会导致未定义行为。

[spin mutex lock]: https://docs.rs/spin/0.5.2/spin/struct.Mutex.html#method.lock

现在我们可以在 `init` 函数中初始化 8259 PIC：

```rust
// in src/lib.rs

pub fn init() {
    gdt::init();
    interrupts::init_idt();
    unsafe { interrupts::PICS.lock().initialize() }; // new
}
```

我们使用 [`initialize`] 函数来执行 PIC 初始化操作。 像 `ChainedPics::new` 函数一样，这个函数也不安全，因为如果 PIC 配置错误，它可能导致未定义行为。

[`initialize`]: https://docs.rs/pic8259_simple/0.1.1/pic8259_simple/struct.ChainedPics.html#method.initialize

如果一切顺利，我们应该在执行 `cargo xrun` 时继续看到「It did not crash」这条消息。

## 启用中断

到目前为止还没有发生任何事情，因为 CPU 配置中禁用了中断。 这意味着 CPU 根本没有侦听中断控制器，因此没有中断可以到达 CPU。让我们改变这一点:

```rust
// in src/lib.rs

pub fn init() {
    gdt::init();
    interrupts::init_idt();
    unsafe { interrupts::PICS.lock().initialize() };
    x86_64::instructions::interrupts::enable();     // new
}
```

`x86_64` 包的 `interrupts::enable` 函数执行特殊的 `sti` 指令（设置中断「set interrupts」）以启用外部中断。现在尝试 cargo xrun，我们将会看到发生了一个双重错误：

![QEMU printing `EXCEPTION: DOUBLE FAULT` because of hardware timer](https://os.phil-opp.com/hardware-interrupts/qemu-hardware-timer-double-fault.png)

出现这种双重错误的原因是硬件定时器（确切地说是 [Intel 8253]）被设置为默认启用，一旦启用中断，我们就会开始接收定时器中断。 由于我们还没有为它定义中断处理程序，因此调用了双错处理程序。

[intel 8253]: https://en.wikipedia.org/wiki/Intel_8253

## 处理定时器中断

如 [上图](#8259-可编程中断控制器) 所示，定时器使用了主 PIC 的第 0 条中断控制线。 这意味着它以中断类型码 32（ 0 + 偏移量 32 ）的形式到达 CPU。 我们不使用硬编码索引 32，而是将它存储在 `InterruptIndex` 的枚举结构（enum）中:

```rust
// in src/interrupts.rs

#[derive(Debug, Clone, Copy)]
#[repr(u8)]
pub enum InterruptIndex {
    Timer = PIC_1_OFFSET,
}

impl InterruptIndex {
    fn as_u8(self) -> u8 {
        self as u8
    }

    fn as_usize(self) -> usize {
        usize::from(self.as_u8())
    }
}
```

Rust 中的枚举是 [c-like 风格的枚举]，因此我们可以直接为其内的每个变量指定索引。 `repr(u8)` 属性指定每个变量都以 `u8` 类型表示。 接下来，我们将会为其他中断添加更多的变量。

[c-like 风格的枚举]: https://doc.rust-lang.org/reference/items/enumerations.html#custom-discriminant-values-for-field-less-enumerations

现在我们可以为定时器中断添加一个处理函数：

```rust
// in src/interrupts.rs

use crate::print;

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        […]
        idt[InterruptIndex::Timer.as_usize()]
            .set_handler_fn(timer_interrupt_handler); // new

        idt
    };
}

extern "x86-interrupt" fn timer_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    print!(".");
}
```

定时器中断处理程序 `timer_interrupt_handler` 具有与异常处理程序相同的函数签名，因为 CPU 对异常和外部中断的反应是相同的（唯一的区别是有些异常会返回错误代码）。 [`InterruptDescriptorTable`] 结构实现了 [`IndexMut`] 特质（trait），因此我们可以通过数组索引语法访问单个条目。

[`interruptdescriptortable`]: https://docs.rs/x86_64/0.8.1/x86_64/structures/idt/struct.InterruptDescriptorTable.html
[`indexmut`]: https://doc.rust-lang.org/core/ops/trait.IndexMut.html

定时器定时器中断处理程序将会在屏幕上输出一个点。由于定时器中断周期性地发生，我们期望看到每当定时器「滴答」一下就输出一个点。但是，当我们运行程序时，屏幕上只输出了一个点:

![QEMU printing only a single dot for hardware timer](https://os.phil-opp.com/hardware-interrupts/qemu-single-dot-printed.png)

### 中断结束

之所以出现上面的故障，是因为 PIC 期望从错误处理程序得到一个明确的「中断结束」（end of interrupt, EOI）信号。 这个信号告诉控制器：中断已经被处理，这样系统就会准备好接收下一个中断。 所以 PIC 认为系统仍然忙于处理第一个定时器中断，并在发送下一个点之前耐心地等待 EOI 信号。

为了发送 EOI ，我们再次使用静态 `PICS` 结构:

```rust
// in src/interrupts.rs

extern "x86-interrupt" fn timer_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    print!(".");

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Timer.as_u8());
    }
}
```

通知中断处理函数 `notify_end_of_interrupt` 将会指出主控制器还是从控制器是否发送中断，然后使用 `命令` 和 `数据` 端口向各控制器发送 EOI 信号。 如果从 PIC 发送了中断，那么需要通知两个 PIC ，因为从 PIC 与主 PIC 的一条输入线相连。

我们需要谨慎地使用正确的中断类型码，否则可能会意外地删除一个重要的未发送中断或导致我们的系统挂起。这也是该函数不安全的原因。

现在执行 `cargo xrun` ，我们会看到一些点周期性地出现在屏幕上:

![QEMU printing consecutive dots showing the hardware timer](https://os.phil-opp.com/hardware-interrupts/qemu-hardware-timer-dots.gif)

### 配置定时器

我们使用的硬件定时器是可编程间隔定时器（Progammable Interval Timer, PIT）。 正如其名称所说，可以配置两个中断之间的间隔。我们不会详细介绍这些，因为我们很快就会切换到 [APIC 定时器]，但是 OSDev wiki 上有一篇关于「[如何配置 PIT ]」的详细文章。

[APIC 定时器]: https://wiki.osdev.org/APIC_timer
[如何配置 PIT ]: https://wiki.osdev.org/Programmable_Interval_Timer

## 死锁

现在内核中存在一种并发的情形：定时器中断是异步发生的，因此它们可以在任何时候中断我们的 `_start` 函数。 幸运的是 Rust 的所有权系统在编译期防止了许多种与并发相关的错误。 一个值得注意的例外是死锁。 如果一个线程试图获取一个永远不会释放的锁，就会发生死锁。 这样，线程将会无限期地处于挂起状态。

当前我们的内核中已经可以引发死锁。请记住，我们的 `println` 宏调用 `vga_buffer::_print` 函数，它使用自旋锁来[锁定一个全局的 WRITER 类][vga spinlock]：

[vga spinlock]: ./03-vga-text-mode.md#自旋锁

```rust
// in src/vga_buffer.rs

[…]

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```

它锁定 `WRITER`，调用 `write_fmt`，并在函数的末尾隐式地将其解锁。现在我们设想一下，如果在 `WRITER` 被锁定时，触发一个中断，同时相应的中断处理程序也试图打印一些东西：

| Timestep | \_start                | interrupt_handler                               |
| -------- | ---------------------- | ----------------------------------------------- |
| 0        | calls `println!`       | &nbsp;                                          |
| 1        | `print` locks `WRITER` | &nbsp;                                          |
| 2        |                        | **interrupt occurs**, handler begins to run     |
| 3        |                        | calls `println!`                                |
| 4        |                        | `print` tries to lock `WRITER` (already locked) |
| 5        |                        | `print` tries to lock `WRITER` (already locked) |
| …        |                        | …                                               |
| _never_  | _unlock `WRITER`_      |                                                 |

`WRITER` 是锁定的，所以中断处理程序将会一直等待，直到它可用。但这种情况不会发生，因为 `_start` 函数只有在中断处理程序返回后才继续运行。 这样整个系统就挂起来了。

### 引发死锁

通过在 `_start` 函数末尾的循环中打印一些内容，我们很容易在内核中引发这样的死锁:

```rust
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() -> ! {
    […]
    loop {
        use blog_os::print;
        print!("-");        // new
    }
}
```

当我们在 QEMU 中运行它时，我们得到的输出如下:

![QEMU output with many rows of hyphens and no dots](https://os.phil-opp.com/hardware-interrupts/qemu-deadlock.png)

只有有限数量的连字符被打印，直到第一个定时器中断发生。接着系统挂起，因为定时器中断处理程序试图打印一个点时，引发了死锁。这就是为什么我们在上面的输出中看不到点的原因。

由于定时器中断是异步发生的，因此连字符的实际数量在运行时会发生变化。 这种不确定性使得与并发相关的错误很难调试。

### 修复死锁

为了避免这种死锁，我们可以采取这样的方案：只要互斥锁 `Mutex` 是锁定的，就禁用中断。

```rust
// in src/vga_buffer.rs

/// Prints the given formatted string to the VGA text buffer
/// through the global `WRITER` instance.
#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    use x86_64::instructions::interrupts;   // new

    interrupts::without_interrupts(|| {     // new
        WRITER.lock().write_fmt(args).unwrap();
    });
}
```

[`without_interrupts`] 函数接受一个 [闭包（closure）]，并在一个无中断的环境中执行它。我们使用它来确保只要 `Mutex` 处于锁定状态，就不会发生中断。现在运行内核，就可以看到它一直运行而不会挂起。（我们仍然无法看到任何点，但这是因为他们滚动过快。尝试减慢打印速度，例如在循环中加上 `for _ in 0..10000 {}`）

[`without_interrupts`]: https://docs.rs/x86_64/0.8.1/x86_64/instructions/interrupts/fn.without_interrupts.html
[闭包（closure）]: https://doc.rust-lang.org/book/second-edition/ch13-01-closures.html

我们可以对串行打印函数进行相同的更改，以确保它不会发生死锁:

```rust
// in src/serial.rs

#[doc(hidden)]
pub fn _print(args: ::core::fmt::Arguments) {
    use core::fmt::Write;
    use x86_64::instructions::interrupts;       // new

    interrupts::without_interrupts(|| {         // new
        SERIAL1
            .lock()
            .write_fmt(args)
            .expect("Printing to serial failed");
    });
}
```

请注意，禁用中断不应该成为一种通用的解决方案。这一方案将会延长最坏情况下的中断延迟，也就是系统对中断做出反应的时间。 因此，应该只在非常短的时间内禁用中断。

## 修复竞争条件

如果你运行 `cargo xtest`，可能会看到 `test_println_output` 测试失败:

```shell
> cargo xtest --lib
[…]
Running 4 tests
test_breakpoint_exception...[ok]
test_println... [ok]
test_println_many... [ok]
test_println_output... [failed]

Error: panicked at 'assertion failed: `(left == right)`
  left: `'.'`,
 right: `'S'`', src/vga_buffer.rs:205:9
```

这是由测试和定时器处理程序之间的竞争条件导致的。测试程序是这样的：

```rust
// in src/vga_buffer.rs

#[test_case]
fn test_println_output() {
    serial_print!("test_println_output... ");

    let s = "Some test string that fits on a single line";
    println!("{}", s);
    for (i, c) in s.chars().enumerate() {
        let screen_char = WRITER.lock().buffer.chars[BUFFER_HEIGHT - 2][i].read();
        assert_eq!(char::from(screen_char.ascii_character), c);
    }

    serial_println!("[ok]");
}
```

测试将一个字符串打印到 VGA 缓冲区，然后通过在缓冲区字符数组 `buffer_chars` 上手动迭代来检查输出。 出现竞争条件是因为定时器中断处理程序可能在 `println` 和读取屏幕字符之间的空隙运行。 请注意，这不是一个危险的 **数据竞争**，Rust 在编译时完全防止了这种竞争。 查看 [_Rustonomicon_][nomicon-races] 获得详细信息。

[nomicon-races]: https://doc.rust-lang.org/nomicon/races.html

为了解决这个问题，我们需要在测试的整个持续时间内保持 `WRITER` 的锁定状态，这样定时器处理程序就不能在空隙期间将 `.` 输出到屏幕上。修复后的测试看起来像这样：

```rust
// in src/vga_buffer.rs

#[test_case]
fn test_println_output() {
    use core::fmt::Write;
    use x86_64::instructions::interrupts;

    serial_print!("test_println_output... ");

    let s = "Some test string that fits on a single line";
    interrupts::without_interrupts(|| {
        let mut writer = WRITER.lock();
        writeln!(writer, "\n{}", s).expect("writeln failed");
        for (i, c) in s.chars().enumerate() {
            let screen_char = writer.buffer.chars[BUFFER_HEIGHT - 2][i].read();
            assert_eq!(char::from(screen_char.ascii_character), c);
        }
    });

    serial_println!("[ok]");
}
```

我们做了下述改动：

- 显式地使用 `lock()` 方法来保持 `writer` 处于锁定状态，以完成测试。使用允许打印到已锁定的 `writer` 中的 [`writeln`] 宏替代 `println`。
- 为了避免另一个死锁，我们在测试期间禁用中断。 否则，在 `writer` 仍然处于锁定状态时，测试可能会中断。
- 由于计时器中断处理程序仍然可以在测试之前运行，因此我们在打印字符串 `s` 之前再打印一个新行 `\n` 。这样可以避免因计时器处理程序已经将一些 `.` 字符打印到当前行而引起的测试失败。

[`writeln`]: https://doc.rust-lang.org/core/macro.writeln.html

经过修改后，`cargo xtest` 现在确实又成功了。

这是一个相对无害的竞争条件，它只会导致测试失败。可以想象，由于其他竞争条件的非确定性，它们的调试可能更加困难。幸运的是，Rust 防止了数据竞争的出现，这是最严重的竞争条件，因为它们可以导致各种各样的未定义行为，包括系统崩溃和无声的内存损坏。

## `hlt` 指令

到目前为止，我们在 `_start` 和 `panic` 函数的末尾使用了一个简单的空循环语句。 这将导致 CPU 无休止地自旋，从而按预期工作。 但是这也是非常低效的，因为即使在没有任何工作要做的情况下，CPU 仍然会全速运行。 在运行内核时，您可以在任务管理器中看到这个问题: QEMU 进程在整个过程中都需要接近100% 的 CPU。

我们真正想做的是让 CPU 停下来直到触发下一个中断。这使得 CPU 进入一个休眠状态，在这种状态下它消耗的能量要少得多。[`hlt` 指令] 应运而生。让我们使用它来创建一个节能的无限循环：

[`hlt` 指令]: https://en.wikipedia.org/wiki/HLT_(x86_instruction)

```rust
// in src/lib.rs

pub fn hlt_loop() -> ! {
    loop {
        x86_64::instructions::hlt();
    }
}
```

`instructions::hlt` 函数只是汇编指令的一个 [瘦包装器]。 它是安全的，因为它不可能危及内存安全。

[瘦包装器]: https://github.com/rust-osdev/x86_64/blob/5e8e218381c5205f5777cb50da3ecac5d7e3b1ab/src/instructions/mod.rs#L16-L22

我们现在可以使用这个 `hlt_loop` 循环来代替 `_start` 和 `panic` 函数中的无限循环：

```rust
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() -> ! {
    […]

    println!("It did not crash!");
    blog_os::hlt_loop();            // new
}


#[cfg(not(test))]
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    blog_os::hlt_loop();            // new
}

```

让我们也更新一下 `lib.rs`：

```rust
// in src/lib.rs

/// Entry point for `cargo xtest`
#[cfg(test)]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    init();
    test_main();
    hlt_loop();         // new
}

pub fn test_panic_handler(info: &PanicInfo) -> ! {
    serial_println!("[failed]\n");
    serial_println!("Error: {}\n", info);
    exit_qemu(QemuExitCode::Failed);
    hlt_loop();         // new
}
```

现在在 QEMU 中运行内核时，我们会看到比原来低得多的 CPU 使用率。

## 键盘输入

现在已经能够处理来自外部设备的中断，最终我们需要添加对键盘输入的支持。 这将是我们与内核进行的第一次交互。

注意，这里只描述如何处理 [PS/2] 键盘，而不包括 USB 键盘。然而，主板会模拟 USB 键盘作为 PS/2 设备来支持旧的软件，所以可以安全地忽略 USB 键盘，直到我们的内核中有 USB 支持。

[PS/2]: https://en.wikipedia.org/wiki/PS/2_port

与硬件定时器一样，键盘控制器也被设置为默认启用。因此，当你按下一个键时，键盘控制器会向 PIC 发送一个中断，然后由 PIC 将中断转发给 CPU。 CPU 在 IDT 中查找处理程序函数，但是相应的条目是空的。 因此发生了双重错误。

因此，让我们为键盘中断添加一个处理程序函数。 它和我们定义的定时器中断处理程序非常相似，只是使用了一个不同的中断类型码：

```rust
// in src/interrupts.rs

#[derive(Debug, Clone, Copy)]
#[repr(u8)]
pub enum InterruptIndex {
    Timer = PIC_1_OFFSET,
    Keyboard, // new
}

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        […]
        // new
        idt[InterruptIndex::Keyboard.as_usize()]
            .set_handler_fn(keyboard_interrupt_handler);

        idt
    };
}

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    print!("k");

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

正如我们从 [上面](#8259-可编程中断控制器) 的图片中看到的那样，键盘使用了主 PIC 的第 1 条中断控制线。这意味着它以中断类型码 33（ 1 + 偏移量 32 ）的形式到达 CPU 。我们将这个索引作为新的 `Keyboard` 变量添加到 `InterruptIndex` 枚举中。 我们不需要明确指定这个值，因为它默认为前一个值加 1，也就是 33 。 在中断处理程序中，我们打印一个 `k` 并将中断信号发送给中断控制器。

现在看到，当我们按下一个键时，屏幕上会出现一个 `k` 。 然而，这只适用于按下的第一个键，即使我们继续按键，也不会有更多的 `k` 出现在屏幕上。 这是因为键盘控制器不会发送另一个中断，直到我们读取所谓的「键盘扫描码（scancode）」。

### 读取键盘扫描码

要找出按了 **哪个** 键，需要查询键盘控制器。 我们可以通过读取 PS/2 控制器的数据端口来实现这一点，该端口属于 [I/O 端口] ，编号为 `0x60` ：

[I/O 端口]: @/second-edition/posts/04-testing/index.md#i-o-ports

```rust
// in src/interrupts.rs

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    use x86_64::instructions::port::Port;

    let mut port = Port::new(0x60);
    let scancode: u8 = unsafe { port.read() };
    print!("{}", scancode);

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

我们使用 `x86_64` 包提供的端口类型 [`Port`] 从键盘的数据端口读取一个字节。 这个字节称为 「[**键盘扫描码**]」 ，是一个表示物理键按下/松开的数字。 目前还没有对键盘扫描码进行处理，只是把它打印到屏幕上：

[`Port`]: https://docs.rs/x86_64/0.8.1/x86_64/instructions/port/struct.Port.html
[**键盘扫描码**]: https://en.wikipedia.org/wiki/Scancode

![QEMU printing scancodes to the screen when keys are pressed](https://os.phil-opp.com/hardware-interrupts/qemu-printing-scancodes.gif)

上面的图片显示我正在慢慢地输入「123」。 我们可以看到，相邻键具有相邻的键盘扫描码，按下 / 释放一个物理键会触发一个不同的键盘扫描码。但是我们如何将键盘扫描码转换为实际的按键操作呢？

### 解释键盘扫描码

键盘扫描码和物理键之间的映射有三种不同的标准，通常被称作键盘扫描码集。这三者都可以追溯到早期 IBM 计算机的键盘: [IBM XT]、 [IBM 3270 PC] 和 [IBM AT] 。 幸运地是，后来的计算机没有继续定义新的键盘扫描码集的趋势，而是模仿现有的集合并扩展它们。今天，大多数键盘都可以配置为模拟这三套键盘中的任何一套。

[IBM XT]: https://en.wikipedia.org/wiki/IBM_Personal_Computer_XT
[IBM 3270 PC]: https://en.wikipedia.org/wiki/IBM_3270_PC
[IBM AT]: https://en.wikipedia.org/wiki/IBM_Personal_Computer/AT

默认情况下，PS/2 键盘模拟键盘扫描码集 1（「XT」）。 在这个集合中，键盘扫描码的较低的 7 位字节定义了物理键信息，而最高有效位定义了它是按下（「0」）还是释放（「1」）。原始的「IBM XT」键盘上没有的键，如键盘上的 `enter` 键，会连续生成两个键盘扫描码: 一个 `0xe0` 转义字节和一个表示物理键信息的字节。 有关集合 1 的所有键盘扫描码及其对应物理键的列表，请访问 [OSDev Wiki][scancode set 1]。

[scancode set 1]: https://wiki.osdev.org/Keyboard#Scan_Code_Set_1

要想将键盘扫描码转换为物理键，可以使用一个 `match` 语句:

```rust
// in src/interrupts.rs

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    use x86_64::instructions::port::Port;

    let mut port = Port::new(0x60);
    let scancode: u8 = unsafe { port.read() };

    // new
    let key = match scancode {
        0x02 => Some('1'),
        0x03 => Some('2'),
        0x04 => Some('3'),
        0x05 => Some('4'),
        0x06 => Some('5'),
        0x07 => Some('6'),
        0x08 => Some('7'),
        0x09 => Some('8'),
        0x0a => Some('9'),
        0x0b => Some('0'),
        _ => None,
    };
    if let Some(key) = key {
        print!("{}", key);
    }

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

上面的代码转换数字键 0-9 的按键操作，并忽略所有其他键。 它使用 [match] 语句为每个键盘扫描码分配一个相应的字符或 `None`。 然后它使用 [`if let`] 来解构可选的 `key`。 通过在模式中使用相同的变量名 `key`，我们 [隐藏] 了前面的声明，这是 Rust 中解构 `Option` 类型的常见模式。

[match]: https://doc.rust-lang.org/book/ch06-02-match.html
[`if let`]: https://doc.rust-lang.org/book/ch18-01-all-the-places-for-patterns.html#conditional-if-let-expressions
[隐藏]: https://doc.rust-lang.org/book/ch03-01-variables-and-mutability.html#shadowing

现在我们可以输出数字了:

![QEMU printing numbers to the screen](https://os.phil-opp.com/hardware-interrupts/qemu-printing-numbers.gif)

我们也可以用同样的方式转换其他键。 幸运的是，有一个名为 [`pc-keyboard`] 的包，专门用于翻译键盘扫描码集 1 和 2 的键盘扫描码，因此我们无须自己实现。要使用这个包，需要将它添加到 `Cargo.toml` 内，并导入到 `lib.rs` 中:

[`pc-keyboard`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/

```toml
# in Cargo.toml

[dependencies]
pc-keyboard = "0.5.0"
```

现在我们可以使用这个包来重写键盘中断处理程序 `keyboard_interrupt_handler`：

```rust
// in/src/interrupts.rs

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame)
{
    use pc_keyboard::{layouts, DecodedKey, HandleControl, Keyboard, ScancodeSet1};
    use spin::Mutex;
    use x86_64::instructions::port::Port;

    lazy_static! {
        static ref KEYBOARD: Mutex<Keyboard<layouts::Us104Key, ScancodeSet1>> =
            Mutex::new(Keyboard::new(layouts::Us104Key, ScancodeSet1,
                HandleControl::Ignore)
            );
    }

    let mut keyboard = KEYBOARD.lock();
    let mut port = Port::new(0x60);

    let scancode: u8 = unsafe { port.read() };
    if let Ok(Some(key_event)) = keyboard.add_byte(scancode) {
        if let Some(key) = keyboard.process_keyevent(key_event) {
            match key {
                DecodedKey::Unicode(character) => print!("{}", character),
                DecodedKey::RawKey(key) => print!("{:?}", key),
            }
        }
    }

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

我们使用 `lazy_static` 宏来创建一个由互斥锁保护的静态键盘对象 [`Keyboard`]。 我们使用美国键盘布局初始化键盘，并采用键盘扫描码集 1 。 [`HandleControl`] 参数允许将 `ctrl+[a-z]` 映射到 Unicode 字符 `U+0001` 到 `U+001A` 。 我们不想这样做，所以使用 `Ignore` 选项来像处理普通键一样处理 `ctrl` 键。

[`Handlecontrol`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/enum.HandleControl.html

每当中断发生，我们锁定互斥对象，从键盘控制器读取键盘扫描码并将其传递给 [`add_byte`] 方法，后者将键盘扫描码转换为可选的按键事件 `Option<KeyEvent>` 。 [`KeyEvent`] 包含引发事件的物理键以及它的事件类型——按下或是松开。

[`Keyboard`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/struct.Keyboard.html
[`add_byte`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/struct.Keyboard.html#method.add_byte
[`keyEvent`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/struct.KeyEvent.html

为了解释按键事件，我们将其传递给按键事件处理方法 [`process_keyevent`]，该方法将按键事件转换为字符。例如，根据是否按下 `shift` 键，将物理键 `a` 的按下事件转换为对应的小写字符或大写字符。

[`process_keyevent`]: https://docs.rs/pc-keyboard/0.5.0/pc_keyboard/struct.Keyboard.html#method.process_keyevent

有了这个修改过的中断处理程序，我们现在可以输出文本：

![Typing "Hello World" in QEMU](https://os.phil-opp.com/hardware-interrupts/qemu-typing.gif)

### 配置键盘

配置 PS/2 键盘的某些方面是可行的，例如应该使用哪个键盘扫描码集。但我们不会在这里讨论它，因为这篇文章已经足够长了，但是 OSDev Wiki 上有一个关于可能的 [配置命令] 的概述。

[配置命令]: https://wiki.osdev.org/PS/2_Keyboard#Commands

## 小结

这篇文章解释了如何启用和处理外部中断。 我们学习了 8259 PIC 和它的主/从二级布局，中断类型码的重新映射，以及「中断结束」信号。 我们实现了硬件定时器和键盘的处理程序，并且学习了 `hlt` 指令，它会停止 CPU 直到触发下一个中断。

现在我们可以与内核进行交互，并且有一些基本的构建块来创建一个小 shell 或简单的游戏。

## 下篇预告

定时器中断对于操作系统是必不可少的，因为它们提供了一种周期性地中断运行进程并在内核中恢复控制的方法。这样一来，内核就可以切换到不同的进程，并营造出一种多个进程并行运行的错觉。

但是在创建进程或线程之前，我们需要一种为它们分配内存的方法。 下一篇文章将探讨内存管理，以提供这一基本构建块。
