> 原文：https://os.phil-opp.com/testing/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust编写操作系统（四）：内核测试

这篇文章中，我们将使用Rust内置的测试框架，探索单元测试`no_std`程序的方式。

## `no_std`下的单元测试

Rust提供一个[内置的测试框架](https://doc.rust-lang.org/book/second-edition/ch11-00-testing.html)，允许在不做额外操作的前提下运行单元测试——我们只需要创建一个函数，为它添加`#[test]`属性，再通过**断言**（assertions）检查一些结果。准备了这些之后，`cargo test`便能自动寻找包中所有的测试函数，并把它们挨个儿跑一遍。

### 自定义测试框架

## 退出QEMU

### IO端口

### 使用“退出设备”

### 设定成功返回值

## 输出到控制台

### 串行接口

### QEMU参数

### 打印panic错误信息

### 隐藏QEMU

### 超时

## 测试VGA缓冲区

## 集成测试

### 创建库

### 完成集成测试

### 展望

## 测试panic处理器

### 去除harness

### 实现测试

### 检查PanicInfo

## 小结

## 下篇预告

