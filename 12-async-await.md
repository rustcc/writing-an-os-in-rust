---
title: Async/Await
date: 2019-09-29 09:45:40
tags: [Multitasking]
summary: 在本文中，我们将探讨协作式多任务处理以及Rust的async/await功能。我们详细研究了Rust中async/await的工作方式，包括Future trait的设计，状态机转换和pinning。然后，我们将会通过创建异步键盘任务和基本executor，将对async/await的基本支持添加到内核中。
---

该博客在[GitHub上](https://github.com/phil-opp/blog_os)公开开发。如果您有任何问题或疑问，请在此处打开一个issue。您也可以在底部留下评论。这篇文章的完整源代码可以在[post-12](https://github.com/phil-opp/blog_os/tree/post-12)分支中找到。

## 多任务

多数操作系统的基本特征之一是[*多任务处理*](https://en.wikipedia.org/wiki/Computer_multitasking) ，即能够同时执行多个任务的功能。 例如，您可能在看这篇文章时打开了其他程序，例如文本编辑器或终端窗口。 即使只打开一个浏览器窗口，也可能会有各种后台任务来管理桌面窗口，检查更新或为文件建立索引。

尽管看起来所有任务都是并行运行的，但实际上一个CPU核心一次只能执行一个任务。 为了产生任务并行运行的错觉，操作系统会在活动任务之间快速切换，以便每个任务都可以向前推进。 由于计算机速度很快，因此我们大多数时候不会注意到这些切换。

单核CPU一次只能执行一个任务，而多核CPU可以真正并行地运行多个任务。 例如，具有8个内核的CPU可以同时运行8个任务。 我们将在以后的文章中解释如何设置多核CPU。 在本文中，为简单起见，我们将重点介绍单核CPU。 （值得注意的是，所有多核CPU一开始都是以单核启动，因此我们现在可以将它们视为单核CPU。）

多任务处理有两种形式： *协作*多任务处理要求任务定期放弃对CPU的控制，以便其他任务可以取得进展。 *抢占式*多任务使用操作系统功能通过在任意时间点强行暂停线程切换线程。 在下文中，我们将更详细地探讨多任务的两种形式，并讨论它们各自的优点和缺点。

### 抢占式多任务

抢占式多任务处理的思想是操作系统控制何时切换任务。 为此，它利用了它在每个中断上重新获得对CPU的控制这一事实。 这样，只要系统有新输入可用，就可以切换任务。 例如，当移动鼠标或到达网络数据包时，可以切换任务。 操作系统还可以通过配置硬件计时器在该时间之后发送中断来确定允许任务运行的确切时间。

下图说明了硬件中断时的任务切换过程：

![img](https://os.phil-opp.com/async-await/regain-control-on-interrupt.svg)

在第一行中，CPU正在执行程序`A`任务`A1` 。 所有其他任务都已暂停。 在第二行中，硬件中断到达CPU。 如[*硬件中断*](https://os.phil-opp.com/hardware-interrupts)中所述，CPU立即停止执行任务`A1`并跳转到中断描述符表（IDT）中定义的中断处理程序。 通过此中断处理程序，操作系统现在再次具有对CPU的控制权，从而允许它切换到任务`B1`而不是继续执行任务`A1` 。

#### 保存状态

任务在任意时间点都可能被中断，此时它们可能正在进行某些计算。 为了能够在以后继续进行运算，操作系统必须备份任务的整个状态，包括其[调用堆栈](https://en.wikipedia.org/wiki/Call_stack)和所有CPU寄存器的值。 此过程称为[*上下文切换*](https://en.wikipedia.org/wiki/Context_switch) 。

由于调用堆栈可能非常大，因此操作系统通常为每个任务设置单独的调用堆栈，而不是在每每次切换任务时备份整个调用堆栈的内容。 具有单独堆栈的此类任务称为*线程* 。 通过为每个任务使用单独的堆栈，在上下文切换时仅需要保存寄存器内容（包括程序计数器和堆栈指针）。 这种方法可以最大程度地减少上下文切换的性能开销，这非常重要，因为上下文切换通常每秒发生100次。

#### 讨论

抢占式多任务处理的主要优点是操作系统可以完全控制任务的允许执行时间。 这样，它可以确保每个任务都能公平分配CPU时间，而无需信任任务之间的协作。 当运行第三方任务或多个用户共享系统时，这一点尤其重要。

抢占式任务调度的缺点是每个任务都需要自己的堆栈。 与共享堆栈相比，这将导致每个任务的内存使用量更高，并且通常会限制系统中的任务数量。 另一个缺点是，即使任务仅使用了寄存器中的一小部分，操作系统仍必须在每次任务切换时保存完整的CPU寄存器状态。

抢占式多任务处理和线程是操作系统的基本组件，因为它们使运行不受信任的用户空间程序成为可能。 我们将在以后的文章中详细讨论这些概念。 但是，在这篇文章中，我们将专注于协作式多任务处理，这也为我们的内核提供了有用的功能。

### 协作式多任务

协作式多任务处理可以使每个任务运行直到自愿放弃对CPU的控制，而不是在任意时间点强行暂停正在运行的任务。 这使任务可以在方便的时间点（例如，无论如何需要等待I/O操作时）暂停自己的运行。

协作式多任务通常在语言级别使用，例如以[协程](https://en.wikipedia.org/wiki/Coroutine)或[async/await的形式](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html)。 主要想法是由程序员或编译器将[*yield*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Yield_(multithreading)&usg=ALkJrhhQcxajutZrtjPo80ap592S0-q1Xw)操作插入到程序中，从而放弃对CPU的控制并允许其他任务运行。 例如，可以在复杂循环的每次迭代之后插入一个yield。

通常都是将协作式多任务与[异步操作](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Asynchronous_I/O&usg=ALkJrhhFH_sPNGha3oLOtqlv77-mFYnO2Q)结合在一起。 如果操作尚未完成，则异步操作将返回“未就绪”状态，而不是等到操作完成并阻止此时再次执行其他任务。 在这种情况下，等待的任务可以执行yield操作以让其他任务运行。

#### 保存状态

由于任务自己定义了暂停点，因此它们不需要操作系统来保存其状态。 相反，他们可以在暂停之前准确保存需要继续的状态，这通常可以提高性能。 例如，刚完成复杂计算的任务可能只需要备份计算的最终结果，因为它不再需要中间结果。

语言支持的协作任务实现通常甚至可以在暂停之前备份调用堆栈的所需部分。 例如，Rust的async/await实现将所有仍需要的局部变量存储在自动生成的结构中（见下文）。 通过在暂停之前备份调用堆栈的相关部分，所有任务可以共享一个调用堆栈，从而使每个任务的内存消耗大大减少。这样就可以创建几乎任意数量的协作任务，而不会耗尽内存。

#### 讨论

协作式多任务处理的缺点是，不让出执行机会的CPU的协作任务可能会无限期地运行。 因此，恶意或有Bug的任务可能会阻止其他任务运行并减慢甚至使整个系统停止运行。 因此，仅当已知所有任务都会让出执行机会时才应使用协作式多任务处理。 作为反例，操作系统对用户级程序使用协作式多任务调度就不是一个好主意。

但是，协作多任务的强大性能和内存优势使其成为程序*内*使用的好方法，尤其是与异步操作结合使用时。 由于操作系统内核是与异步硬件进行交互的，性能至关重要的程序，因此协作多任务似乎是实现并发的好方法。

## Rust Rust中的Async /Await

Rust语言以async/await的形式为协作式多任务提供了支持。 在探讨什么是async/await及其如何工作之前，我们需要了解*futures*和异步编程如何在Rust中工作。

### Futures

*Future*代表一个可能尚不可用值。 例如，这可能是由另一个任务计算的整数，也可以是从网络下载的文件。 Future无需等到该值可用时，就可以继续执行，直到需要该值为止。

#### 例子

最好用一个小例子来说明Futures的概念：

![Sequence diagram: main calls read_file and is blocked until it returns; then it calls foo() and is also blocked until it returns. The same process is repeated, but this time async_read_file is called, which directly returns a future; then foo() is called again, which now runs concurrently to the file load. The file is available before foo() returns.](https://os.phil-opp.com/async-await/async-example.svg)



此序列图显示了一个`main`函数，该函数从文件系统读取文件，然后调用函数`foo` 。 此过程重复两次：一次通过同步`read_file`调用，一次通过异步`async_read_file`调用。

通过同步调用访问文件时， `main`函数需要等待文件系统加载完文件。 只有这样，它才能调用`foo`函数，这要求它再次等待结果。

通过异步`async_read_file`调用，文件系统直接返回一个`future`并在后台异步加载文件。 这允许`main`函数更早地调用`foo` ，然后`foo`与文件加载并行运行。 在此示例中，文件加载甚至能在`foo`返回之前完成，因此`main`可以直接处理文件，而无需在`foo`返回之后再等待。

#### Rust的Future

在Rust中，Future由[`Future`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html) trait表示，如下所示：

```Rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
}
```

[关联类型](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#specifying-placeholder-types-in-trait-definitions-with-associated-types) `Output`指定异步值的类型。 例如， `async_read_file`函数将返回`Future`实例，并且`Output`设置为`File` 。

[`poll`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/future/trait.Future.html&usg=ALkJrhjkxk--77ETh7OgMyt44sTRaCqxaA#tymethod.poll)方法允许检查该值是否已可用。 它返回一个[`Poll`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/future/trait.Future.html&usg=ALkJrhjkxk--77ETh7OgMyt44sTRaCqxaA#tymethod.poll)枚举，如下所示：

```Rust
pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

当该值已经可用时（例如，已从磁盘完全读取文件），将其包装在`Ready`中返回。 否则，将返回`Pending`，它告知调用方该值尚不可用。

`poll`方法采用两个参数： `self: Pin<&mut Self>`和`cx: &mut Context` 。 前者的行为类似于普通的`&mut self`引用，不同之处在于`self`值 [*固定*](https://doc.rust-lang.org/nightly/core/pin/index.html)在其内存位置 如果不先了解异步/等待的工作原理，就很难了解`Pin`以及为什么需要它。 因此，我们将在本文后面解释。

`cx: &mut Context`参数的目的是将[`Waker`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/task/struct.Waker.html&usg=ALkJrhh1VExi94vM8AqKJrzNZJgsyP4dPw)实例传递给异步任务，例如文件系统负载。 该`Waker`允许异步任务发信号通知它（或其一部分）已完成，例如，文件是从磁盘加载的。 由于主任务知道在`Future`准备就绪时会收到通知，因此它不需要一遍又一遍地调用`poll` 。 当我们实现自己的waker类型时，我们将在本文后面的部分中更详细地说明此过程。

### 使用futures

现在，我们知道了如何定义future并了解`poll`方法背后的基本思想。 但是，我们仍然不知道如何有效地使用futures。 问题在于，future代表了异步任务的结果，这可能尚不可用。 但是，实际上，我们经常直接需要这些值以进行进一步的计算。 所以问题是：我们如何在需要时有效地获取future的值？

#### 等待future

一种可能的方法是等到future准备就绪。 可能看起来像这样：

```Rust
let future = async_read_file("foo.txt");
let file_content = loop {
    match future.poll(…) {
        Poll::Ready(value) => break value,
        Poll::Pending => {}, // do nothing
    }
}
```

在这里，我们在一个循环中调用`poll`来*积极地*等待future。 `poll`的参数在这里无关紧要，因此我们省略了它们。 尽管这个解决方案能用，但效率很低，因为我们一直使CPU忙碌，直到该值可用为止。

一种更有效的方法可能是*阻塞*当前线程，直到将来数据可用。 当然，这只有在有线程的情况下才有可能，因此该解决方案不适用于我们的内核，至少目前还不能。 即使在支持阻塞的系统上，也常常不希望这样做，因为它将再次将异步任务转换为同步任务，从而抑制了并行任务的潜在性能优势。

#### Future组合子

等待的替代方法是使用Future的组合子。 Future组合子是类似于`map`方法，它允许将Future链接和组合在一起，类似于[`Iterator`](https://doc.rust-lang.org/stable/core/iter/trait.Iterator.html)上的方法。 这些组合器不等待Future，而是自己返回一个Future，这将对`poll`应用映射操作。

例如，用于将`Future<Output=String>`转换为`Future<Output=usize>`的简单`string_len`组合器可能如下所示：

```Rust
struct StringLen<F> {
    inner_future: F,
}

impl<F> Future for StringLen<F> where F: Future<Output = String> {
    type Output = usize;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        match self.inner_future.poll(cx) {
            Poll::Ready(s) => Poll::Ready(s.len()),
            Poll::Pending => Poll::Pending,
        }
    }
}

fn string_len(string: impl Future<Output = String>)
    -> impl Future<Output = usize>
{
    StringLen {
        inner_future: string,
    }
}

// Usage
fn file_len() -> impl Future<Output = usize> {
    let file_content_future = async_read_file("foo.txt");
    string_len(file_content_future)
}
```

该代码无法正常工作，因为它不处理[*固定*](https://doc.rust-lang.org/stable/core/pin/index.html)问题，但是作为示例就足够了。 基本思想是`string_len`函数将给定的`Future`实例包装到新的`StringLen`结构中，该结构还实现了`Future` 。 当对打包的future进行轮询时，它将轮询内部的未来。 如果该值尚未准备好，则包装的future也返回`Poll::Pending` 。 如果准备好该值，则从`Poll::Ready`变量中提取字符串，并计算其长度。然后，将其包装在`Poll::Ready`并返回。

使用这个`string_len`函数，我们可以计算异步字符串的长度，而无需等待它。 由于该函数再次返回`Future` ，因此调用者无法直接对返回的值进行操作，而需要再次使用组合器函数。 这样，整个调用图就变得异步了，我们可以高效地在某个时候一次等待多个future，例如在main函数中。

手动编写组合器函数很困难，因此它们通常由库提供。 尽管Rust标准库本身还没有提供组合器方法，但半官方（和`no_std`兼容）的[`futures`](https://docs.rs/futures/0.3.4/futures) crate却提供了。 它的[`FutureExt`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html&usg=ALkJrhip68n2ChVnp4OvzFIkddAvkNxzeA)特性提供了高级组合器方法，例如[`map`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html&usg=ALkJrhip68n2ChVnp4OvzFIkddAvkNxzeA#method.map)或[`then`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html&usg=ALkJrhip68n2ChVnp4OvzFIkddAvkNxzeA#method.then) ，可用于对future结果的操作。

##### 优势

Future组合子的最大优点是它们使操作保持异步。与异步I/O接口结合使用，这种方法可以带来非常高的性能。 Future组合子是通过常规的使用struct和trait实现的，这使得编译器可以对其进行优化。 有关更多详细信息，请参阅《[Rust中的零开销Future](https://aturon.github.io/blog/2016/08/11/futures/)》一文，该文章宣布将Future添加到Rust生态系统中。

##### 缺点

尽管Future组合子可以编写非常有效的代码，但是由于类型系统和基于闭包的接口，在某些情况下它们可能很难使用。 例如，考虑如下代码：

```Rust
fn example(min_len: usize) -> impl Future<Output = String> {
    async_read_file("foo.txt").then(move |content| {
        if content.len() < min_len {
            Either::Left(async_read_file("bar.txt").map(|s| content + &s))
        } else {
            Either::Right(future::ready(content))
        }
    })
}
```

在这里，我们读取文件`foo.txt` ，然后使用[`then`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html&usg=ALkJrhip68n2ChVnp4OvzFIkddAvkNxzeA#method.then)组合器根据文件内容链接第二个future。如果内容长度小于给定的`min_len` ，我们将读取另一个`bar.txt`文件，然后使用[`map`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/trait.FutureExt.html&usg=ALkJrhip68n2ChVnp4OvzFIkddAvkNxzeA#method.map)组合器将其附加到`content` 。 否则，我们仅返回`foo.txt`的内容。

我们需要对传递给`then`的闭包使用[`move`关键字](https://doc.rust-lang.org/std/keyword.move.html) ，因为否则将会对`min_len`造成生命周期错误。 使用[`Either`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/enum.Either.html&usg=ALkJrhhYdLDwZBgIEwMjk4f7ORxn8GK85Q)包装器的原因是if和else块必须始终具有相同的类型。 由于我们在块中返回了不同类型的Future，因此必须使用包装器类型将它们统一为一个类型。 [`ready`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures/0.3.4/futures/future/fn.ready.html&usg=ALkJrhjGlCdXHWuPkApugnx6SwnEFKiQSg)函数将值包装到将来立即准备好的future中。 这里需要该函数，因为`Either`包装器希望包装的值实现`Future` 。

可以想象，这会很快导致大型项目的代码非常复杂。 如果涉及借贷和不同的生存期，则变得特别复杂。 由于这个原因，为了使异步代码从根本上更易于编写，我们投入了大量工作来为Rust添加对异步/等待的支持。

### Async/Await模式

async/await背后的想法是让程序员编写*看起来*像普通同步代码的代码，但被编译器转换为异步代码。 它基于两个关键字`async`和`await`起作用。 可以在函数签名中使用`async`关键字，以将同步函数转换为返回Future的异步函数：

```RustRu s
async fn foo() -> u32 {
    0
}

// the above is roughly translated by the compiler to:
fn foo() -> impl Future<Output = u32> {
    future::ready(0)
}
```

仅此关键字不会有用。 但是在`async`函数内部，可以使用`await`关键字来检索future的异步值：

```Rust
async fn example(min_len: usize) -> String {
    let content = async_read_file("foo.txt").await;
    if content.len() < min_len {
        content + &async_read_file("bar.txt").await
    } else {
        content
    }
}
```

此函数是从[上面](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/async-await/&usg=ALkJrhjPwHsjeYsVEhyo_3NTm2YfROaXvQ#drawbacks)使用组合子的`example`函数的直接翻译。 使用`.await`运算符，我们可以检索future的值，而无需任何闭包或`Either`类型。 结果，我们可以像编写普通的同步代码一样编写代码，区别在于*这仍然是异步代码* 。

#### 状态机转换

编译器在此幕后所做的就是将`async`函数的主体转换为[*状态机*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Finite-state_machine&usg=ALkJrhiQPX47iIIfJQrZOT0XJkKeYLjsdw) ，每个`.await`调用代表一个不同的状态。 对于上面的`example`函数，编译器创建具有以下四个状态的状态机：

![特定状态：开始，等待foo.txt，等待bar.txt，结束](https://os.phil-opp.com/async-await/async-state-machine-states.svg)



每个状态代表该函数的不同暂停点。 *“Start”*和*“End”*状态代表函数在其执行的开始和结束时的状态。 *“Waiting on foo.txt”*状态表示该函数当前正在等待第一个`async_read_file`结果。 同样， *“Waiting on bar.txt”*状态表示功能在第二个`async_read_file`结果上等待的暂停点。

状态机通过使每个`poll`调用可能的状态转换来实现`Future`特性：

![Four states: start, waiting on foo.txt, waiting on bar.txt, end](https://os.phil-opp.com/async-await/async-state-machine-basic.svg)

该图使用箭头表示状态切换，并使用菱形表示选择执行。 例如，如果`foo.txt`文件尚未准备好，则采用标记为*“ no”*的路径，并达到*“ Waiting on foo.txt”*状态。 否则，采用*“yes”*路径。 没有标题的红色小菱形代表`example`函数的`if content.len() < 100`分支。

我们看到，第一个`poll`调用启动了该函数并使它运行，直到到达尚未准备好的Future为止。 如果路径上的所有Future都准备就绪，则该函数可以运行到*“ End”*状态，并返回包装在`Poll::Ready`中的结果 。否则，状态机进入等待状态并返回`Poll::Pending` 。 在下一个`poll`呼叫中，状态机从上一个等待状态开始，然后重试上一个操作。

#### 保存状态

为了能够从上一个等待状态继续，状态机必须在内部跟踪当前状态。 此外，它必须保存在下一个`poll`调用中继续执行所需的所有变量。 这是编译器真正发挥作用的地方：由于它知道何时使用哪些变量，因此它可以自动生成具有所需变量的结构。

作为示例，编译器为上述`example`函数生成类似于以下的结构：

```Rust
// The `example` function again so that you don't have to scroll up
async fn example(min_len: usize) -> String {
    let content = async_read_file("foo.txt").await;
    if content.len() < min_len {
        content + &async_read_file("bar.txt").await
    } else {
        content
    }
}

// The compiler-generated state structs:

struct StartState {
    min_len: usize,
}

struct WaitingOnFooTxtState {
    min_len: usize,
    foo_txt_future: impl Future<Output = String>,
}

struct WaitingOnBarTxtState {
    content: String,
    bar_txt_future: impl Future<Output = String>,
}

struct EndState {}
```

在“start”和*“waiting on foo.txt”*状态下，需要存储`min_len`参数，因为稍后与`content.len()`比较时需要使用该参数。 *“waiting on foo.txt”*状态还存储了一个`foo_txt_future` ，它表示`async_read_file`调用返回的future。 状态机继续运行时，需要再次轮询此future，因此需要保存它。

*“waiting on bar.txt”*状态包含`content`变量，因为在`bar.txt`准备就绪后，字符串拼接需要使用该变量。 它还存储了一个`bar_txt_future` ，它表示`bar_txt_future`的正在进行中的负载。该结构不包含`min_len`变量，因为在`content.len()`比较之后不再需要它。 在*“End”*状态下，没有存储任何变量，因为该函数已经运行完毕。

请记住，这只是编译器可能生成的代码的示例。 结构名称和字段布局是实现细节，可能有所不同。

#### 整个状态机的类型

尽管确切的编译器生成的代码是实现细节，但有助于理解想象生成的状态机如何查找`example`函数。 我们已经定义了代表不同状态并包含所需变量的结构。 为了在它们之上创建一个状态机，我们可以将它们组合成一个[`enum`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html&usg=ALkJrhhSZuP8PCSTV4Cs7ml_J59xpRHsWQ) ：

```Rust
enum ExampleStateMachine {
    Start(StartState),
    WaitingOnFooTxt(WaitingOnFooTxtState),
    WaitingOnBarTxt(WaitingOnBarTxtState),
    End(EndState),
}
```

我们为每个状态定义一个单独的枚举变量，并将对应的状态结构作为字段添加到每个变量。 为了实现状态转换，编译器根据`example`函数生成`Future` trait的实现：

```Rust
impl Future for ExampleStateMachine {
    type Output = String; // return type of `example`

    fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output> {
        loop {
            match self { // TODO: handle pinning
                ExampleStateMachine::Start(state) => {…}
                ExampleStateMachine::WaitingOnFooTxt(state) => {…}
                ExampleStateMachine::WaitingOnBarTxt(state) => {…}
                ExampleStateMachine::End(state) => {…}
            }
        }
    }
}
```

`Output`类型为`String`因为它是`example`函数的返回类型。 为了实现`poll`函数，我们在`loop`内的当前状态上使用match语句。 这个想法是我们尽可能长时间地切换到下一个状态，并在无法继续时使用显式`return Poll::Pending` 。

为简单起见，我们仅显示简化的代码，不处理[固定](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/core/pin/index.html&usg=ALkJrhgBb6s7BfzBtFOTJvpa-SiXi4xloQ) ，所有权，生命周期等。因此，此代码和以下代码应视为伪代码，不能直接使用。 当然，真正的编译器生成的代码可以正确处理所有内容，尽管可能以不同的方式进行。

为了使代码片段较小，我们将分别显示每个match分支的代码。 让我们从`Start`状态开始：

```Rust
ExampleStateMachine::Start(state) => {
    // from body of `example`
    let foo_txt_future = async_read_file("foo.txt");
    // `.await` operation
    let state = WaitingOnFooTxtState {
        min_len: state.min_len,
        foo_txt_future,
    };
    *self = ExampleStateMachine::WaitingOnFooTxt(state);
}
```

在函数一开始调用时，状态机就处于`Start`状态。 在这种情况下，我们将执行`example`函数主体中的所有代码，直到第一个`.await` 。 为了处理`.await`操作，我们将`self`状态机的状态更改为`WaitingOnFooTxt` ，其中包括`WaitingOnFooTxtState`结构的构造。

由于`match self {…}`语句是在循环中执行的，因此该执行将跳转到下一个`WaitingOnFooTxt`：

```Rust
ExampleStateMachine::WaitingOnFooTxt(state) => {
    match state.foo_txt_future.poll(cx) {
        Poll::Pending => return Poll::Pending,
        Poll::Ready(content) => {
            // from body of `example`
            if content.len() < state.min_len {
                let bar_txt_future = async_read_file("bar.txt");
                // `.await` operation
                let state = WaitingOnBarTxtState {
                    content,
                    bar_txt_future,
                };
                *self = ExampleStateMachine::WaitingOnBarTxt(state);
            } else {
                *self = ExampleStateMachine::End(EndState));
                return Poll::Ready(content);
            }
        }
    }
}
```

在此匹配分支中，我们首先调用`foo_txt_future`的`poll`函数。 如果其尚未准备好，则退出循环并返回`Poll::Pending` 。 由于在这种情况下`self`保持在`WaitingOnFooTxt`状态，因此状态机上的下一个`poll`调用将进入相同的匹配并再试一次。

当`foo_txt_future`准备就绪时，我们将结果分配给`content`变量，然后继续执行`example`函数的代码：如果`content.len()`小于状态struct中保存的`min_len`则异步读取`bar.txt`文件。   我们再次将`.await`操作转换为状态更改，这次转换为`WaitingOnBarTxt`状态。 由于我们是在循环内执行`match` ，因此执行会直接跳到新状态的匹配分支，然后轮询`bar_txt_future` 。

如果我们进入`else`分支，则不会进行进一步的`.await`操作。 我们到达函数的结尾，并返回包装在`Poll::Ready` 。 我们还将当前状态更改为`End`状态。

`WaitingOnBarTxt`状态的代码如下所示：

```Rust
ExampleStateMachine::WaitingOnBarTxt(state) => {
    match state.bar_txt_future.poll(cx) {
        Poll::Pending => return Poll::Pending,
        Poll::Ready(bar_txt) => {
            *self = ExampleStateMachine::End(EndState));
            // from body of `example`
            return Poll::Ready(state.content + &bar_txt);
        }
    }
}
```

与`WaitingOnFooTxt`状态类似，我们从轮询`bar_txt_future`开始。 如果仍在等待处理，则退出循环并返回`Poll::Pending` 。 否则，我们可以执行`example`函数的最后一个操作：将`content`变量与将来的结果连接起来。 我们将状态机更新为`End`状态，然后返回包装在`Poll::Ready`的结果。

最后， `End`状态的代码如下所示：

```Rust
ExampleStateMachine::End(_) => {
    panic!("poll called after Poll::Ready was returned");
}
```

Future返回`Poll::Ready`之后，不应再次对其进行轮询，因此，如果在已经处于`End`状态的情况下调用`poll` ，我们将panic。

现在我们知道了编译器生成的状态机及其对`Future` trait的实现。 实际上，编译器以不同的方式生成代码。 （如果您有兴趣，当前的实现是基于[*generator*](https://doc.rust-lang.org/nightly/unstable-book/language-features/generators.html)的 ，但这只是实现的细节。）

难题的最后一部分是为`example`函数本身生成代码。 记住，函数头是这样定义的：

```Rust
async fn example(min_len: usize) -> String
```

由于现在完整的功能主体是由状态机实现的，因此该功能唯一需要做的就是初始化状态机并将其返回。 为此生成的代码如下所示：

```Rust
fn example(min_len: usize) -> ExampleStateMachine {
    ExampleStateMachine::Start(StartState {
        min_len,
    })
}
```

该函数不再具有`async`修饰符，因为它现在显式返回`ExampleStateMachine`类型，该类型实现了`Future`特性。 如预期的那样，状态机被构造为“`Start`”状态，并且使用`min_len`参数初始化了相应的状态结构。

请注意，此功能不会启动状态机的执行。 这是Rust future的一个基本设计决策：在第一次poll之前，它们什么都不做。

### 固定

我们已经在这篇文章中偶然发现了多次*固定* 。 现在终于可以探索固定是什么以及为什么需要固定了。

#### 自引用结构

如上所述，状态机转换将每个暂停点的局部变量存储在结构中。 对于像我们的`example`函数这样的小例子，这很简单，并且不会导致任何问题。 但是，当变量互相引用时，事情变得更加困难。例如，考虑以下函数：

```Rust
async fn pin_example() -> i32 {
    let array = [1, 2, 3];
    let element = &array[2];
    async_write_file("foo.txt", element.to_string()).await;
    *element
}
```

此函数创建一个包含内容`1`，`2`和`3`的小`array` 。 然后，它创建对最后一个数组元素的引用，并将其存储在`element`变量中。 接下来，它将异步将转换为字符串的数字写入`foo.txt`文件。 最后，它返回由`element`引用的数字。

由于该函数使用单个`await`操作，因此结果状态机具有三种状态：开始，结束和“等待写入”。 该函数不带参数，因此开始状态的结构为空。 像以前一样，结束状态的结构也是空的，因为该功能已在此时完成。 “等待写入”状态的结构更有趣：

```Rust
struct WaitingOnWriteState {
    array: [1, 2, 3],
    element: 0x1001a, // address of the last array element
}
```

我们需要存储`array`和`element`变量，因为返回值需要`element` ，而`array`是由`element`引用的。 由于`element`是引用，因此它存储*指向*所引用元素的*指针* （即内存地址）。我们`0x1001a`在这里以内存地址为例。实际上，它必须是`array`字段的最后一个元素的地址，因此它的值取决于该结构在内存中的位置。具有此类内部指针的结构被称为*自引用*结构，因为它们对其字段之一进行引用。

#### 自我指涉的Structs存在的问题

我们的自引用结构的内部指针会导致一个基本的问题，当我们查看其内存布局时，该问题变得显而易见：

![array at 0x10014 with fields 1, 2, and 3; element at address 0x10020, pointing to the last array element at 0x1001a](https://os.phil-opp.com/async-await/self-referential-struct.svg)

这里的`array`字段从地址0x10014开始，`element`字段从地址0x10020开始。它指向地址0x1001a，因为最后一个数组元素位于该地址。到现在为止还看不出什么问题。但是，当我们将此结构移到其他内存地址时，会发生问题：

![在0x10024处具有字段1、2和3的数组； 即使最后一个数组元素现在位于0x1002a，地址0x10030处的元素仍指向0x1001a](https://os.phil-opp.com/async-await/self-referential-struct-moved.svg)

我们稍微移动了结构，使其现在从`0x10024`的地址开始。例如，当我们将struct作为函数参数传递或将其赋值给其他堆栈中的变量时，可能会发生这种情况。问题在于，即使最后一个元素现在位于`0x1002a` ，该`element`字段仍然指向`0x1001a` 。因此，指针现在处于悬垂状态，结果就是在下一次调用时发生未定义的行为。

#### 可能的解决方案

有三种基本方法可以解决悬垂指针问题： 

- **在移动时更新指针：**想法是每当结构在内存中移动时都更新内部指针，以使其在移动后仍然有效。不幸的是，这种方法将需要对Rust进行大量更改，从而可能导致巨大的性能损失。原因是某种运行时需要跟踪所有结构字段的类型，并检查每个移动操作是否需要更新指针。

- **存储偏移量而不是自引用：**为了避免更新指针的要求，编译器可以尝试将自引用存储为结构体开始处的偏移量。例如，`element`上面`WaitingOnWriteState`结构的`element_offset`字段可以以值8的形式存储，因为引用指向的数组元素在结构开始后的8个字节处开始。由于在移动结构时偏移量保持不变，因此不需要字段更新。

  这种方法的问题在于，它要求编译器检测所有自引用。这在编译时是不可能的，因为引用的值可能取决于用户输入，因此我们将再次需要运行时来分析引用并正确创建状态结构。这不仅会导致运行时成本增加，而且还会阻止某些编译器优化，从而会再次导致较大的性能损失。

- **禁止移动该结构：**如上所示，仅当我们在内存中移动该结构时，才会出现悬垂指针。通过完全禁止对自引用结构的移动操作，也可以避免该问题。这种方法的最大优点是，可以在类型系统级别上实现它，而不会增加运行时成本。缺点是它将处理移动操作的负担放在程序员的可能自引用的结构上。

由于其原则是提供*零成本抽象*，这意味着抽象不应施加额外的运行时成本，因此Rust选择了第三个解决方案。为此，在[RFC 2349](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md)中提出了[*固定*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/core/pin/index.html&usg=ALkJrhgBb6s7BfzBtFOTJvpa-SiXi4xloQ) API 。在下文中，我们将简要概述此API，并说明其如何与async / await和futures一起使用。

#### 堆上的值

第一个观察结果是，[堆分配的](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://os.phil-opp.com/heap-allocation/&usg=ALkJrhgeACYVwoeyL3uU2Y6L8zDc9_N_aA)值大多数时候已经具有固定的内存地址。它们是调用`allocate`创建的，然后由指针类型（如`Box`）引用。虽然可以移动指针类型，但指针指向的堆值将保持在相同的内存地址，直到`deallocate`再次通过调用将其释放为止。

使用堆分配，我们可以尝试创建一个自引用结构：

```Rust
fn main() {
    let mut heap_value = Box::new(SelfReferential {
        self_ptr: 0 as *const _,
    });
    let ptr = &*heap_value as *const SelfReferential;
    heap_value.self_ptr = ptr;
    println!("heap value at: {:p}", heap_value);
    println!("internal reference: {:p}", heap_value.self_ptr);
}

struct SelfReferential {
    self_ptr: *const Self,
}
```

我们创建一个名为`SelfReferential`的简单结构，其中包含单个指针字段。首先，我们使用空指针初始化此结构，然后使用将其分配到堆上`Box::new`。然后，我们确定堆分配的结构的内存地址，并将其存储在`ptr`变量中。最后，通过将`ptr`变量分配给`self_ptr`字段，使结构自引用。

在[playground中](https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2018%26gist%3Dce1aff3a37fcc1c8188eeaf0f39c97e8)执行此代码时，我们看到堆值的地址及其内部指针相等，这意味着该`self_ptr`字段是有效的自引用。由于`heap_value`变量只是指针，因此移动变量（例如，通过将其传递给函数）不会更改结构本身的地址，因此`self_ptr`即使指针被移动，其保持有效。

但是，仍然有一种方法可以破坏此示例：我们可以移出`Box`或替换其内容：

```Rust
let stack_value = mem::replace(&mut *heap_value, SelfReferential {
    self_ptr: 0 as *const _,
});
println!("value at: {:p}", &stack_value);
println!("internal reference: {:p}", stack_value.self_ptr);
```

在这里，我们使用该[`mem::replace`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/mem/fn.replace.html&usg=ALkJrhjw0xphkuD3qZU5jcZ2f63TCSH6vg)函数用新的struct实例替换堆分配的值。这使我们可以将原始对象`heap_value`移到堆栈中，而`self_ptr `struct 的字段现在是一个悬垂指针，该指针仍然指向旧的堆地址。当您尝试在playground中运行示例时，您会看到打印的*“ value at：”*和*“ internal reference：”*行确实显示了不同的指针。因此，堆分配值不足以使自引用安全。

导致上述破坏的根本问题是`Box`允许我们获得`&mut T`，即对堆分配值的可变引用。该`&mut`引用使使用[`mem::replace`](https://doc.rust-lang.org/nightly/core/mem/fn.replace.html)或[`mem::swap`](https://doc.rust-lang.org/nightly/core/mem/fn.swap.html)来使得堆分配的值无效成为可能。若要解决此问题，我们必须防止`&mut`创建对自引用结构的引用。

#### `Pin<Box<T>>` 和 `Unpin`

固定API以`Pin`包装器类型和`Unpin`标记trait的形式提供了对`&mut T`问题的解决方案。 这些类型背后的思想是对Pin的所有能获取包装中值的的`&mut`的方法，都将其转发给`Unpin` trait。 Unpin trait是一个自动trait，对于除明确选择不实现的类型之外，所有类型均自动实现了`Unpin`。 只要自引用结构明确选择不实现Unpin，就没有（safe的）方法可以从`Pin<Box<T>>`类型获取`&mut T`。 这样做的结果就是保证了它们的内部自引用保持有效。

例如，让我们更新上面的`SelfReferential`类型以选择取消固定：

```Rust
use core::marker::PhantomPinned;

struct SelfReferential {
    self_ptr: *const Self,
    _pin: PhantomPinned,
}
```

我们选择强制不实现`Unping`，方法是添加类型为[`PhantomPinned`](https://doc.rust-lang.org/nightly/core/marker/struct.PhantomPinned.html)的一个字段。此类型是大小为零的标记类型，其唯一目的是*不*实现`Unpin`特征。由于[auto trait](https://doc.rust-lang.org/reference/special-types-and-traits.html#auto-traits)的工作方式，单个不`Unpin`的字段也会使得整个struct `Unpin`。

第二步是将例子中的`Box<SelfReferential>`类型更改为`Pin<Box<SelfReferential>>`类型。最简单的方法是使用[`Box::pin`](https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html#method.pin)函数而不是[`Box::new`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html&usg=ALkJrhgm5P3u85GIcQAK0vWvRdhU5G9TsA#method.new)创建堆分配的值：

```Rust
let mut heap_value = Box::pin(SelfReferential {
    self_ptr: 0 as *const _,
    _pin: PhantomPinned,
});
```

除了更改`Box::new`为`Box::pin`，我们还需要在struct初始化中添加新字段`_pin`。由于`PhantomPinned`是零大小的类型，因此我们只需要使用其类型名称即可对其进行初始化。

现在，当我们[尝试运行调整后的示例时](https://play.rust-lang.org/%3Fversion%3Dstable%26mode%3Ddebug%26edition%3D2018%26gist%3D961b0db194bbe851ff4d0ed08d3bd98a)，我们发现它不能编译了：

```shell
error[E0594]: cannot assign to data in a dereference of `std::pin::Pin<std::boxed::Box<SelfReferential>>`
  --> src/main.rs:10:5
   |
10 |     heap_value.self_ptr = ptr;
   |     ^^^^^^^^^^^^^^^^^^^^^^^^^ cannot assign
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `std::pin::Pin<std::boxed::Box<SelfReferential>>`

error[E0596]: cannot borrow data in a dereference of `std::pin::Pin<std::boxed::Box<SelfReferential>>` as mutable
  --> src/main.rs:16:36
   |
16 |     let stack_value = mem::replace(&mut *heap_value, SelfReferential {
   |                                    ^^^^^^^^^^^^^^^^ cannot borrow as mutable
   |
   = help: trait `DerefMut` is required to modify through a dereference, but it is not implemented for `std::pin::Pin<std::boxed::Box<SelfReferential>>`
```

发生这两个错误是因为`Pin<Box<SelfReferential>>`类型不再实现`DerefMut` trait。这正是我们想要的，因为`DerefMut` trait将返回一个`&mut`引用，我们希望避免这样做。这正是因为我们选择了不自动实现`Unpin`并更改`Box::new`为`Box::pin`。

现在的问题是，编译器不仅禁止了在第16行中移动这个类型的变量，而且还禁止了在第10行中初始化`self_ptr`字段。这是因为编译器无法区分`&mut`引用的有效使用和无效使用。为了正确初始化这个结构体，我们必须使用unsafe的 [`get_unchecked_mut`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html&usg=ALkJrhidyLKfTsLxgp8abZumPURfnAi4GQ#method.get_unchecked_mut)方法：

```rust
// safe because modifying a field doesn't move the whole struct
unsafe {
    let mut_ref = Pin::as_mut(&mut heap_value);
    Pin::get_unchecked_mut(mut_ref).self_ptr = ptr;
}
```

该[`get_unchecked_mut`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html&usg=ALkJrhidyLKfTsLxgp8abZumPURfnAi4GQ#method.get_unchecked_mut)函数在 `Pin<&mut T>`工作，而不是在`Pin<Box<T>>`上，因此我们必须先使用[`Pin::as_mut`](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.as_mut)转换值。然后，我们可以使用`get_unchecked_mut`返回的可变引用来设置字段`self_ptr`。

现在剩下的唯一错误来源于`mem::replace`。此操作尝试将堆上分配的值移动到栈上，这会破坏存储在`self_ptr`字段中的自引用。通过选择不自动实现`Unpin`并使用`Pin<Box<T>>`，我们可以防止在编译时执行此操作，从而可以安全地使用自引用结构。如我们所见，编译器（暂时）无法证明自引用的创建是安全的，因此我们需要使用unsafe块并自己验证其正确性。

#### 栈上的固定和`Pin<&mut T>`

在上一节中，我们学习了如何用`Pin<Box<T>>`安全地创建分配了堆的自引用值。尽管这种方法可以很好地工作并且相对safe（除了unsafe的构造），但是所需的堆分配会带来性能成本。由于Rust一直希望尽可能提供*零成本的抽象*，因此固定API还允许创建`Pin<&mut T>`指向栈上分配的值的实例。

与*拥有*包装值*所有权*的`Pin<Box<T>>`实例不同， `Pin<&mut T>`实例仅临时借用包装值。这使事情变得更加复杂，因为它要求程序员自己提供一些额外的保证。最重要的是，一个`Pin<&mut T>`实例必须在被引用的整个生命周期中保持固定状态，这对于基于堆栈的变量来说很难验证。为了解决这个问题，有类似[`pin-utils`](https://docs.rs/pin-utils/0.1.0-alpha.4/pin_utils/)这样的crate的存在，但除非您真的知道自己在做什么，否则我仍然不建议固定在堆栈上的变量。

要进一步阅读，请查阅[`pin`模块](https://doc.rust-lang.org/nightly/core/pin/index.html)和[`Pin::new_unchecked`](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.new_unchecked)方法的文档。

#### 固定与Future

正如我们在本文中已经看到的那样，[`Future::poll`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法以`Pin<&mut Self>`参数形式使用固定：

```Rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>
```

[如上所示](https://os.phil-opp.com/async-await#self-referential-structs)，此方法采用非常规的`self: Pin<&mut Self>`而非一般的`&mut self`作为参数，从async/await创建的Future实例通常是自引用的。通过将`Self`包装入`Pin`并让编译器让async/await创建的含有自我指涉的Future不自动实现Unpin，它保证了Future在多次调用之间未在内存之间移动。这样可以确保所有内部引用仍然有效。

值得注意的是，在第一个`poll`调用之前移动Future是可以的。这是由于Future是懒惰的，在第一次被poll之前什么都不做。因此生成的状态机的`start`状态，只包含了函数的参数，而不会有内部引用。为了调用`poll`，调用者必须先将future包装到`Pin`，以确保future无法再在内存中移动。由于栈上的固定很难正确完成，因此我建议为此始终使用结合[`Box::pin`](https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html#method.pin)和[`Pin::as_mut`](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.as_mut)的方式。

如果你有兴趣了解如何使用栈上固定安全地实现Future的组合子函数，可以看看`futures`crate中相对较短的[`map`组合子方法](https://docs.rs/futures-util/0.3.4/src/futures_util/future/future/map.rs.html)的源码，以及有关[投影和结构的固定](https://doc.rust-lang.org/stable/std/pin/index.html#projections-and-structural-pinning)的文档。

### Executors 和 Wakers

使用async/await，可以以符合人的习惯却完全异步的方式处理Future。但是，正如我们从上面了解到的那样，在poll之前，Future什么都不做。这意味着我们必须在某个时候`poll`它们，否则异步代码将永远不会执行。

对于单一的Future，我们总是可以使用[如上所述](https://os.phil-opp.com/async-await#waiting-on-futures)的循环手动等待每个Future。但是，这种方法效率很低，并且对于创建大量Future的程序不实用。解决此问题的最常见方法是定义一个全局*Executor*，该*Executor*负责轮询系统中的所有Future，直到完成为止。

#### Executor

*Executor*的目的是允许将Future作为独立的任务生成，通常通过某种`spawn`方法。然后，Executor负责轮询所有Future，直到完成为止。中心化管理所有Future的最大优势在于，每当Future返回`Poll::Pending`时，Executor就可以切换到另一个Future。因此，异步操作可以并行运行，并且CPU保持繁忙。

许多*Executor*的实现也可以利用具有多个CPU内核的系统。他们创建了一个[线程池](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Thread_pool&usg=ALkJrhgkcQe0TaMFtM2LMSjMFyuetBdU6w)，如果有足够的可用工作，该[线程池](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Thread_pool&usg=ALkJrhgkcQe0TaMFtM2LMSjMFyuetBdU6w)就可以利用所有内核，并使用诸如[工作窃取](https://en.wikipedia.org/wiki/Work_stealing)之类的技术来平衡内核之间的负载。嵌入式系统还有一些特殊的执行器实现，它们可以优化以降低延迟和内存开销。

为了避免一遍又一遍地查询Future的开销，Executor通常还利用Rust Future支持的*唤醒* API。

#### Wakers

Waker API背后的想法是，将特殊的[`Waker`](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)类型变量传递给包装在该[`Context`](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)类型中的每次`poll`调用。此`Waker`类型变量由Executor创建，并且异步任务可以用它来表示其（部分）完成。这样Executor在相应的Waker通知它之前，就不再需要不断在先前返回了`Poll::Pending`的future上调用`poll`了。

最好通过一个小例子来说明：

```rust
async fn write_file() {
    async_write_file("foo.txt", "Hello").await;
}
```

此函数将字符串“Hello”异步写入`foo.txt`文件。由于硬盘写入需要一些时间，因此`poll`此将来的首次调用很可能会返回`Poll::Pending`。但是，硬盘驱动器将在内部存储`poll`调用中的`Waker`，并在文件写入磁盘时使用它来通知执行程序。这样，执行程序`poll`在接收唤醒通知之前，不需要浪费任何时间再次poll这个future。

在本文的实现部分中，当我们创建具有唤醒器支持的执行程序时，我们将详细了解`Waker`类型如何工作。

### 合作式多任务？

在本文的开头，我们讨论了抢先式和协作式多任务。抢占式多任务处理依靠操作系统在正在运行的任务之间进行强制切换，而协作式多任务处理则要求任务通过定期的*yield*操作自愿放弃对CPU的控制。协作方法的最大优点是任务可以自己保存状态，从而可以更有效地进行上下文切换，并可以在任务之间共享相同的调用堆栈。

这可能并不明显，但是future和async/await是协作式多任务处理模式的实现： 

- 添加到执行器中的每个Future基本上都是一项合作任务。 

- Future不使用显式的yield操作，而是通过返回

  `Poll::Pending`（或最后的`Poll::Ready`）来放弃对CPU内核的控制。

  - 没有什么可以强迫future放弃CPU。如果他们愿意，他们将永远无法从`poll`中返回，例如通过死循环。
  - 由于每个future都可以阻止Executor中其他future的执行，因此我们需要相信它们不是恶意的。 

- Future在内部存储了在下一次poll被调用时继续执行所需的所有状态。使用async/await，编译器会自动检测所需的所有变量，并将其存储在生成的状态机中。

  - 仅保存继续所需的最小状态。 
  - 由于`poll`方法在返回时会放弃调用堆栈，因此该堆栈可用于轮询其他future。

我们看到Future和async/await非常适合协作多任务处理模式，它们只是使用一些不同的术语。因此，在下文中，我们将互换使用术语“任务”和“future”。

## 实现

现在，我们了解了基于Future和异步/等待的协作式多任务在Rust中的工作原理，是时候向我们的内核添加对它的支持了。由于[`Future`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html) trait是`core`库的一部分，而async/await是语言本身的功能，因此我们无需做任何特殊操作即可在`#![no_std]`内核中使用它。唯一的要求是我们至少使用nightly `2020-03-25` 之后的Rust，因为在此之前async/await 不兼容`no_std`。（那以后的nightly rust版本还没有rustfmt和clippy组件，因此您可能必须将`--force`标志传递给`rustup update`，即使会删除了一些已安装的组件，使用该标志后也会执行更新。）

在使用了一个足够新的nightly Rust 版本后，我们可以开始在我们`main.rs`中使用async/await：

```rust
// in src/main.rs

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

`async_number`函数是一个 `async fn`，因此编译器将其转换为实现了`Future`的状态机。由于该函数仅返回`42`，因此生成的future将在第一次`poll`调用时直接返回`Poll::Ready(42)`。就像`async_number`那样，`example_task`函数也是`async fn`。它等待返回的数字`async_number`，然后使用`println`宏将其打印出来。

要运行`example_task`返回的future，我们需要对其进行调用`poll`，直到它通过返回来表示已完成`Poll::Ready`。为此，我们需要创建一个简单的executor类型。

### 任务

在开始执行执行程序之前，我们创建一个`task`模块，其具有以下`Task`类型：

```rust
// in src/lib.rs

pub mod task;
```

```rust
// in src/task/mod.rs

use core::{future::Future, pin::Pin};
use alloc::boxed::Box;

pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}
```

该`Task`结构是一个包裹了一个堆上的，固定的，且动态派发的输出为空类型`()`的future输出。让我们详细研究一下：

- 我们要求与任务相关的future返回`()`。这意味着任务不会返回任何结果，它们只是为了副作用而执行。例如，上面定义的`example_task`函数没有返回值，但是它会在屏幕上显示一些副作用。
- `dyn`关键字表明我们存储在`Box`中的是一个[*trait对象*](https://doc.rust-lang.org/book/ch17-02-trait-objects.html)中。这意味着Future上的方法是[*动态派发*](https://doc.rust-lang.org/book/ch17-02-trait-objects.html#trait-objects-perform-dynamic-dispatch)的，从而可以在该`Task`类型中存储不同类型的Future 。这很重要，因为每个`async fn`都有自己的类型，我们希望能够创建多个不同的任务。
- 正如我们在[固定](https://os.phil-opp.com/async-await#pinning)一节中所了解的那样，`Pin`类型可以通过将值放在堆上并防止创建对它的`&mut`引用来确保它不能在内存中移动。这很重要，因为async/await生成的future可能是自我指涉的，即包含指向自身的指针，这些指针在移动future时将失效。

为了允许用future创建新`Task`，我们创建一个`new`函数：

```rust
// in src/task/mod.rs

impl Task {
    pub fn new(future: impl Future<Output = ()> + 'static) -> Task {
        Task {
            future: Box::pin(future),
        }
    }
}
```

该函数接受一个输出值为`()`类型的future并通过[`Box::pin`](https://doc.rust-lang.org/nightly/alloc/boxed/struct.Box.html#method.pin)函数将其固定在内存中。然后，将装入Box的future包装在`Task`结构中并返回。`'static`生命周期在此处是必须的，因为返回的`Task`可能存活任意长的时间，因此那个future也需要在那段时间内一直有效。

我们还添加了一个`poll`方法，使executor可以轮询存储的future：

```rust
// in src/task/mod.rs

use core::task::{Context, Poll};

impl Task {
    fn poll(&mut self, context: &mut Context) -> Poll<()> {
        self.future.as_mut().poll(context)
    }
}
```

由于`Future`trait的[`poll`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/future/trait.Future.html&usg=ALkJrhjkxk--77ETh7OgMyt44sTRaCqxaA#tymethod.poll)方法希望在`Pin<&mut T>`类型上调用，因此我们使用[`Pin::as_mut`](https://doc.rust-lang.org/nightly/core/pin/struct.Pin.html#method.as_mut)方法首先转换字段`self.future`的类型。然后，我们在转换后的`self.future`字段上调用`poll`并返回结果。由于该`Task::poll`方法仅应由我们稍后创建的executor调用，因此我们将函数保留为`task`模块私有。

### 简单执行器

由于Executor可能非常复杂，因此，我们刻意从创建一个非常基本的执行程序开始，之后再实现功能更强大的Executor。为此，我们首先创建一个新的`task::simple_executor`子模块：

```rust
// in src/task/mod.rs

pub mod simple_executor;
```

```rust
// in src/task/simple_executor.rs

use super::Task;
use alloc::collections::VecDeque;

pub struct SimpleExecutor {
    task_queue: VecDeque<Task>,
}

impl SimpleExecutor {
    pub fn new() -> SimpleExecutor {
        SimpleExecutor {
            task_queue: VecDeque::new(),
        }
    }

    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
}
```

该结构包含一个[`VecDeque`](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)类型的字段`task_queue`。[`VecDeque`](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)类型基本上是一个允许在两端进行推入和弹出操作的向量。使用这种类型背后的想法是，我们在`spawn`方法的最后插入新任务，并从前面弹出要执行的下一个任务。这样，我们得到一简单的[FIFO队列](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)&usg=ALkJrhhRCKxAHewelZq3weF-AvTwGuZ7hQ)（*“* [先进先出](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/FIFO_(computing_and_electronics)&usg=ALkJrhhRCKxAHewelZq3weF-AvTwGuZ7hQ)*”*）。

#### Dummy Waker

为了调用该`poll`方法，我们需要创建一个[`Context`](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)，作为[`Waker`](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)类型的包装类型。首先，我们将创建一个不执行任何操作的Dummy唤醒器。为此，我们创建一个[`RawWaker`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html&usg=ALkJrhjOuwz5oSe58stzvVL7q0DlBpW03w)实例，该实例定义了不同`Waker`方法的实现，然后使用[`Waker::from_raw`](https://doc.rust-lang.org/stable/core/task/struct.Waker.html#method.from_raw)函数将其转换为`Waker`：

```rust
// in src/task/simple_executor.rs

use core::task::{Waker, RawWaker};

fn dummy_raw_waker() -> RawWaker {
    todo!();
}

fn dummy_waker() -> Waker {
    unsafe { Waker::from_raw(dummy_raw_waker()) }
}
```

该`from_raw`函数是unsafe的，因为如果程序员不遵守文档中对`RawWaker`的要求，则会发生未定义的行为。在研究`dummy_raw_waker`函数的实现之前，我们首先尝试了解`RawWaker`类型的工作方式。

##### `RawWaker`

该[`RawWaker`](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)类型要求程序员明确定义一个[*虚函数表*](https://en.wikipedia.org/wiki/Virtual_method_table)（*vtable*），该*表*指定了在`RawWaker`克隆，唤醒或删除它们时应调用的函数。此vtable的布局由[`RawWakerVTable`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/core/task/struct.RawWakerVTable.html&usg=ALkJrhhw23YeIwD9TejZL0gPQapSx4FlKw)类型定义。每个函数接收一个`*const ()`参数，该参数基本上是指向某个结构的*类型被擦除掉的* `&self`指针，例如，堆上的内存分配。使用`*const ()`指针而不是适当的引用的原因是该`RawWaker`类型应该是非泛型的，但仍支持任意类型。传递给函数的指针值是[`RawWaker::new`](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html#method.new)中传入的`data`指针值。

通常，`RawWaker`为包裹在[`Box`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html&usg=ALkJrhh5TXJ5H6x82XpGXpGzQuQC88TCeg)或[`Arc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html&usg=ALkJrhhytzcnWX7C38Bdbd_rI5EPjRlGig)类型中的一些堆分配的结构创建。对于此类类型，可以使用诸如[`Box::into_raw`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/alloc/boxed/struct.Box.html&usg=ALkJrhh5TXJ5H6x82XpGXpGzQuQC88TCeg#method.into_raw)之类的方法将`Box<T>`转换为`*const T`指针。然后，可以将该指针转换为匿名`*const ()`指针并传递给`RawWaker::new`。由于每个vtable函数接收同样的的`*const ()`参数，因此这些函数可以安全地将指针强制转换回`Box<T>`或 `&T`进行操作。可以想象，此过程非常危险，很容易导致错误的不确定行为。因此，除非必要，否则不建议手动创建`RawWaker`。

##### Dummy`RawWaker`

虽然不建议手动创建`RawWaker` ，但是目前没有其他方法可以创建一个不执行任何操作的dummy `Waker`。幸运的是，我们实际上什么也不做的事实使得实现`dummy_raw_waker`函数相对安全：

```rust
// in src/task/simple_executor.rs

use core::task::RawWakerVTable;

fn dummy_raw_waker() -> RawWaker {
    fn no_op(_: *const ()) {}
    fn clone(_: *const ()) -> RawWaker {
        dummy_raw_waker()
    }

    let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
    RawWaker::new(0 as *const (), vtable)
}
```

首先，我们定义两个名为`no_op`和`clone`的内部函数。`no_op`函数需要一个`*const ()`指针，但不执行任何操作。`clone`函数获取一个`*const ()`指针，并通过`dummy_raw_waker`返回一个新的`RawWaker`。我们使用这两个函数来创建一个最小的`RawWakerVTable`：`clone`函数用于克隆操作，`no_op`函数用于所有其他操作。既然`RawWaker`什么都不做，那么在克隆它时直接返回一个新`RawWaker`而不是真的克隆它也没有关系。

创建`vtable`之后，我们使用[`RawWaker::new`](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html#method.new)函数创建`RawWaker`。传入的`*const ()`无关紧要，因为vtable函数均未使用它。因此，我们只传递一个空指针。

#### `run`方法

现在我们有了创建`Waker`实例的方法，可以使用它在executor上实现一个`run`方法。最简单的`run`方法是在循环中重复轮询所有排队的任务，直到完成所有任务。这不是很高效，因为它没有利用`Waker`类型的通知，但是它是让一切运行起来的简单方法：

```rust
// in src/task/simple_executor.rs

use core::task::{Context, Poll};

impl SimpleExecutor {
    pub fn run(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let waker = dummy_waker();
            let mut context = Context::from_waker(&waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {} // task done
                Poll::Pending => self.task_queue.push_back(task),
            }
        }
    }
}
```

该函数使用`while let`循环来处理`task_queue`中的所有任务。对于每个任务，它首先通过包装由我们的`dummy_waker`函数返回的`Waker`实例来创建一个`Context`类型。然后，它使用这个`context`调用方法`Task::poll`。如果`poll`方法返回`Poll::Ready`，则任务完成，我们可以继续下一个任务。如果任务仍然存在`Poll::Pending`，我们将其再次添加到队列的后面，以便在后续循环迭代中再次对其进行轮询。

#### 试一试

使用我们的`SimpleExecutor`类型，我们现在可以尝试运行`main.rs`中`example_task`函数返回的任务：

```rust
// in src/main.rs

use blog_os::task::{Task, simple_executor::SimpleExecutor};

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including `init_heap`

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.run();

    // […] test_main, "it did not crash" message, hlt_loop
}


// Below is the example_task function again so that you don't have to scroll up

async fn async_number() -> u32 {
    42
}

async fn example_task() {
    let number = async_number().await;
    println!("async number: {}", number);
}
```

当我们运行它时，我们可以看到预期的*“async number: 42”*消息打印到屏幕上：

![QEMU printing "Hello World", "async number: 42", and "It did not crash!"](https://os.phil-opp.com/async-await/qemu-simple-executor.png)

让我们总结一下此示例发生的各个步骤： 

- 首先，使用一个空的`task_queue`创建一个我们`SimpleExecutor`类型的新实例。

- 接下来，我们调用异步`example_task`函数，该函数返回一个future。我们将这个future包装在`Task`类型中，将其移动到堆中并固定，然后通过`spawn`方法将任务添加到`Executor`的`task_queue`。

- 然后，我们调用该`run`方法以开始执行队列中的单个任务。

  这涉及：

  - 从`task_queue`头部弹出任务。
  - 为任务创建一个`RawWaker`，将其转换为[`Waker`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/task/struct.Waker.html&usg=ALkJrhh1VExi94vM8AqKJrzNZJgsyP4dPw)实例，然后从中创建一个[`Context`](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)实例。
  - 使用我们刚创建的`Context`实例在任务的future上调用[`poll`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法。
  - 由于`example_task`无需等待，因此可以直接在第一次`poll`调用就运行到底。也就会打印出*“async number:42”*。
  - 由于`example_task`直接返回`Poll::Ready`，因此不会将其添加回任务队列。

- `task_queue`变为空后，`run`方法返回。我们的`kernel_main`功能继续执行，并且打印"*It did not crash!*"信息。

### 异步键盘输入

我们简单的executor不使用`Waker`通知，而是循环遍历所有任务，直到完成为止。对于我们的例子来说，这不是问题，因为我们`example_task`可以直接在第一次`poll`调用时就运行完。要观察适当`Waker`实现的性能优势，我们首先需要创建一个真正异步的任务，即在第一次`poll`调用时可能返回`Poll::Pending`的任务。

我们的系统中已经存在某种异步性：硬件中断。正如我们在[*硬件中断一文中*](https://os.phil-opp.com/hardware-interrupts)了解到的那样，硬件中断可以在取决于某些外部设备的任意时间发生。例如，在经过一段预定义的时间后，硬件计时器将中断发送到CPU。当CPU接收到中断时，它将立即将控制权转移到在中断描述符表（IDT）中定义的相应处理程序。

下面，我们将基于键盘中断创建一个异步任务。键盘中断是一个很好的选择，因为它既不确定又对延迟至关重要。非确定性意味着无法预测下一次按键的发生时间，因为它完全取决于用户。延迟关键性意味着我们要及时处理键盘输入，否则用户会感到系统卡顿。为了以有效的方式支持此类任务，executor必须具有适当的`Waker`通知支持，这一点至关重要。

#### 扫描码队列

当前，我们直接在中断处理程序中处理键盘输入。从长远来看，这不是一个好主意，因为中断处理程序应尽可能短，因为它们可能会中断重要的工作。更好的方案是，中断处理程序仅执行必要的最少工作量（例如，读取键盘扫描码），而将其余工作（例如，解释扫描码）留给后台任务。

将工作委派给后台任务的常见模式是创建某种队列。中断处理程序将工作单元推送到队列，而后台任务处理队列中的工作。应用于我们的键盘中断，这意味着中断处理程序仅从键盘读取扫描代码，然后将其推送到队列，然后返回。键盘任务位于队列的另一端，并解释和处理推入队列的每个扫描代码：



![Scancode queue with 8 slots on the top. Keyboard interrupt handler on the bottom left with a "push scancode" arrow to the left of the queue. Keyboard task on the bottom right with a "pop scancode" queue coming from the right side of the queue.](https://os.phil-opp.com/async-await/scancode-queue.svg)



该队列的一个简单实现可以是用互斥锁保护的[`VecDeque`](https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html)。但是，在中断处理程序中使用互斥锁不是一个好主意，因为它很容易导致死锁。例如，当用户在键盘任务锁定队列时按下某个键时，中断处理程序将尝试再次获取该锁定并无限期挂起。这种方法的另一个问题是，`VecDeque`满时，会进行新的堆分配来自动增加其容量。这可能再次导致死锁，因为我们的分配器还在内部使用了互斥锁。更进一步的问题是，当堆碎片化时，堆分配可能会失败或花费大量时间。

为避免这些问题，我们需要更好的一个队列实现，该队列实现不需要互斥量或在`push`操作时分配内存。可以通过使用无锁[原子操作](https://doc.rust-lang.org/core/sync/atomic/index.html)来推送和弹出元素来实现此类队列。这样，`push`和`pop`操作仅需要`&self`引用，因此无需互斥即可使用。为了避免在`push`时分配内存，可以使用预先分配的固定大小的缓冲区来支持队列。虽然这使队列*有界*（即具有一个固定的最大长度），但实际上通常可以为队列长度定义合理的上限，因此这样做问题不大。

##### `crossbeam` Crate

以正确有效的方式实现这样的队列非常困难，因此我建议坚持使用经过良好测试的现有实现。有一个流行的Rust项目[`crossbeam`](https://github.com/crossbeam-rs/crossbeam)实现了并发编程的各种无锁类型。它提供了一个名为[`ArrayQueue`](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html)的类型，正是这种情况下我们需要的类型。而且我们很幸运：该类型与具有内存分配支持的`no_std` crate完全兼容。

要使用该类型，我们需要添加一个依赖项`crossbeam-queue` crate：

```toml
# in Cargo.toml

[dependencies.crossbeam-queue]
version = "0.2.1"
default-features = false
features = ["alloc"]
```

默认情况下，crate使用了标准库。为了使其与`no_std`兼容，我们需要禁用其默认功能，同时启用该`alloc`功能。（请注意，`crossbeam` crate在这里不能用，因为它在`no_std`下没有导出 `queue`模块。我提出了[拉取请求](u=https://github.com/crossbeam-rs/crossbeam/pull/480)以解决此问题，但这个版本尚未在crates.io上发布。）

##### 队列的实现

使用该`ArrayQueue`类型，我们现在可以在新`task::keyboard`模块中创建全局扫描代码队列：

```rust
// in src/task/mod.rs

pub mod keyboard;
```

```rust
// in src/task/keyboard.rs

use conquer_once::spin::OnceCell;
use crossbeam_queue::ArrayQueue;

static SCANCODE_QUEUE: OnceCell<ArrayQueue<u8>> = OnceCell::uninit();
```

由于[`ArrayQueue::new`](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.htm#method.new)执行堆分配，而这（[暂时](https://github.com/rust-lang/const-eval/issues/20)）是在编译时不能进行的，因此我们无法直接初始化静态变量。相反，我们使用[`conquer_once`](https://docs.rs/conquer-once/0.2.0/conquer_once/index.html) crate 的[`OnceCell`](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html)类型，这使得执行安全的静态值一次性初始化成为可能。要使用这个crate，我们需要将其作为依赖项添加到我们的`Cargo.toml`：

```toml
# in Cargo.toml

[dependencies.conquer-once]
version = "0.2.0"
default-features = false
```

除了[`OnceCell`](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html)，我们还可以在此处使用[`lazy_static`](https://docs.rs/lazy_static/1.4.0/lazy_static/index.html)宏。但是，`OnceCell`类型的优点是我们可以确保初始化不会在中断处理程序中发生，从而防止了在中断处理程序中执行堆分配。

#### 填充队列

为了填充扫描代码队列，我们创建了一个新`add_scancode`函数，我们将会从中断处理程序中调用它：

```rust
// in src/task/keyboard.rs

use crate::println;

/// Called by the keyboard interrupt handler
///
/// Must not block or allocate.
pub(crate) fn add_scancode(scancode: u8) {
    if let Ok(queue) = SCANCODE_QUEUE.try_get() {
        if let Err(_) = queue.push(scancode) {
            println!("WARNING: scancode queue full; dropping keyboard input");
        }
    } else {
        println!("WARNING: scancode queue uninitialized");
    }
}
```

我们使用[`OnceCell::try_get`](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)来获取对初始化过的队列的引用。如果队列尚未初始化，我们将忽略键盘扫描码并打印警告。重要的是我们不会尝试在此函数中初始化队列，因为它将由中断处理程序调用，该中断处理程序不应执行堆分配。由于不应从`main.rs`调用此函数，我们使用`pub(crate)`可见性使得其仅在`lib.rs`中可用。

[`ArrayQueue::push`](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html#method.push)方法仅需要`&self`引用，这使得在静态队列上调用该方法非常简单。`ArrayQueue`类型本身会执行所有必要的同步，因此这里不需要互斥包装。如果队列已满，我们也会打印警告。

为了在键盘中断时调用`add_scancode`函数，我们要更新在`interrupts`模块中的`keyboard_interrupt_handler`函数：

```rust
// in src/interrupts.rs

extern "x86-interrupt" fn keyboard_interrupt_handler(
    _stack_frame: &mut InterruptStackFrame
) {
    use x86_64::instructions::port::Port;

    let mut port = Port::new(0x60);
    let scancode: u8 = unsafe { port.read() };
    crate::task::keyboard::add_scancode(scancode); // new

    unsafe {
        PICS.lock()
            .notify_end_of_interrupt(InterruptIndex::Keyboard.as_u8());
    }
}
```

我们从该函数中删除了所有键盘处理代码，而是添加了对`add_scancode`函数的调用。其余功能与以前相同。

不出所料，当我们`cargo xrun`现在使用项目运行项目时，按键不再显示在屏幕上。取而代之的是，每次击键时我们都会看到未初始化scancode队列的警告。

#### 键盘扫描码Stream

要`SCANCODE_QUEUE`以异步方式初始化并从队列中读取扫描代码，我们需要创建一个新`ScancodeStream`类型：

```rust
// in src/task/keyboard.rs

pub struct ScancodeStream {
    _private: (),
}

impl ScancodeStream {
    pub fn new() -> Self {
        SCANCODE_QUEUE.try_init_once(|| ArrayQueue::new(100))
            .expect("ScancodeStream::new should only be called once");
        ScancodeStream { _private: () }
    }
}
```

`_private`域的目的是防止从模块外部构造结构。这使`new`函数成为构造该类型的唯一方法。在函数中，我们首先尝试初始化`SCANCODE_QUEUE`静态变量。如果它已经初始化，则panic，以确保只能为`ScancodeStream`创建一个实例。

为了使扫描代码可用于异步任务，下一步是实现与`poll`类似的方法，该方法尝试将下一个扫描代码弹出队列。虽然这听起来像我们应该为我们的类型实现[`Future`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/future/trait.Future.html&usg=ALkJrhjkxk--77ETh7OgMyt44sTRaCqxaA) trait，但这在这里并不完全合适。问题在于，`Future`特征仅抽象出一个异步值，并且期望该`poll`方法返回`Poll::Ready`后不再被调用。但是，我们的扫描代码队列包含多个异步值，因此可以继续对其进行polling。

##### `Stream` Trait

由于产生多个异步值的类型很常见，所以[`futures`](https://docs.rs/futures/0.3.4/futures) crate为此类类型提供了有用的抽象：[`Stream`](https://rust-lang.github.io/async-book/05_streams/01_chapter.html) trait。Trait定义如下：

```rust
pub trait Stream {
    type Item;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context)
        -> Poll<Option<Self::Item>>;
}
```

此定义与[`Future`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html) trait非常相似，但有以下区别：

- 关联的类型名为`Item`而不是`Output`。
- 代替返回`Poll<Self::Item>`的`poll`方法，`Stream` trait定义了`poll_next`方法，该方法返回一个`Poll<Option<Self::Item>>`（注意多出来的`Option`）。

还有一个语义上的区别：`poll_next`可以重复调用，直到返回`Poll::Ready(None)`以表明流已完成。在这方面，该方法类似于在最后一个值之后[`Iterator::next`](https://doc.rust-lang.org/stable/core/iter/trait.Iterator.html#tymethod.next)方法返回`None`。

##### 实现`Stream`

让我们为`ScancodeStream`实现`Stream` trait来以异步方式提供`SCANCODE_QUEUE`的值。为此，我们首先需要添加对`futures-util` crate的依赖关系，其中包含`Stream`类型：

```toml
# in Cargo.toml

[dependencies.futures-util]
version = "0.3.4"
default-features = false
features = ["alloc"]
```

我们禁用默认feature以使crate 与 `no_std`兼容，同时也要启用`alloc ` feature以使其基于分配的类型可用（稍后我们将需要它）。（请注意，我们还可以在主`futures`包装箱上添加一个依赖项，从而重新导出该`futures-util`包装箱，但这将导致更多的依赖项和更长的编译时间。）

现在我们可以导入并实现`Stream`特征：

```rust
// in src/task/keyboard.rs

use core::{pin::Pin, task::{Poll, Context}};
use futures_util::stream::Stream;

impl Stream for ScancodeStream {
    type Item = u8;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<u8>> {
        let queue = SCANCODE_QUEUE.try_get().expect("not initialized");
        match queue.pop() {
            Ok(scancode) => Poll::Ready(Some(scancode)),
            Err(crossbeam_queue::PopError) => Poll::Pending,
        }
    }
}
```

我们首先使用该[`OnceCell::try_get`](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)方法来获取对初始化的扫描代码队列的引用。这应该永远不会失败，因为我们在`new`函数中初始化了队列，我们可以安全地使用`expect`方法在未初始化队列时panic。接下来，我们使用[`ArrayQueue::pop`](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html#method.pop)方法尝试从队列中获取下一个元素。如果成功，我们返回包裹在`Poll::Ready(Some(…))`中的键盘扫描代码。如果失败，则表示队列为空。在这种情况下，我们返回`Poll::Pending`。

#### Waker支持

与该`Futures::poll`方法类似，`Stream::poll_next`方法要求异步任务在`Poll::Pending`返回后准备就绪时通知执行程序。这样，执行程序无需再次poll相同的任务，直到waker通知它为止，这大大降低了等待任务的性能开销。

要发送此通知，任务应从传入的[`Context`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/task/struct.Context.html&usg=ALkJrhienQeYRRGlCkB2M_ymB7qxtTE8Gg)引用中提取[`Waker`](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)并将其存储在某处。当任务准备就绪时，它应该调用`Waker`存储的[`wake`](https://doc.rust-lang.org/stable/core/task/struct.Waker.html#method.wake)方法以通知executor应该再次轮询任务。

##### AtomicWaker

要为`ScancodeStream`实现`Waker`通知，我们需要一个可以在两次轮询调用之间存储`Waker`的位置。我们不能将其存储为`ScancodeStream`本身的字段，因为需要从`add_scancode`函数中对其进行访问。解决方案是使用crate `futures-util`提供的 [`AtomicWaker`](https://docs.rs/futures-util/0.3.4/futures_util/task/struct.AtomicWaker.html)类型的静态变量。与`ArrayQueue`类型一样，此类型也基于原子指令，可以安全地存储在static变量中并可以并发安全地进行修改。

让我们使用[`AtomicWaker`](https://docs.rs/futures-util/0.3.4/futures_util/task/struct.AtomicWaker.htmlA)类型来定义一个静态变量 `WAKER`：

```rust
// in src/task/keyboard.rs

use futures_util::task::AtomicWaker;

static WAKER: AtomicWaker = AtomicWaker::new();
```

`poll_next`的实现会将waker放在这个static里，并且当`add_scancode`将新的扫描代码添加到队列时会调用这个`wake`函数。

##### 存放Waker

`poll`/`poll_next` 要求任务在返回 `Poll::Pending` 时为传入的Waker注册一个唤醒函数。让我们修改`poll_next`实现以满足这些要求：

```rust
// in src/task/keyboard.rs

impl Stream for ScancodeStream {
    type Item = u8;

    fn poll_next(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Option<u8>> {
        let queue = SCANCODE_QUEUE
            .try_get()
            .expect("scancode queue not initialized");

        // fast path
        if let Ok(scancode) = queue.pop() {
            return Poll::Ready(Some(scancode));
        }

        WAKER.register(&cx.waker());
        match queue.pop() {
            Ok(scancode) => {
                WAKER.take();
                Poll::Ready(Some(scancode))
            }
            Err(crossbeam_queue::PopError) => Poll::Pending,
        }
    }
}
```

像之前一样，我们首先使用[`OnceCell::try_get`](https://docs.rs/conquer-once/0.2.0/conquer_once/raw/struct.OnceCell.html#method.try_get)函数来获取对初始化的scancode队列的引用。然后，我们乐观地尝试`pop`从队列中返回并`Poll::Ready`在成功时返回。这样，我们可以避免在队列不为空时注册唤醒程序的性能开销。

如果第一次调用`queue.pop()`失败，则队列可能为空。这里用“可能”是因为中断处理程序可能在检查后立即异步填充了队列。由于此竞态条件可能在下一次检查时再次出现，因此我们需要在第二次检查之前在`WAKER`静态变量中注册`Waker`。这样，在我们返回`Poll::Pending`之前可能会发生唤醒，但是这可以确保在检查后推送的所有扫描代码都能够触发唤醒。

在通过函数[`AtomicWaker::register`](https://docs.rs/futures-util/0.3.4/futures_util/task/struct.AtomicWaker.html#method.register)注册了传入的[`Context`](https://doc.rust-lang.org/nightly/core/task/struct.Context.html)中的`Waker`后，我们尝试再次从队列中弹出。如果现在成功了，我们将返回`Poll::Ready`。我们还使用[`AtomicWaker::take`](https://docs.rs/futures/0.3.4/futures/task/struct.AtomicWaker.html#method.take)再次删除了已注册的唤醒器，因为不再需要唤醒器通知。万一`queue.pop()`第二次失败，我们会像以前一样返回`Poll::Pending`，但是这次注册过了唤醒。

请注意，对于不返回`Poll::Pending`的任务，也在两种情况下可能会被唤醒。一种方法是在返回`Poll::Pending`之前立即发生唤醒时提到的竞态条件。另一种方法是在注册唤醒程序后队列不再为空时`Poll::Ready`返回。由于这些伪造的唤醒是无法避免的，因此执行者需要能够正确处理它们。

##### 唤醒存储的Waker

要唤醒存储的`Waker`，我们在`add_scancode`函数中添加了对`WAKER.wake()`的调用：

```rust
// in src/task/keyboard.rs

pub(crate) add_scancode(scancode: u8) {
    if let Ok(queue) = SCANCODE_QUEUE.try_get() {
        if let Err(_) = scancode_queue.push(scancode) {
            println!("WARNING: scancode queue full; dropping keyboard input");
        } else {
            WAKER.wake(); // new
        }
    } else {
        println!("WARNING: scancode queue uninitialized");
    }
}
```

我们做的唯一更改是，如果成功推送到scancode队列，则添加一个对`WAKER.wake()`的调用。如果唤醒程序在`WAKER`静态变量中注册过，则此方法将在其上调用同名方法[`wake`](https://doc.rust-lang.org/stable/core/task/struct.Waker.html#method.wake)，该方法将会通知executor。否则，该操作为空操作，即什么都不会发生。

重要的是，我们仅在推送到队列后才调用`wake`，因为否则当队列仍然为空时，可能会太早唤醒任务。例如，当使用多线程执行程序在另一个CPU内核上同时启动唤醒的任务时，可能会发生这种情况。虽然我们还没有支持线程，但我们会尽快添加它，并且我们不希望那时出现问题。

#### 键盘任务

现在，我们已经为`ScancodeStream`实现了`Stream  `trait，我们可以使用它来创建异步键盘任务：

```rust
// in src/task/keyboard.rs

use futures_util::stream::StreamExt;
use pc_keyboard::{layouts, DecodedKey, HandleControl, Keyboard, ScancodeSet1};
use crate::print;

pub async fn print_keypresses() {
    let mut scancodes = ScancodeStream::new();
    let mut keyboard = Keyboard::new(layouts::Us104Key, ScancodeSet1,
        HandleControl::Ignore);

    while let Some(scancode) = scancodes.next().await {
        if let Ok(Some(key_event)) = keyboard.add_byte(scancode) {
            if let Some(key) = keyboard.process_keyevent(key_event) {
                match key {
                    DecodedKey::Unicode(character) => print!("{}", character),
                    DecodedKey::RawKey(key) => print!("{:?}", key),
                }
            }
        }
    }
}
```

该代码与我们在本文中对其进行修改之前的[键盘中断处理程序中](https://os.phil-opp.com/hardware-interrupts#interpreting-the-scancodes)使用的代码非常相似。唯一的区别是，我们不是从I/O端口读取扫描代码，而是从`ScancodeStream`那里获取它。为此，我们首先创建一个新`Scancode`流，然后重复使用[`StreamExt`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/futures-util/0.3.4/futures_util/stream/trait.StreamExt.html&usg=ALkJrhi9qF4h34_mgzwOmdMkkTUAoNktzg) trait提供的[`next`](https://docs.rs/futures-util/0.3.4/futures_util/stream/trait.StreamExt.html#method.next)方法来获取包裹了流中下一个元素的`Future`。通过在其上使用`await`，我们异步等待将来的结果。

我们使用`while let`循环，直到流返回`None`以表示其结束。由于我们的`poll_next`方法从不返回`None`，因此实际上这是一个无休止的循环，因此`print_keypresses`任务永远不会完成。

让我们将`print_keypresses`任务添加到`main.rs`的执行器中，以便再次获得有效的键盘输入：

```rust
// in src/main.rs

use blog_os::task::keyboard; // new

fn kernel_main(boot_info: &'static BootInfo) -> ! {

    // […] initialization routines, including init_heap, test_main

    let mut executor = SimpleExecutor::new();
    executor.spawn(Task::new(example_task()));
    executor.spawn(Task::new(keyboard::print_keypresses())); // new
    executor.run();

    // […] "it did not crash" message, hlt_loop
}
```

现在执行时`cargo xrun`，我们看到键盘输入再次能用了：

![QEMU printing ".....H...e...l...l..o..... ...W..o..r....l...d...!"](https://os.phil-opp.com/async-await/qemu-keyboard-output.gif)

如果您关注计算机的CPU利用率，您会发现该`QEMU`进程现在持续使CPU处于繁忙状态。发生这种情况是因为我们的`SimpleExecutor`poll任务反复循环进行。因此，即使我们没有按键盘上的任何键，执行器也会重复调用`poll`我们的`print_keypresses`任务，即使该任务无法取得任何进展并且`Poll::Pending`每次都会返回。

### Waker支持下的Executor

为了解决性能问题，我们需要创建一个可以正确利用`Waker`通知的执行程序。这样，当下一个键盘中断发生时，将通知执行程序，因此它不需要一遍又一遍地poll`print_keypresses`任务。

#### 任务编号

创建支持waker通知的执行程序的第一步是为每个任务赋予唯一的ID。这是必需的，因为我们需要一种方法来指定应唤醒的任务。我们首先创建一个新的`TaskId`包装器类型：

```rust
// in src/task/mod.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
struct TaskId(usize);
```

`TaskId`结构是对`usize`的简单包装。我们为它derive了许多trait，使其可打印，可复制，可比较和可排序。后者很重要，因为我们想马上将其`TaskId`用作[`BTreeMap`](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html)的键类型。

为了为每个任务分配唯一的ID，我们利用了每个任务都存储固定的，由堆分配的Future这一特点：

```rust
pub struct Task {
    future: Pin<Box<dyn Future<Output = ()>>>,
}
```

主要想法是使用此future的内存地址作为ID。该地址是唯一的，因为没有两个future存储在同一地址。`Pin`类型确保他们不能在内存中移动，所以我们也知道地址保持不变，只要任务存在。这些属性使地址成为ID的良好候选者。

实现看起来像这样：

```rust
// in src/task/mod.rs

impl Task {
    fn id(&self) -> TaskId {
        use core::ops::Deref;

        let addr = Pin::deref(&self.future) as *const _ as *const () as usize;
        TaskId(addr)
    }
}
```

我们使用[`Deref`](https://doc.rust-lang.org/core/ops/trait.Deref.html) trait的`deref`方法来获取对将来分配的堆的引用。为了获得相应的内存地址，我们将此引用转换为原始指针，然后转换为`usize`。最后，我们返回包装在`TaskId`结构中的地址。

#### `Executor`类型

我们在`task::executor`模块中创建新`Executor`：

```rust
// in src/task/mod.rs

pub mod executor;
```

```rust
// in src/task/executor.rs

use super::{Task, TaskId};
use alloc::{collections::{BTreeMap, VecDeque}, sync::Arc};
use core::task::Waker;
use crossbeam_queue::ArrayQueue;

pub struct Executor {
    task_queue: VecDeque<Task>,
    waiting_tasks: BTreeMap<TaskId, Task>,
    wake_queue: Arc<ArrayQueue<TaskId>>,
    waker_cache: BTreeMap<TaskId, Waker>,
}

impl Executor {
    pub fn new() -> Self {
        Executor {
            task_queue: VecDeque::new(),
            waiting_tasks: BTreeMap::new(),
            wake_queue: Arc::new(ArrayQueue::new(100)),
            waker_cache: BTreeMap::new(),
        }
    }
}
```

除了存储准备执行的任务的`task_queue`之外，该类型还具有个`waiting_tasks` map，一个 `wake_queue`和一个 `waker_cache`。这些字段具有以下目的：

- `waiting_tasks `map存储了返回`Poll::Pending`的任务。map由`TaskId`索引，以允许有效地继续执行特定任务。

- `wake_queue` 是一个task ID的 [`ArrayQueue`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html&usg=ALkJrhgYss2zDLVUgDk7pgbZR6MKjexiCw)，被包装入[`Arc`](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)，一个实现*引用计数*的类型。通过引用计数，可以在多个所有者之间共享所有权。它的工作原理是在堆上分配值并计算对其的引用数。当引用的数量达到零时，说明不再需要该值，可以将其释放。

  我们将`Arc`包装器用于`wake_queue`因为它将在执行者和唤醒者之间共享。这个想法是唤醒者将已唤醒任务的ID推入队列。执行程序位于队列的接收端，并将所有唤醒的任务从`waiting_tasks`映射移回到`task_queue`。使用固定大小队列而不是无限制队列，如[`SegQueue`](https://docs.rs/crossbeam-queue/0.2.1/crossbeam_queue/struct.SegQueue.html)的原因是，不能在其中进行动态内存分配的的中断处理程序将推送到此队列。

- 任务创建后，`waker_cache` map会对[`Waker`](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)进行缓存。这有两个原因：首先，它通过将同一唤醒程序重复用于同一任务的多次唤醒来提高性能，而不是每次都创建一个新的唤醒程序。其次，它确保不会在中断处理程序中释放`Arc`中的唤醒程序，因为这可能会导致死锁（下面有更多详细信息）。

要创建一个`Executor`，我们提供了一个简单的`new`功能。我们选择的容量为100 `wake_queue`，这在可预见的将来应该足够了。如果我们的系统在某个时候将有100个以上的并发任务，我们可以轻松地增加这个大小。

#### Spawn任务

至于`SimpleExecutor`，我们在`Executor`类型上提供了一个`spawn`方法，可将给定的任务添加到`task_queue`：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn spawn(&mut self, task: Task) {
        self.task_queue.push_back(task)
    }
}
```

由于此方法需要对executor的`&mut`引用，此方法executor启动后不可调用。如果要让任务本身在某个时候产生其他任务，我们可以将任务队列的类型更改为并发队列，例如[`SegQueue`](https://docs.rs/crossbeam-queue/0.2.1/crossbeam_queue/struct.SegQueue.html)并与任务共享对此队列的引用。

#### 运行任务

要执行`task_queue`中的所有任务，我们创建一个私有`run_ready_tasks`方法：

```rust
// in src/task/executor.rs

use core::task::{Context, Poll};

impl Executor {
    fn run_ready_tasks(&mut self) {
        while let Some(mut task) = self.task_queue.pop_front() {
            let task_id = task.id();
            if !self.waker_cache.contains_key(&task_id) {
                self.waker_cache.insert(task_id, self.create_waker(task_id));
            }
            let waker = self.waker_cache.get(&task_id).expect("should exist");
            let mut context = Context::from_waker(waker);
            match task.poll(&mut context) {
                Poll::Ready(()) => {
                    // task done -> remove cached waker
                    self.waker_cache.remove(&task_id);
                }
                Poll::Pending => {
                    if self.waiting_tasks.insert(task_id, task).is_some() {
                        panic!("task with same ID already in waiting_tasks");
                    }
                },
            }
        }
    }
}
```

此函数的基本思想类似于我们的`SimpleExecutor`：遍历`task_queue`中的所有任务，为每个任务创建一个唤醒器，然后对其进行poll。但是，我们没有将待处理的任务添加回`task_queue`的末尾，而是将它们存储在`waiting_tasks` map中，直到再次唤醒它们为止。唤醒器的创建是通过`create_waker`方法完成的，稍后将展示其实现。

为了避免每次poll时都创建唤醒程序的性能开销，我们使用`waker_cache` map在创建每个任务后为每个任务存储唤醒程序。为此，我们首先使用[`BTreeMap::contains_key`](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html#method.contains_key)方法检查任务是否存在缓存的唤醒器。如果没有，我们使用[`BTreeMap::insert`](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html#method.insert)方法创建它。此后，我们可以确定唤醒器存在，因此我们将该[`BTreeMap::get`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html&usg=ALkJrhjSeGuQgzKpGI3zFBfr7v7KHf7Zjg#method.get)方法与[`expect`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/core/option/enum.Option.html&usg=ALkJrhhrcr3tliazH2802ZCiz5sbhEl84A#method.expect)调用结合使用以获取对其的引用。

请注意，并非所有唤醒器实现都可以像这样重用唤醒器，但是我们的实现允许这样做。当任务完成时为了清理`waker_cache`，我们使用[`BTreeMap::remove`](https://doc.rust-lang.org/alloc/collections/btree_map/struct.BTreeMap.html#method.remove)方法从map中删除该任务的所有缓存的唤醒器。

#### Waker设计

唤醒程序的工作是将已唤醒任务的ID推送到`wake_queue`。我们通过创建一个新`TaskWaker`结构来实现此目的，该结构存储任务ID和对`wake_queue`的引用：

```rust
// in src/task/executor.rs

struct TaskWaker {
    task_id: TaskId,
    wake_queue: Arc<ArrayQueue<TaskId>>,
}
```

由于`wake_queue`的所有权是在executor和waker之间共享的，因此我们使用[`Arc`](https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html)包装器类型来实现共享的所有权引用计数。

唤醒操作的实现非常简单：

```rust
// in src/task/executor.rs

impl TaskWaker {
    fn wake_task(&self) {
        self.wake_queue.push(self.task_id).expect("wake_queue full");
    }
}
```

我们将`task_id`到压入引用的`wake_queue`。由于修改[`ArrayQueue`](https://docs.rs/crossbeam/0.7.3/crossbeam/queue/struct.ArrayQueue.html)类型仅需要一个共享引用，我们可以用`&self`而非`&mut self`来实现此方法。

##### `Wake` Trait

为了使用我们的`TaskWaker`类型来poll future，我们需要先将其转换为[`Waker`](https://doc.rust-lang.org/nightly/core/task/struct.Waker.html)实例。这是必需的，因为[`Future::poll`](https://doc.rust-lang.org/nightly/core/future/trait.Future.html#tymethod.poll)方法以[`Context`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/nightly/core/task/struct.Context.html&usg=ALkJrhienQeYRRGlCkB2M_ymB7qxtTE8Gg)实例作为参数，该实例只能从`Waker`类型构造。尽管我们可以通过提供[`RawWaker`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html&usg=ALkJrhjOuwz5oSe58stzvVL7q0DlBpW03w)类型的实现来做到这一点，但实现基于`Arc` 的`Waker` trait，然后使用标准库提供的[`From`](https://doc.rust-lang.org/nightly/core/convert/trait.From.html)实现来构造`Waker`，既简单又安全。

trait实现如下所示：

```rust
// in src/task/simple_executor.rs

use alloc::task::Wake;

impl Wake for TaskWaker {
    fn wake(self: Arc<Self>) {
        self.wake_task();
    }

    fn wake_by_ref(self: &Arc<Self>) {
        self.wake_task();
    }
}
```

该trait仍然不稳定，因此我们必须添加`#![feature(wake_trait)]`到`lib.rs`顶部才能使用它。由于唤醒程序通常在executor 和异步任务之间共享，因此trait方法要求`Self`实例被包装在[`Arc`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/alloc/sync/struct.Arc.html&usg=ALkJrhhytzcnWX7C38Bdbd_rI5EPjRlGig)类型中，该类型实现引用计数所有权。这意味着，我们对我们移动`TaskWaker`到一个`Arc`以调用他们。

`wake`和`wake_by_ref`方法之间的区别在于，后者仅需要引用`Arc`，而前者则拥有`Arc`的所有权，`Arc`因此通常需要增加引用计数。并非所有类型都支持按引用唤醒，因此实现该`wake_by_ref`方法是可选的，但是由于可以避免不必要的引用计数修改，因此可以提高性能。在我们的例子中，我们可以简单地将这两个特征方法都转发到我们的`wake_task`函数中，该函数仅需要一个共享的`&self`引用。

##### 创建Waker

由于`Waker`类型支持从所有所有包装在`Arc`内实现了`Wake` trait的值的[`From`](https://doc.rust-lang.org/nightly/core/convert/trait.From.html)转换，因此我们现在可以使用`TaskWaker`来实现`Executor::create_waker`方法：

```rust
// in src/task/executor.rs

impl Executor {
    fn create_waker(&self, task_id: TaskId) -> Waker {
        Waker::from(Arc::new(TaskWaker {
            task_id,
            wake_queue: self.wake_queue.clone(),
        }))
    }
}
```

我们使用传递的`task_id`和`wake_queue`的副本创建`TaskWaker`。由于`wake_queue`封装为`Arc`，因此`clone`只会增加值的引用计数，但仍指向相同的堆上的队列。我们也将`TaskWaker`储存在`Arc`中，因为`Waker::from`的实现需要它。然后，此函数负责为我们的`TaskWaker`类型构造[`RawWakerVTable`](https://doc.rust-lang.org/stable/core/task/struct.RawWakerVTable.html)和[`RawWaker`](https://doc.rust-lang.org/stable/core/task/struct.RawWaker.html)实例。如果您对它的详细工作方式感兴趣，请[在`alloc`crate中检查其实现](https://github.com/rust-lang/rust/blob/cdb50c6f2507319f29104a25765bfb79ad53395c/src/liballoc/task.rs#L58-L87)。

##### 处理唤醒

为了处理executor中的唤醒，我们添加了一个`wake_tasks`方法：

```rust
// in src/task/executor.rs

impl Executor {
    fn wake_tasks(&mut self) {
        while let Ok(task_id) = self.wake_queue.pop() {
            if let Some(task) = self.waiting_tasks.remove(&task_id) {
                self.task_queue.push_back(task);
            }
        }
    }
}
```

我们使用`while let`循环从`wake_queue`中弹出所有项目。对于每个弹出的任务ID，我们从`waiting_tasks`map中删除相应的任务，并将其添加到`task_queue`的后面。由于我们在检查是否需要使任务进入睡眠状态之前注册了唤醒器，因此即使任务不在`waiting_tasks`map中，也可能会发生任务唤醒。在这种情况下，我们只是忽略唤醒。

#### `run`方法

通过我们的唤醒器实现，我们最终可以为executor构造一个`run`方法：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn run(&mut self) -> ! {
        loop {
            self.wake_tasks();
            self.run_ready_tasks();
        }
    }
}
```

此方法仅在循环中调用`wake_tasks`和`run_ready_tasks`函数。从理论上讲，当`task_queue`和都`waiting_tasks`变为空时我们可以从函数返回，但这永远不会发生，因为我们`keyboard_task`永远都不会结束，因此简单地`loop`就足够了。由于函数永远不会返回，我们使用的`!`返回类型来告知编译器函数是[发散](https://doc.rust-lang.org/stable/rust-by-example/fn/diverging.html)的。

现在，我们可以更改`kernel_main`，使其使用新的`Executor`而不是`SimpleExecutor`：

```rust
// in src/main.rs

use blog_os::task::executor::Executor; // new

fn kernel_main(boot_info: &'static BootInfo) -> ! {
    // […] initialization routines, including init_heap, test_main

    let mut executor = Executor::new(); // new
    executor.spawn(Task::new(example_task()));
    executor.spawn(Task::new(keyboard::print_keypresses()));
    executor.run();
}
```

我们只需要更改导入和类型名称。由于我们的`run`函数被标记为发散，编译器知道它永远不会返回，因此我们不再需要在`kernel_main`函数末尾调用`hlt_loop`。

现在当我们使用`cargo xrun`运行内核时，我们看到键盘输入仍然有效：

![QEMU printing ".....H...e...l...l..o..... ...a..g..a....i...n...!"](https://os.phil-opp.com/async-await/qemu-keyboard-output-again.gif)

但是，QEMU的CPU使用率没有任何改善。原因是我们仍然使CPU始终保持忙碌状态。在任务被唤醒之前，我们不再轮询任务，但仍需要在忙循环中检查`wake_queue`和`task_queue`。要解决此问题，我们需要在没有其他工作要做时使CPU进入睡眠状态。

#### 空闲时睡眠

基本思想是在`task_queue`和`wake_queue`都为空时执行[`hlt`指令](https://en.wikipedia.org/wiki/HLT_(x86_instruction))。该指令使CPU进入睡眠状态，直到下一个中断到来。CPU立即在中断后再次变为活动状态，这确保了当中断处理程序向`wake_queue`推送时，我们仍然可以直接做出反应。

为了实现这一点，我们为executor创建了一个新`sleep_if_idle`方法，并从我们的`run`方法中调用它：

```rust
// in src/task/executor.rs

impl Executor {
    pub fn run(&mut self) -> ! {
        loop {
            self.wake_tasks();
            self.run_ready_tasks();
            self.sleep_if_idle();   // new
        }
    }

    fn sleep_if_idle(&self) {
        if self.wake_queue.is_empty() {
            x86_64::instructions::hlt();
        }
    }
}
```

由于我们在`run_ready_tasks`之后直接调用`sleep_if_idle`，它会一直循环直到`task_queue`变为空，因此我们只需要检查即可`wake_queue`。如果它也为空，则没有准备运行的任务，此时我们将通过[`x86_64`](https://docs.rs/x86_64/0.9.6/x86_64/index.html) crate 提供的[`instructions::hlt`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://docs.rs/x86_64/0.9.6/x86_64/instructions/fn.hlt.html&usg=ALkJrhjR_AOULnJlUUNudJ2x7x3rHcxz6Q)包装器函数执行`hlt`指令。

不幸的是，此实现中存在一个微妙的竞态条件。由于中断是异步的，并且可以随时发生，因此有可能在`is_empty`检查和`hlt`调用之间发生中断：

```rust
if self.wake_queue.is_empty() {
    /// <--- interrupt can happen here
    x86_64::instructions::hlt();
}
```

如果中断向`wake_queue`推送，即使现在有一个准备好的任务，我们也会使CPU进入睡眠状态。在最坏的情况下，这可能会延迟键盘中断的处理，直到下一次按键或下一个定时器中断为止。那么我们如何预防呢？

答案是在检查之前禁用CPU上的中断，并与`hlt`指令一起自动启用它们。这样，两个指令之间的所有中断都会被延迟到指令之后，从而不会丢失任何唤醒。要实现此方法，我们可以使用[`x86_64`](https://docs.rs/x86_64/0.9.6/x86_64/index.html) crate提供的函数[`enable_interrupts_and_hlt`](https://docs.rs/x86_64/0.9.6/x86_64/instructions/interrupts/fn.enable_interrupts_and_hlt.html)。此函数仅在0.9.6版以后可用，因此您可能需要更新`x86_64`依赖项才能使用它。

我们`sleep_if_idle`函数的更新实现如下所示：

```rust
// in src/task/executor.rs

impl Executor {
    fn sleep_if_idle(&self) {
        use x86_64::instructions::interrupts::{self, enable_interrupts_and_hlt};

        // fast path
        if !self.wake_queue.is_empty() {
            return;
        }

        interrupts::disable();
        if self.wake_queue.is_empty() {
            enable_interrupts_and_hlt();
        } else {
            interrupts::enable();
        }
    }
}
```

为了避免不必要地禁用中断，如果`wake_queue`不为空，我们会尽早返回。否则，我们禁用中断并再次检查`wake_queue`。如果它仍然为空，则使用[`enable_interrupts_and_hlt`](https://docs.rs/x86_64/0.9.6/x86_64/instructions/interrupts/fn.enable_interrupts_and_hlt.html)函数启用中断，并使CPU以单个原子操作的方式进入睡眠状态。如果队列不再为空，则意味着中断在第一次和第二检查之间唤醒了一个任务。在这种情况下，我们将再次启用中断并直接继续执行而不执行`hlt`。

现在，我们的执行器可以在无事可做的情况下正确地使CPU进入睡眠状态。我们可以看到，`cargo xrun`再次运行内核时，QEMU进程的CPU使用率要低得多。

#### 可能的扩展

我们的executor现在可以高效地运行任务。它利用唤醒程序来避免不断poll等待中的任务，并在当前无工作要做时使CPU进入睡眠状态。但是，我们的executor仍然是非常基础的，并且有许多可能的方式来扩展其功能：

- **调度：**我们目前使用的[`VecDeque`](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://doc.rust-lang.org/stable/alloc/collections/vec_deque/struct.VecDeque.html&usg=ALkJrhiXuJElXDamcn6zQ9-511Hx54ABqw)类型来为我们的`task_queue`实现一个*先进先出*（FIFO）的策略，这也经常被称为*轮转*调度。此策略可能并不对所有类型的工作负载来说都是最有效的。例如，优先考虑对延迟至关重要的任务或执行大量I / O的任务可能会更好。可以阅读 [*Operating Systems: Three Easy Pieces*](http://pages.cs.wisc.edu/~remzi/OSTEP/) 中的 [scheduling 一章](http://pages.cs.wisc.edu/~remzi/OSTEP/cpu-sched.pdf)或者Wikipedia上的[调度页面]([Wikipedia article on scheduling](https://en.wikipedia.org/wiki/Scheduling_(computing))来获取更多信息。
- **任务Spawning**：我们的`Executor::spawn`方法当前需要`&mut self`，因此启动`run`方法后其不再可用。为了解决这个问题，我们可以另外创建一个`Spawner`类型，该类型与执行程序共享某种队列，并允许从任务内部创建新的任务。这个队列可以是`task_queue` 或是另一个executor在其运行的循环中不断检查的单独的队列。
- **使用线程**：我们尚不支持线程，但我们将在下一篇文章中添加它。这将使得可以在不同的线程中启动执行程序的多个实例。这种方法的优点是可以减少长时间运行的任务带来的延迟，因为其他任务可以同时运行。这种方法还允许它利用多个CPU内核。
- **负载均衡**：添加线程支持时，如何在执行程序之间分配任务以确保利用所有CPU内核变得很重要。一种常见的技术是[*工作窃取*](https://translate.googleusercontent.com/translate_c?depth=1&rurl=translate.google.com.hk&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Work_stealing&usg=ALkJrhhJLg327BIIM2T-3Np3Rid7wrnD5Q)。

## 总结

我们通过介绍**多任务**和区分*抢占式*多任务（强制性地定期中断正在运行的任务）和*协作*式多任务（使任务一直运行到自愿放弃对CPU的控制）之间的区别来开始本文。

然后，我们探讨了Rust对**async/await**的支持如何提供协作式多任务处理的语言级实现。Rust使用了基于polling的`Future` trait，其为抽象了的异步任务。使用async/await，可以像编写普通同步代码一样使用Future。不同之处在于异步函数还是会返回`Future` ，需要在某个时候将其添加到执行程序才能运行它。

在后台，编译器将异步/等待代码转换为*状态机*，每个`.await`操作对应一个可能的暂停点。通过利用其对程序的了解，编译器每个暂停点只会保存最小状态，从而使每个任务的内存消耗非常小。一个挑战是生成的状态机可能包含*自引用*结构，例如，当异步函数的局部变量相互引用时。为了防止指针失效，Rust使用该`Pin`类型来确保在第一次对其进行poll之后，就不能再将其在内存中移动。

对于我们的**实现**，我们首先创建了一个非常基本的执行程序，该执行程序不断使用循环poll所有产生的任务，而`Waker`根本什么都不做。然后，我们通过实现异步键盘任务来展示唤醒通知的优势。该任务使用`crossbeam` crate中`ArrayQueue`提供的无锁类型定义静态变量`SCANCODE_QUEUE`。现在，键盘中断处理程序不再直接处理按键，而是将所有接收到的扫描代码放入队列中，然后唤醒已注册的`Waker`以发出新输入可用的信号。在接收端，我们创建了一个`ScancodeStream`类型以提供`Future`对队列中下一个扫描代码的解析。这使得使用async/await的解释和打印队列中扫描代码的异步`print_keypresses`成为可能。 

为了利用键盘任务的唤醒通知，我们创建了一种新`Executor`类型，可以区分准备好了的任务和等待任务。使用`Arc`内的的`wake_queue`，我们实现了一种`TaskWaker`类型，该类型将唤醒通知直接发送到执行程序，然后执行程序可以将相应的任务再次标记为就绪。为了在无任务可运行时节省电量，我们增加了使用`hlt`指令使CPU进入睡眠状态的支持。最后，我们讨论了executor的一些潜在扩展，例如，用于提供多核支持。

## 下一步是什么？

使用async/await，我们现在对内核中的协作多任务有了基本的支持。尽管协作式多任务处理非常有效，但是当单个任务保持运行太长时间从而导致其他任务无法运行时，则会导致延迟问题。因此，在我们的内核中添加对抢占式多任务的支持也是有意义的。

在下一篇文章中，我们将介绍*线程*作为抢占式多任务处理的最常见形式。除了解决任务长期运行的问题之外，线程还将为让我们可以准备好利用多个CPU核心并在将来运行不受信任的用户程序。



