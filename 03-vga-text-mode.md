> 原文：https://os.phil-opp.com/vga-text-mode/
>
> 原作者：@phil-opp
>
> 译者：洛佳  华中科技大学

# 使用Rust创造操作系统（三）：VGA字符模式

**VGA字符模式**（[VGA text mode](https://en.wikipedia.org/wiki/VGA-compatible_text_mode)）是打印字符到屏幕的一种简单方式。在这篇文章中，为了包装这个模式为一个安全而简单的接口，我们包装unsafe代码到独立的模块。我们还将实现对Rust语言**格式化宏**（[formatting macros](https://doc.rust-lang.org/std/fmt/#related-macros)）的支持。

## VGA字符缓冲区

为了在VGA字符模式向屏幕打印字符，我们必须将它写入硬件提供的**VGA字符缓冲区**（VGA text buffer）。通常状况下，VGA字符缓冲区是一个25行、80列的二维数组，它的内容将被实时渲染到屏幕。这个数组的元素被称作**字符单元**（character cell），它使用下面的格式描述一个屏幕上的字符：

| Bit(s)    | Value |
|-----|----------------|
| 0-7   | ASCII code point |
| 8-11  | Foreground color |
| 12-14 | Background color |
| 15    | Blink |

其中，**前景色**（foreground color）和**背景色**（background color）取值范围如下：

| Number  | Color  | Number + Bright Bit  | Bright Color |
|-----|----------|------|--------|
| 0x0  | Black  | 0x8  | Dark Gray |
| 0x1  | Blue  | 0x9  | Light Blue |
| 0x2  | Green  | 0xa  | Light Green |
| 0x3  | Cyan  | 0xb  | Light Cyan |
| 0x4  | Red  | 0xc  | Light Red |
| 0x5  | Magenta  | 0xd  | Pink |
| 0x6  | Brown  | 0xe  | Yellow |
| 0x7  | Light Gray  | 0xf  | White |

每个颜色的第四位称为**加亮位**（bright bit）。

要修改VGA字符缓冲区，我们可以通过**存储器映射输入输出**（[memory-mapped I/O](https://en.wikipedia.org/wiki/Memory-mapped_I/O)）的方式，读取或写入地址`0xb8000`；这意味着，我们可以像操作普通的内存区域一样操作这个地址。

需要主页的是，一些硬件虽然映射到存储器，却可能不会完全支持所有的内存操作：可能会有一些设备支持按`u8`字节读取，却在读取`u64`时返回无效的数据。幸运的是，字符缓冲区都[支持标准的读写操作](https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vgamem.htm#manip)，所以我们不需要用特殊的标准对待它。

## 包装到Rust模块

既然我们已经知道VGA文字缓冲区如何工作，也是时候创建一个Rust模块来处理文字打印了。我们输入这样的代码：

```rust
// in src/main.rs
mod vga_buffer;
```

~~这行代码定义了一个Rust模块，它的内容可以保存在`src/vga_buffer.rs`或`src/vga_buffer/mod.rs`中；后一种保存方式支持使用**子模块**（submodule），而前者不支持。~~

这行代码定义了一个Rust模块，它的内容应当保存在`src/vga_buffer.rs`文件中。使用**2018版次**（2018 edition）的Rust时，我们可以把模块的**子模块**（submodule）文件直接保存到`src/vga_buffer/`文件夹下，与`vga_buffer.rs`文件共存，而无需创建一个`mod.rs`文件。

我们的模块暂时不需要添加子模块，所以我们将它创建为`src/vga_buffer.rs`文件。除非另有说明，本文中的代码都保存到这个文件中。

## 颜色

首先，我们使用Rust的**枚举**（enum）表示一种颜色：

```rust
// in src/vga_buffer.rs

#[allow(dead_code)]
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(u8)]
pub enum Color {
    Black = 0,
    Blue = 1,
    Green = 2,
    Cyan = 3,
    Red = 4,
    Magenta = 5,
    Brown = 6,
    LightGray = 7,
    DarkGray = 8,
    LightBlue = 9,
    LightGreen = 10,
    LightCyan = 11,
    LightRed = 12,
    Pink = 13,
    Yellow = 14,
    White = 15,
}
```

我们使用**类似于C语言的枚举**（C-like enum），为每个颜色明确指定一个数字。在这里，每个用`repr(u8)`注记标注的枚举类型，都会以一个`u8`的形式存储——事实上4个二进制位就足够了，但Rust语言并不提供`u4`类型。

通常来说，编译器会对每个未使用的变量发出**警告**（warning）；使用`#[allow(dead_code)]`，我们可以对`Color`枚举类型禁用这个警告。

我们还**生成**（[derive](http://rustbyexample.com/trait/derive.html)）了 [`Copy`](https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html)、[`Clone`](https://doc.rust-lang.org/nightly/core/clone/trait.Clone.html)、[`Debug`](https://doc.rust-lang.org/nightly/core/fmt/trait.Debug.html)、[`PartialEq`](https://doc.rust-lang.org/nightly/core/cmp/trait.PartialEq.html)和[`Eq`](https://doc.rust-lang.org/nightly/core/cmp/trait.Eq.html) 这几个trait：这让我们的类型遵循**复制语义**（[copy semantics](https://doc.rust-lang.org/book/first-edition/ownership.html#copy-types)），也让它可以被比较、被调试打印。

为了描述包含前景色和背景色的、完整的**颜色代码**（color code），我们基于`u8`创建一个新类型：

```rust
// in src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
struct ColorCode(u8);

impl ColorCode {
    fn new(foreground: Color, background: Color) -> ColorCode {
        ColorCode((background as u8) << 4 | (foreground as u8))
    }
}
```

这里，`ColorCode`类型包装了一个完整的颜色代码字节，它包含前景色和背景色信息。和`Color`类型类似，我们为它生成`Copy`和`Debug`等一系列trait。

## 字符缓冲区

现在，我们可以添加更多的结构体，来描述屏幕上的字符和整个字符缓冲区：

```rust
// in src/vga_buffer.rs

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
#[repr(C)]
struct ScreenChar {
    ascii_character: u8,
    color_code: ColorCode,
}

const BUFFER_HEIGHT: usize = 25;
const BUFFER_WIDTH: usize = 80;

struct Buffer {
    chars: [[ScreenChar; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

在内存布局层面，Rust并不保证按顺序布局成员变量。因此，我们需要使用`#[repr(C)]`标记结构体；这将按C语言约定的顺序布局它的成员变量，让我们能正确地映射内存片段。

为了输出字符到屏幕，我们来创建一个`Writer`类型：

```rust
// in src/vga_buffer.rs

pub struct Writer {
    column_position: usize,
    color_code: ColorCode,
    buffer: &'static mut Buffer,
}
```

我们将让这个`Writer`类型将字符写入屏幕的最后一行，并在一行写满或收到换行符`\n`的时候，将所有的字符向上位移一行。`column_position`变量将跟踪光标在最后一行的位置。当前字符的前景和背景色将由`color_code`变量指定；另外，我们存入一个VGA字符缓冲区的可变借用到`buffer`变量中。需要注意的是，这里我们对借用使用**显式生命周期**（[explicit lifetime](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#lifetime-annotation-syntax)），告诉编译器这个借用在何时有效：我们使用**`'static`生命周期**（['static lifetime](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html#the-static-lifetime)），意味着这个借用应该在整个程序的运行期间有效；这对一个全局有效的VGA字符缓冲区来说，是非常合理的。

## 打印字符

现在我们可以使用`Writer`类型来更改缓冲区内的字符了。首先，为了写入一个ASCII码字节，我们创建这样的函数：

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                if self.column_position >= BUFFER_WIDTH {
                    self.new_line();
                }

                let row = BUFFER_HEIGHT - 1;
                let col = self.column_position;

                let color_code = self.color_code;
                self.buffer.chars[row][col] = ScreenChar {
                    ascii_character: byte,
                    color_code,
                };
                self.column_position += 1;
            }
        }
    }

    fn new_line(&mut self) {/* TODO */}
}
```

如果这个字节是一个**换行符**（[line feed](https://en.wikipedia.org/wiki/Newline)）字节`\n`，我们的`Writer`不应该打印新字符，相反，它将调用我们稍后会实现的`new_line`方法；其它的字节应该将在`match`语句的第二个分支中被打印到屏幕上。

当打印字节时，`Writer`将检查当前行是否已满。如果已满，它将首先调用`new_line`方法来将这一行字向上提升，再将一个新的`ScreenChar`写入到缓冲区，最终将当前的光标位置前进一位。

要打印整个字符串，我们把它转换为字节并依次输出：

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_string(&mut self, s: &str) {
        for byte in s.bytes() {
            match byte {
                // 可以是能打印的ASCII码字节，也可以是换行符
                0x20...0x7e | b'\n' => self.write_byte(byte),
                // 不包含在上述范围之内的字节
                _ => self.write_byte(0xfe),
            }

        }
    }
}
```

VGA字符缓冲区只支持ASCII码字节和**代码页437**（[Code page 437](https://en.wikipedia.org/wiki/Code_page_437)）定义的字节。Rust语言的字符串默认编码为[UTF-8](http://www.fileformat.info/info/unicode/utf8.htm)，也因此可能包含一些VGA字符缓冲区不支持的字节：我们使用`match`语句，来区别可打印的ASCII码或换行字节，和其它不可打印的字节。对每个不可打印的字节，我们打印一个`■`符号；这个符号在VGA硬件中被编码为十六进制的`0xfe`。

