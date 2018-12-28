>原文：https://os.phil-opp.com/red-zone/
>
>原作者：@phil-opp
>
>译者：洛佳  华中科技大学

# 使用Rust创造操作系统（附录一）：禁用红区

**红区**（redzone）是System V ABI提供的一种优化的产物，它允许函数无需调整**栈指针**（stack pointer），便能临时使用其**栈帧**（stack frame）下方的128个字节：

![](https://os.phil-opp.com/red-zone/red-zone.svg)

这张图片展示了一个有`n`个**局部变量**（local variable）的函数的栈帧。在进入函数时，两个**指令**（instruction）将栈指针调节到合适的位置，以便为**返回地址**（return address）和局部变量提供内存空间。

红区被定义为调整过的栈指针下方128个字节的区域——函数将会使用这个区域，来存放一些无需跨越函数调用的临时数据。因此，在一些情况下，比如在小的**叶函数**（leaf function）[1]中，我们可以优化掉调整栈指针所需的两个指令。

然而，当**异常**（exception）或**硬件中断**（hardware interrupt）发生时，这种优化却可能产生严重的问题。为了理解这一点，我们假设，当某个函数正在使用红区时，发生了一个异常：

![](https://os.phil-opp.com/red-zone/red-zone-overwrite.svg)

当异常发生时，CPU和**异常处理器**（exception handler）会向下**覆写**（overwrite）红区内的数据；这些曾经存在但被覆写的红区数据，却可能仍然将被被中断的函数使用。这之后，当我们从异常处理器中返回时，这个函数不再像它的定义一样正常工作。这个特性可能会产生许多奇怪而隐蔽的bug，甚至需要几周时间才能找到它的成因。

为了避免这样的bug发生，我们编写异常处理器时，常常从一开始便禁用红区。因此，要实现这一点，我们可以在编译目标的**配置文件**（configuration file）中，添加一行`"disable-redzone": true`。

---

[1] **叶函数**（leaf function）指的是不调用其它函数的函数；可以理解为，是函数调用图的叶子节点。特别地，**尾递归函数**（tail recursive function）的尾部可以看作是叶函数。