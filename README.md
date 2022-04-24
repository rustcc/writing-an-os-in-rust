# writing-an-os-in-rust

《编写 Rust 语言的操作系统》简体中文翻译

## 目录

### 正文

| 序号 | 标题             | 链接                                              | 源文件                                   | 状态    | 长度    |
| ---- | ---------------- | ------------------------------------------------- | ---------------------------------------- | ------- | ------- |
| 一   | 独立式可执行程序 | [知乎专栏](https://zhuanlan.zhihu.com/p/53064186) | [点我](./01-freestanding-rust-binary.md) | Done    | 11 千字 |
| 二   | 最小化内核       | [知乎专栏](https://zhuanlan.zhihu.com/p/56433770) | [点我](./02-minimal-rust-kernel.md)      | Done    | 19 千字 |
| 三   | VGA 字符模式     | [知乎专栏](https://zhuanlan.zhihu.com/p/53745617) | [点我](./03-vga-text-mode.md)            | Done    | 21 千字 |
| 四   | 内核测试         | [知乎专栏](https://zhuanlan.zhihu.com/p/90758552) | [点我](./04-testing.md)                  | Done    | 27 千字 |
| 五   | CPU 异常         | 待添加                                            | [点我](./05-cpu-exceptions.md)           | Pending | -       |
| 六   | 双重异常         | 待添加                                            | [点我](./06-double-fault-exceptions.md)  | Pending | -       |
| 七   | 硬件中断         | 待添加                                            | [点我](./07-hardware-interrupts.md)      | Done | 21 千字 |
| 八   | 内存分页简介     | 待添加                                            | [点我](./08-introduction-to-paging.md)   | Done | 16 千字 |
| 九   | 内存分页实现     | 待添加                                            | [点我](./09-paging-implementation.md)     | Done | 28 千字 |
| 十   | 堆分配           | 待添加                                            | [点我](./10-heap-allocation.md)          | Done | 20 千字 |
| 十一 | 内存分配器设计   | 待添加                                            | [点我](./11-allocator-designs.md)        | Done | 35 千字 |
| 十二 | Async/Await   | 待添加                                            | [点我](./12-async-await.md)        | Done | 51 千字 |

### 附录

| 序号   | 标题             | 链接                                              | 源文件                                   | 状态    | 长度   |
| ------ | ---------------- | ------------------------------------------------- | ---------------------------------------- | ------- | ------ |
| 附录一 | 链接器参数       | [知乎专栏](https://zhuanlan.zhihu.com/p/69393545) | [点我](./appendix-a-linker-arguments.md) | Done    | 6 千字 |
| 附录二 | 禁用红区         | [知乎专栏](https://zhuanlan.zhihu.com/p/53240133) | [点我](./appendix-b-red-zone.md)         | Done    | 2 千字 |
| 附录三 | 禁用 SIMD        | [知乎专栏](https://zhuanlan.zhihu.com/p/53350970) | [点我](./appendix-c-disable-simd.md)     | Done    | 2 千字 |
| 附录四 | 在安卓系统上构建 | 待添加                                            | 待添加                                   | Pending | -      |

### 译名表

[点我](./translation-table.md)

## 译者

- 洛佳 (@luojia65)，华中科技大学
- 龙方淞 (@longfangsong)，上海大学开源社区
- readlnh (@readlnh)
- 尚卓燃 (@psiace)，华中农业大学
