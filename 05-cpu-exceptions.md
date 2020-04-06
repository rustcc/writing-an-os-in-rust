> 原文：https://os.phil-opp.com/cpu-exceptions/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust编写操作系统（五）：CPU异常

当出错情况发生，CPU将抛出异常，例如除以零或者访问非法地址。为了处理这些异常，我们应当设置**中断描述表**（interrupt descriptor table），来提供处理函数。这篇文章中，我们尝试抓取**断点异常**（[breakpoint exceptions]），处理这个异常并返回正常运行流程。

[breakpoint exceptions]: https://wiki.osdev.org/Exceptions#Breakpoint

## 什么是异常？

执行当前指令时可能出现的错误状况称作**异常**（exception）。例如，如果当前指令尝试除以零，CPU将抛出除零异常。当异常发生时，CPU打断当前的工作，并由异常的类型，转而立即调用对应的**异常处理函数**（exception handler function）。

x86平台提供了大约20种不同的CPU异常。我们列举最重要的几种：

- **缺页异常**（page fault）是对分页内存的非法访问产生的。举个例子，如果当前的指令试着读取没有映射的内存页，或者尝试写只读的内存页，就会发生这个异常。
- **非法操作码**（invalid opcode）发生在运行非法指令的时候。如果我们有老旧的CPU，却要执行新的[SSE指令]，这个异常就会发生。
- **通用保护错误**（general protection fault）由多种错误来源产生。比如尝试写入配置寄存器的保留位，或者在用户模式执行特权指令。
- **双重异常**（double fault）是处理异常时发生的异常。当异常发生时，CPU将调用对应的异常处理器函数。如果在这个函数里发生了异常，将成为一个双重异常。另外，如果没有为某个异常注册处理器函数，但这个异常发生了，也会产生双重异常。
- **三重异常**（triple fault）是处理双重异常时发生的异常。这个异常对CPU是致命的，我们不能捕获或处理三重异常。当三重异常发生时，大多数的寄存器会直接复位并重启操作系统。

[SSE指令]: https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions

OSDev Wiki的[这篇文章](https://wiki.osdev.org/Exceptions)列出了完整的x86异常列表。

### 中断描述表

为了捕获和处理异常，我们需要设置中断描述表。我们能通过这个表，为每个CPU异常提供处理函数。CPU硬件将直接使用这个表，所以我们需要遵守CPU定义的格式。它的每个条目都由以下参数组成：

| 类型 | 名称 | 说明 |
|:----|:-----|:-----|
| u16 | 偏移地址 [0:15] | 处理器函数偏移地址的低16位 |
| u16 | GDT选择器 | 参见[全局描述符表] |
| u16 | 选项参数 | 定义见下文 |
| u16 | 偏移地址 [16:31] | 处理器函数偏移地址的中间位 |
| u32 | 偏移地址 [32:63] | 处理器函数偏移地址的高位 |
| u32 | _保留位_ | |

[全局描述符表]: https://en.wikipedia.org/wiki/Global_Descriptor_Table

选项参数的所有位定义如下：

| 位编号 | 名称 | 说明 |
|:-----|:-----|:-----|
| 0-2 | 中断栈表索引 | 0: 不要切换栈。1-7: 切换到中断栈表的第n个栈 |
| 3-7 | _保留位_ | |
| 8 | 中断开关 | 如果是0，当处理器函数被调用时，中断将关闭 |
| 9-11 | _必须为1_ | |
| 12 | _必须为0_ | |
| 13-14 | 特权等级识别符 | 调用这个处理器函数所需最低的特权等级 |
| 15 | 使能位 | 设置为0，表示中断没有处理器函数 |

每个异常在描述表里都有固定的位置。例如，非法操作码的索引位置是6，缺页异常的位置是14。这样，硬件就可以自动加载对应的描述表条目。

当异常发生时，CPU大致会做下面的事情：

1. 把一些寄存器压栈。这包指令指针寄存器IP和标签寄存器RFLAGS；
2. 从中断描述表读取异常对应的条目。如果发生了缺页异常，CPU将读取第14个条目；
3. 检查它的使能位。如果为0，将发生双重异常；
4. 如果条目的中断开关为0，关闭硬件中断；
5. 从全局描述符表选择并读取内容，放入代码段寄存器CS中；
6. 跳转到对应的异常处理函数。

我们将在后面的文章探索全局描述符表的定义和内容。

## 中断描述表类型

知道中断描述表的定义后，我们可以选择自己编写对应的代码。幸运的是，`x86_64`包已经为我们提供了[中断描述表的实现]，它的代码如下：

[中断描述表的实现]: https://docs.rs/x86_64/0.9.6/x86_64/structures/idt/struct.InterruptDescriptorTable.html

```rust
#[repr(C)]
pub struct InterruptDescriptorTable {
    pub divide_by_zero: Entry<HandlerFunc>,
    pub debug: Entry<HandlerFunc>,
    pub non_maskable_interrupt: Entry<HandlerFunc>,
    pub breakpoint: Entry<HandlerFunc>,
    pub overflow: Entry<HandlerFunc>,
    pub bound_range_exceeded: Entry<HandlerFunc>,
    pub invalid_opcode: Entry<HandlerFunc>,
    pub device_not_available: Entry<HandlerFunc>,
    pub double_fault: Entry<HandlerFuncWithErrCode>,
    pub invalid_tss: Entry<HandlerFuncWithErrCode>,
    pub segment_not_present: Entry<HandlerFuncWithErrCode>,
    pub stack_segment_fault: Entry<HandlerFuncWithErrCode>,
    pub general_protection_fault: Entry<HandlerFuncWithErrCode>,
    pub page_fault: Entry<PageFaultHandlerFunc>,
    pub x87_floating_point: Entry<HandlerFunc>,
    pub alignment_check: Entry<HandlerFuncWithErrCode>,
    pub machine_check: Entry<HandlerFunc>,
    pub simd_floating_point: Entry<HandlerFunc>,
    pub virtualization: Entry<HandlerFunc>,
    pub security_exception: Entry<HandlerFuncWithErrCode>,
    // 部分成员省略
}
```

结构体的每个成员都是`idt::Entry<F>`类型的，它表示一个中断描述表的条目。类型参数`F`，表示其中异常处理函数的类型。我们可以看到，异常处理函数需要`HandlerFunc`、`HandlerFuncWithErrCode`或者特殊的`PageFaultHandlerFunc`类型。

我们先看看`HandlerFunc`类型的定义：

```Rust
type HandlerFunc = extern "x86-interrupt" fn(_: &mut InterruptStackFrame);
```

这是一个**类型别名**（[type alias]）定义，它将定义一个`extern "x86-interrupt" fn`的别名。`extern`关键字说明，函数将遵守一定的**外部调用约定**（[foreign calling convention]）；通常我们和C语言代码通讯时，也会以`extern "C"`的形式用到它。我们需要了解什么是`x86-interrupt`调用约定。

[type alias]: https://doc.rust-lang.org/book/ch19-04-advanced-types.html#creating-type-synonyms-with-type-aliases
[foreign calling convention]: https://doc.rust-lang.org/nomicon/ffi.html#foreign-calling-conventions

## x86中断调用约定

处理异常和调用函数比较相似：CPU将找到对应的指令，跳转并开始执行。处理完成后，CPU将跳转到返回地址，返回来继续执行函数的调用者。

需要注意的是，异常和普通函数有一些不同点。普通的函数调用只有执行`call`指令会发生，而异常可能会在任意指令处发生。为了理解这个不同点带来的影响，我们需要了解函数调用的步骤细节。

**调用约定**（[calling convention]）是决定函数调用细节的一个因素。比如说，它决定哪个寄存器存放函数的参数，是寄存器还是栈，函数的返回值又应该放在哪里。在x86_64平台的Linux系统，[System V ABI]提供了一套标准用于C语言函数：

[calling convention]: https://en.wikipedia.org/wiki/Calling_convention
[System V ABI]: https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf

- 前六个参数依次置于寄存器`rdi`、`rsi`、`rdx`、`rcx`、`r8`和`r9`；
- 其它参数应当放在栈上；
- 返回值需要放入`rax`和`rdx`。

当我们谈起Rust语言，需要注意的是Rust暂时还没有一个稳定的ABI标准。它通常有自己的调用约定；但如果使用`extern "C" fn`定义函数，它将遵守上面的C语言调用约定。
