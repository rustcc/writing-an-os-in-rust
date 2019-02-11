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

我们可以亲自试一试已经编写的代码。为了这样做，我们可以临时编写一个函数：

```rust
// in src/vga_buffer.rs

pub fn print_something() {
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_string("ello ");
    writer.write_string("Wörld!");
}
```

这个函数首先创建一个指向`0xb8000`地址VGA缓冲区的`Writer`。实现这一点，我们需要编写的代码可能看起来有点奇怪：首先，我们把整数`0xb8000`强制转换为一个可变的**裸指针**（[raw pointer](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer)）；之后，通过运算符`*`，我们将这个裸指针解引用；最后，我们再通过`&mut`，再次获得它的可变借用。这些转换需要**`unsafe`语句块**（[unsafe block](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html)），因为编译器并不能保证这个裸指针是有效的。

然后它将字节 `b'H'` 写入缓冲区内. 前缀 `b`创建了一个字节常量（[byte literal](https://doc.rust-lang.org/reference/tokens.html#byte-literals)），表示单个ASCII码字符；通过尝试写入 `"ello "` 和 `"Wörld!"`，我们可以测试 `write_string` 方法和其后对无法打印字符的处理逻辑。当我们尝试在`src/main.rs`中的`_start`函数内调用`vga_buffer::print_something`时，黄色的`Hello W■■rld!`字符串将会被打印在屏幕的左下角：

![QEMU output with a yellow Hello W■■rld! in the lower left corner](https://os.phil-opp.com/vga-text-mode/vga-hello.png)

需要注意的是，`ö`字符被打印为两个`■`字符。这是因为在[UTF-8](http://www.fileformat.info/info/unicode/utf8.htm)编码下，字符`ö`是由两个字节表述的——而这两个字节并不处在可打印的ASCII码字节范围之内。事实上，这是UTF-8编码的基本特点之一：**如果一个字符占用多个字节，那么每个组成它的独立字节都不是有效的ASCII码字节**（the individual bytes of multi-byte values are never valid ASCII）。

## Volatile

我们刚才看到，自己想要输出的信息被正确地打印到屏幕上。然而，未来Rust编译器更暴力的优化可能让这段代码不按预期工作。

产生问题的原因在于，我们只向`Buffer`写入，却不再从它读出数据。此时，编译器不知道我们事实上已经在操作VGA缓冲区内存，而不是在操作普通的RAM——因此也不知道产生的**副效应**（side effect），即会有几个字符显示在屏幕上。这时，编译器也许会认为这些写入操作都没有必要，甚至会选择忽略这些操作！所以，为了避免这些并不正确的优化，这些写入操作应当被指定为[volatile](https://en.wikipedia.org/wiki/Volatile_(computer_programming))操作。这将告诉编译器，这些写入可能会产生副效应，不应该被优化掉。

为了在我们的VGA缓冲区中使用volatile写入操作，我们使用[volatile](https://docs.rs/volatile)库。这个**包**（crate）提供一个名为`Volatile`的**包装类型**（wrapping type）和它的`read`、`write`方法；这些方法包装了`core::ptr`内的[read_volatile](https://doc.rust-lang.org/nightly/core/ptr/fn.read_volatile.html)和[write_volatile](https://doc.rust-lang.org/nightly/core/ptr/fn.write_volatile.html) 函数，从而保证读操作或写操作不会被编译器优化。

要添加`volatile`包为项目的**依赖项**（dependency），我们可以在`Cargo.toml`文件的`dependencies`中添加下面的代码：

```toml
# in Cargo.toml

[dependencies]
volatile = "0.2.3"
```

`0.2.3`表示一个**语义版本号**（[semantic version number](http://semver.org/)），在cargo文档的[《指定依赖项》章节](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html)可以找到与它相关的使用指南。

现在，我们使用它来完成VGA缓冲区的volatile写入操作。我们将`Buffer`类型的定义修改为下列代码：

```rust
// in src/vga_buffer.rs

use volatile::Volatile;

struct Buffer {
    chars: [[Volatile<ScreenChar>; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```

在这里，我们不使用`ScreenChar`，而选择使用`Volatile<ScreenChar>`——在这里，`Volatile`类型是一个**泛型**（[generic](https://doc.rust-lang.org/book/ch10-01-syntax.html)），可以包装几乎所有的类型——这确保了我们不会通过普通的写入操作，意外地向它写入数据；我们转而使用提供的`write`方法。

这意味着，我们必须要修改我们的`Writer::write_byte`方法：

```rust
// in src/vga_buffer.rs

impl Writer {
    pub fn write_byte(&mut self, byte: u8) {
        match byte {
            b'\n' => self.new_line(),
            byte => {
                ...

                self.buffer.chars[row][col].write(ScreenChar {
                    ascii_character: byte,
                    color_code: color_code,
                });
                ...
            }
        }
    }
    ...
}
```

正如代码所示，我们不再使用普通的`=`赋值，而使用了`write`方法：这能确保编译器不再优化这个写入操作。

## 格式化宏

支持Rust提供的**格式化宏**（formatting macros）也是一个相当棒的主意。通过这种途径，我们可以轻松地打印不同类型的变量，如整数或浮点数。为了支持它们，我们需要实现[`core::fmt::Write`](https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html) trait；要实现它，唯一需要提供的方法是`write_str`，它和我们先前编写的`write_string`方法差别不大，只是返回值类型变成了`fmt::Result`：

```
// in src/vga_buffer.rs

use core::fmt;

impl fmt::Write for Writer {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        self.write_string(s);
        Ok(())
    }
}
```

The `Ok(())` is just a `Ok` Result containing the `()` type.

Now we can use Rust's built-in `write!`/`writeln!` formatting macros:

```
// in src/vga_buffer.rs

pub fn print_something() {
    use core::fmt::Write;
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_string("ello! ");
    write!(writer, "The numbers are {} and {}", 42, 1.0/3.0).unwrap();
}
```

Now you should see a `Hello! The numbers are 42 and 0.3333333333333333` at the bottom of the screen. The `write!` call returns a `Result` which causes a warning if not used, so we call the [`unwrap`](https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap) function on it, which panics if an error occurs. This isn't a problem in our case, since writes to the VGA buffer never fail.

### Newlines

Right now, we just ignore newlines and characters that don't fit into  the line anymore. Instead we want to move every character one line up  (the top line gets deleted) and start at the beginning of the last line  again. To do this, we add an implementation for the `new_line` method of `Writer`:

```
// in src/vga_buffer.rs

impl Writer {
    fn new_line(&mut self) {
        for row in 1..BUFFER_HEIGHT {
            for col in 0..BUFFER_WIDTH {
                let character = self.buffer.chars[row][col].read();
                self.buffer.chars[row - 1][col].write(character);
            }
        }
        self.clear_row(BUFFER_HEIGHT - 1);
        self.column_position = 0;
    }

    fn clear_row(&mut self, row: usize) {/* TODO */}
}
```

We iterate over all screen characters and move each character one row up. Note that the range notation (`..`) is exclusive the upper bound. We also omit the 0th row (the first range starts at `1`) because it's the row that is shifted off screen.

To finish the newline code, we add the `clear_row` method:

```
// in src/vga_buffer.rs

impl Writer {
    fn clear_row(&mut self, row: usize) {
        let blank = ScreenChar {
            ascii_character: b' ',
            color_code: self.color_code,
        };
        for col in 0..BUFFER_WIDTH {
            self.buffer.chars[row][col].write(blank);
        }
    }
}
```

This method clears a row by overwriting all of its characters with a space character.

## A Global Interface

To provide a global writer that can used as an interface from other modules without carrying a `Writer` instance around, we try to create a static `WRITER`:

```
// in src/vga_buffer.rs

pub static WRITER: Writer = Writer {
    column_position: 0,
    color_code: ColorCode::new(Color::Yellow, Color::Black),
    buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
};
```

However, if we try to compile it now, the following errors occur:

```
error[E0015]: calls in statics are limited to constant functions, tuple structs and tuple variants
 --> src/vga_buffer.rs:7:17
  |
7 |     color_code: ColorCode::new(Color::Yellow, Color::Black),
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error[E0396]: raw pointers cannot be dereferenced in statics
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ dereference of raw pointer in constant

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:22
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values

error[E0017]: references in statics may only refer to immutable values
 --> src/vga_buffer.rs:8:13
  |
8 |     buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
  |             ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ statics require immutable values
```

To understand what's happening here, we need to know that statics are  initialized at compile time, in contrast to normal variables that are  initialized at run time. The component of the Rust compiler that  evaluates such initialization expressions is called the “[const evaluator](https://rust-lang.github.io/rustc-guide/const-eval.html)”. Its functionality is still limited, but there is ongoing work to expand it, for example in the “[Allow panicking in constants](https://github.com/rust-lang/rfcs/pull/2345)” RFC.

The issue about `ColorCode::new` would be solvable by using [`const` functions](https://doc.rust-lang.org/unstable-book/language-features/const-fn.html),  but the fundamental problem here is that Rust's const evaluator is not  able to convert raw pointers to references at compile time. Maybe it  will work someday, but until then, we have to find another solution.

### Lazy Statics

The one-time initialization of statics with non-const functions is a  common problem in Rust. Fortunately, there already exists a good  solution in a crate named [lazy_static](https://docs.rs/lazy_static/1.0.1/lazy_static/). This crate provides a `lazy_static!` macro that defines a lazily initialized `static`. Instead of computing its value at compile time, the `static`  laziliy initializes itself when it's accessed the first time. Thus, the  initialization happens at runtime so that arbitrarily complex  initialization code is possible.

Let's add the `lazy_static` crate to our project:

```
# in Cargo.toml

[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]
```

We need the `spin_no_std` feature, since we don't link the standard library.

With `lazy_static`, we can define our static `WRITER` without problems:

```
// in src/vga_buffer.rs

use lazy_static::lazy_static;

lazy_static! {
    pub static ref WRITER: Writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
}
```

However, this `WRITER` is pretty useless since it is immutable. This means that we can't write anything to it (since all the write methods take `&mut self`). One possible solution would be to use a [mutable static](https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable). But then every read and write to it would be unsafe since it could easily introduce data races and other bad things. Using `static mut` is highly discouraged, there were even proposals to [remove it](https://internals.rust-lang.org/t/pre-rfc-remove-static-mut/1437). But what are the alternatives? We could try to use a immutable static with a cell type like [RefCell](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html#keeping-track-of-borrows-at-runtime-with-refcellt) or even [UnsafeCell](https://doc.rust-lang.org/nightly/core/cell/struct.UnsafeCell.html) that provides [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html). But these types aren't [Sync](https://doc.rust-lang.org/nightly/core/marker/trait.Sync.html) (with good reason), so we can't use them in statics.

### Spinlocks

To get synchronized interior mutability, users of the standard library can use [Mutex](https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.html).  It provides mutual exclusion by blocking threads when the resource is  already locked. But our basic kernel does not have any blocking support  or even a concept of threads, so we can't use it either. However there  is a really basic kind of mutex in computer science that requires no  operating system features: the [spinlock](https://en.wikipedia.org/wiki/Spinlock).  Instead of blocking, the threads simply try to lock it again and again  in a tight loop and thus burn CPU time until the mutex is free again.

To use a spinning mutex, we can add the [spin crate](https://crates.io/crates/spin) as a dependency:

```
# in Cargo.toml
[dependencies]
spin = "0.4.9"
```

Then we can use the spinning Mutex to add safe [interior mutability](https://doc.rust-lang.org/book/ch15-05-interior-mutability.html) to our static `WRITER`:

```
// in src/vga_buffer.rs

use spin::Mutex;
...
lazy_static! {
    pub static ref WRITER: Mutex<Writer> = Mutex::new(Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    });
}
```

Now we can delete the `print_something` function and print directly from our `_start` function:

```
// in src/main.rs
#[no_mangle]
pub extern "C" fn _start() -> ! {
    use core::fmt::Write;
    vga_buffer::WRITER.lock().write_str("Hello again").unwrap();
    write!(vga_buffer::WRITER.lock(), ", some numbers: {} {}", 42, 1.337).unwrap();

    loop {}
}
```

We need to import the `fmt::Write` trait in order to be able to use its functions.

### Safety

Note that we only have a single unsafe block in our code, which is needed to create a `Buffer` reference pointing to `0xb8000`.  Afterwards, all operations are safe. Rust uses bounds checking for  array accesses by default, so we can't accidentally write outside the  buffer. Thus, we encoded the required conditions in the type system and  are able to provide a safe interface to the outside.

### A println Macro

Now that we have a global writer, we can add a `println` macro that can be used from anywhere in the codebase. Rust's [macro syntax](https://doc.rust-lang.org/nightly/book/ch19-06-macros.html#declarative-macros-with-macro_rules-for-general-metaprogramming) is a bit strange, so we won't try to write a macro from scratch. Instead we look at the source of the [`println!` macro](https://doc.rust-lang.org/nightly/std/macro.println!.html) in the standard library:

```
#[macro_export]
macro_rules! println {
    () => (print!("\n"));
    ($($arg:tt)*) => (print!("{}\n", format_args!($($arg)*)));
}
```

Macros are defined through one or more rules, which are similar to `match` arms. The `println` macro has two rules: The first rule for is invocations without arguments (e.g `println!()`), which is expanded to `print!("\n")` and thus just prints a newline. the second rule is for invocations with parameters such as `println!("Hello")` or `println!("Number: {}", 4)`. It is also expanded to an invocation of the `print!` macro, passing all arguments and an additional newline `\n` at the end.

The `#[macro_export]` attribute makes the available to the  whole crate (not just the module it is defined) and external crates. It  also places the macro at the crate root, which means that we have to  import the macro through `use std::println` instead of `std::macros::println`.

The [`print!` macro](https://doc.rust-lang.org/nightly/std/macro.print!.html) is defined as:

```
#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print(format_args!($($arg)*)));
}
```

The macro expands to a call of the [`_print` function](https://github.com/rust-lang/rust/blob/29f5c699b11a6a148f097f82eaa05202f8799bbc/src/libstd/io/stdio.rs#L698) in the `io` module. The [`$crate` variable](https://doc.rust-lang.org/1.30.0/book/first-edition/macros.html#the-variable-crate) ensures that the macro also works from outside the `std` crate by expanding to `std` when it's used in other crates.

The [`format_args` macro](https://doc.rust-lang.org/nightly/std/macro.format_args.html) builds a [fmt::Arguments](https://doc.rust-lang.org/nightly/core/fmt/struct.Arguments.html) type from the passed arguments, which is passed to `_print`. The [`_print` function](https://github.com/rust-lang/rust/blob/29f5c699b11a6a148f097f82eaa05202f8799bbc/src/libstd/io/stdio.rs#L698) of libstd calls `print_to`, which is rather complicated because it supports different `Stdout` devices. We don't need that complexity since we just want to print to the VGA buffer.

To print to the VGA buffer, we just copy the `println!` and `print!` macros, but modify them to use our own `_print` function:

```
// in src/vga_buffer.rs

#[macro_export]
macro_rules! print {
    ($($arg:tt)*) => ($crate::vga_buffer::_print(format_args!($($arg)*)));
}

#[macro_export]
macro_rules! println {
    () => ($crate::print!("\n"));
    ($($arg:tt)*) => ($crate::print!("{}\n", format_args!($($arg)*)));
}

#[doc(hidden)]
pub fn _print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```

One thing that we changed from the original `println` definition is that we prefixed the invocations of the `print!` macro with `$crate` too. This ensures that we don't need to have to import the `print!` macro too if we only want to use `println`.

Like in the standard library, we add the `#[macro_export]`  attribute to both macros to make them available everywhere in our  crate. Note that this places the macros in the root namespace of the  crate, so importing them via `use crate::vga_buffer::println` does not work. Instead, we have to do `use crate::println`.

The `_print` function locks our static `WRITER` and calls the `write_fmt` method on it. This method is from the `Write` trait, we need to import that trait. The additional `unwrap()` at the end panics if printing isn't successful. But since we always return `Ok` in `write_str`, that should not happen.

Since the macros need to be able to call `_print` from  outside of the module, the function needs to be public. However, since  we consider this a private implementation detail, we add the [`doc(hidden)` attribute](https://doc.rust-lang.org/nightly/rustdoc/the-doc-attribute.html#dochidden) to hide it from the generated documentation.

### Hello World using `println`

Now we can use `println` in our `_start` function:

```
// in src/main.rs

#[no_mangle]
pub extern "C" fn _start() {
    println!("Hello World{}", "!");

    loop {}
}
```

Note that we don't have to import the macro in the main function, because it already lives in the root namespace.

As expected, we now see a *“Hello World!”* on the screen:

![QEMU printing “Hello World!”](https://os.phil-opp.com/vga-text-mode/vga-hello-world.png)

### Printing Panic Messages

Now that we have a `println` macro, we can use it in our panic function to print the panic message and the location of the panic:

```
// in main.rs

/// This function is called on panic.
#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    println!("{}", info);
    loop {}
}
```

When we now insert `panic!("Some panic message");` in our `_start` function, we get the following output:

![QEMU printing “panicked at 'Some panic message', src/main.rs:28:5](https://os.phil-opp.com/vga-text-mode/vga-panic.png)

So we know not only that a panic has occurred, but also the panic message and where in the code it happened.

## Summary

In this post we learned about the structure of the VGA text buffer  and how it can be written through the memory mapping at address `0xb8000`.  We created a Rust module that encapsulates the unsafety of writing to  this memory mapped buffer and presents a safe and convenient interface  to the outside.

We also saw how easy it is to add dependencies on third-party libraries, thanks to cargo. The two dependencies that we added, `lazy_static` and `spin`, are very useful in OS development and we will use them in more places in future posts.

## What's next?

The next post explains how to set up Rust's built in unit test  framework. We will then create some basic unit tests for the VGA buffer  module from this post.