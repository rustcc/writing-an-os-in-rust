> 原文：https://os.phil-opp.com/testing/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust编写操作系统（四）：内核测试

这篇文章中，我们将使用Rust内置的测试框架，探索单元测试`no_std`程序的方式。我们将调整我们的代码，以便使用`cargo test`命令；我们还将为上篇文章中编写的VGA缓冲区模块添加一些基础的单元测试。

# `no_std`下的单元测试

Rust提供一个[内置的测试框架](https://doc.rust-lang.org/book/second-edition/ch11-00-testing.html)，允许在不做额外操作的前提下运行单元测试——我们只需要创建一个函数，为它添加`#[test]`属性，再通过**断言**（assertions）检查一些结果。准备了这些之后，`cargo test`便能自动寻找包中所有的测试函数，并把它们挨个儿跑一遍。


