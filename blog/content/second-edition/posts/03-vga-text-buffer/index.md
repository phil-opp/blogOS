+++
title = "VGA Text Mode"
order = 3
path = "vga-text-mode"
date  = 2018-02-26
template = "second-edition/page.html"
+++

The [VGA text mode] is a simple way to print text to the screen. In this post, we create an interface that makes its usage safe and simple, by encapsulating all unsafety in a separate module and providing support for Rust's [formatting macros].

[VGA text mode]: https://en.wikipedia.org/wiki/VGA-compatible_text_mode
[formatting macros]: https://doc.rust-lang.org/std/fmt/#related-macros

<!-- more -->

This blog is openly developed on [Github]. If you have any problems or questions, please open an issue there. You can also leave comments [at the bottom].

[Github]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments

## The VGA Text Buffer
To print a character to the screen in VGA text mode, one has to write it to the text buffer of the VGA hardware. The VGA text buffer is a two-dimensional array with typically 25 rows and 80 columns, which is directly rendered to the screen. Each array entry describes a single screen character through the following format:

Bit(s) | Value
------ | ----------------
0-7    | ASCII code point
8-11   | Foreground color
12-14  | Background color
15     | Blink

The following colors are available:

Number | Color      | Number + Bright Bit | Bright Color
------ | ---------- | ------------------- | -------------
0x0    | Black      | 0x8                 | Dark Gray
0x1    | Blue       | 0x9                 | Light Blue
0x2    | Green      | 0xa                 | Light Green
0x3    | Cyan       | 0xb                 | Light Cyan
0x4    | Red        | 0xc                 | Light Red
0x5    | Magenta    | 0xd                 | Pink
0x6    | Brown      | 0xe                 | Yellow
0x7    | Light Gray | 0xf                 | White

Bit 4 is the _bright bit_, which turns for example blue into light blue.

The VGA text buffer is accessible via [memory-mapped I/O] to the address `0xb8000`. This means that reads and writes to that address don't access the RAM, but directly the text buffer on the VGA hardware. This means that we can read and write it through normal memory operations to that address.

[memory-mapped I/O]: https://en.wikipedia.org/wiki/Memory-mapped_I/O

Note that memory-mapped hardware might not support all normal RAM operations. For example, a device could only support byte-wise reads and return junk when an `u64` is read. Fortunately, the text buffer [supports normal reads and writes], so that we don't have to treat it in special way.

[supports normal reads and writes]: https://web.stanford.edu/class/cs140/projects/pintos/specs/freevga/vga/vgamem.htm#manip

## A Rust Module
Now that we know how the VGA buffer works, we can create a Rust module to handle printing:

```rust
// in src/main.rs
mod vga_buffer;
```

The content of this module can live either in `src/vga_buffer.rs` or `src/vga_buffer/mod.rs`. The latter supports submodules while the former does not. Our module does not need any submodules so we create it as `src/vga_buffer.rs`.

All of the code below goes into our new module (unless specified otherwise).

### Colors
First, we represent the different colors using an enum:

```rust
#[allow(dead_code)]
#[derive(Debug, Clone, Copy)]
#[repr(u8)]
pub enum Color {
    Black      = 0,
    Blue       = 1,
    Green      = 2,
    Cyan       = 3,
    Red        = 4,
    Magenta    = 5,
    Brown      = 6,
    LightGray  = 7,
    DarkGray   = 8,
    LightBlue  = 9,
    LightGreen = 10,
    LightCyan  = 11,
    LightRed   = 12,
    Pink       = 13,
    Yellow     = 14,
    White      = 15,
}
```
We use a [C-like enum] here to explicitly specify the number for each color. Because of the `repr(u8)` attribute each enum variant is stored as an `u8`. Actually 4 bits would be sufficient, but Rust doesn't have an `u4` type.

[C-like enum]: http://rustbyexample.com/custom_types/enum/c_like.html

Normally the compiler would issue a warning for each unused variant. By using the `#[allow(dead_code)]` attribute we disable these warnings for the `Color` enum.

By [deriving] the [`Copy`], [`Clone`] and [`Debug`] traits, we enable [copy semantics] for the type and make it printable.

[deriving]: http://rustbyexample.com/trait/derive.html
[`Copy`]: https://doc.rust-lang.org/nightly/core/marker/trait.Copy.html
[`Clone`]: https://doc.rust-lang.org/nightly/core/clone/trait.Clone.html
[`Debug`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Debug.html
[copy semantics]: https://doc.rust-lang.org/book/first-edition/ownership.html#copy-types

To represent a full color code that specifies foreground and background color, we create a [newtype] on top of `u8`:

[newtype]: https://rustbyexample.com/generics/new_types.html

```rust
#[derive(Debug, Clone, Copy)]
struct ColorCode(u8);

impl ColorCode {
    const fn new(foreground: Color, background: Color) -> ColorCode {
        ColorCode((background as u8) << 4 | (foreground as u8))
    }
}
```
The `ColorCode` contains the full color byte, containing foreground and background color. Like before, we derive the `Copy` and `Debug` traits for it. The `new` function is a [const function] to allow it in static initializers. As `const` functions are unstable we need to add the `const_fn` feature in `src/main.rs`:

[const function]: https://doc.rust-lang.org/unstable-book/language-features/const-fn.html

```rust
// in src/main.rs
#![feature(const_fn)]
```

### Text Buffer
Now we can add structures to represent a screen character and the text buffer:

```rust
#[derive(Debug, Clone, Copy)]
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
Since the field ordering in default structs is undefined in Rust, we need the [`repr(C)`] attribute. It guarantees that the struct's fields are laid out exactly like in a C struct and thus guarantees the correct field ordering.

[`repr(C)`]: https://doc.rust-lang.org/nightly/nomicon/other-reprs.html#reprc

To actually write to screen, we now create a writer type:

```rust
pub struct Writer {
    column_position: usize,
    color_code: ColorCode,
    buffer: &'static mut Buffer,
}
```
The writer will always write to the last line and shift lines up when a line is full (or on `\n`). The `column_position` field keeps track of the current position in the last row. The current foreground and background colors are specified by `color_code` and a reference to the VGA buffer is stored in `buffer`. Note that we need an [explicit lifetime] here to tell the compiler how long the reference is valid. The [`'static`] lifetime specifies that the reference is valid for the whole program run time (which is true for the VGA text buffer).

[explicit lifetime]: https://doc.rust-lang.org/book/first-edition/lifetimes.html#syntax
[`'static`]: https://doc.rust-lang.org/book/first-edition/lifetimes.html#static

### Printing
Now we can use the `Writer` to modify the buffer's characters. First we create a method to write a single ASCII byte:

```rust
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
                    color_code: color_code,
                };
                self.column_position += 1;
            }
        }
    }

    fn new_line(&mut self) {/* TODO */}
}
```
If the byte is the [newline] byte `\n`, the writer does not print anything. Instead it calls a `new_line` method, which we'll implement later. Other bytes get printed to the screen in the second match case.

[newline]: https://en.wikipedia.org/wiki/Newline

When printing a byte, the writer checks if the current line is full. In that case, a `new_line` call is required before to wrap the line. Then it writes a new `ScreenChar` to the buffer at the current position. Finally, the current column position is advanced.

To print whole strings, we can convert them to bytes and print them one-by-one:

```rust
impl Writer {
    pub fn write_string(&mut self, s: &str) {
        for byte in s.bytes() {
            self.write_byte(byte)
        }
    }
}
```

#### Try it out!
To write some characters to the screen, you can create a temporary function:

```rust
pub fn print_something() {
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_string("ello");
}
```
It first creates a new Writer that points to the VGA buffer at `0xb8000`. The syntax for this might seem a bit strange: First, we cast the integer `0xb8000` as an mutable [raw pointer]. Then we convert it to a mutable reference by dereferencing it (through `*`) and immediately borrowing it again (through `&mut`). This conversion requires an [`unsafe` block], since the compiler can't guarantee that the raw pointer is valid.

[raw pointer]: https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#dereferencing-a-raw-pointer
[`unsafe` block]: https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#unsafe-rust

Secondly, it writes the bytes `b'H'` and the string `"ello"'` to it. The `b` prefix creates a [byte literal], which represents an ASCII character. When we call `vga_buffer::print_something` in our `_start` function (in `src/main.rs`), a `Hello` should be printed in the _lower_ left corner of the screen in yellow:

[byte literal]: https://doc.rust-lang.org/reference/tokens.html#byte-literals

![QEMU output with a blue `Hello` in the lower left corner](vga-hello.png)

When printing strings with some special characters like `ä` or `λ`, you'll notice that they cause weird symbols on screen. That's because they are represented by multiple bytes in [UTF-8]. By converting them to bytes, we of course get strange results. But since the VGA buffer doesn't support UTF-8, it's not possible to display these characters anyway.

[core tracking issue]: https://github.com/rust-lang/rust/issues/27701
[UTF-8]: http://www.fileformat.info/info/unicode/utf8.htm

### Volatile
We just saw that our `Hello` was printed correctly. However, it might not work with future Rust compilers that optimize more aggressively.

The problem is that we only write to the `Buffer` and never read from it again. The compiler doesn't know that we really access VGA buffer memory (instead of normal RAM) and knows nothing about the side effect that some characters appear on the screen. So it might decide that these writes are unnecessary and can be omitted. To avoid this erroneous optimization, we need to specify these writes as _[volatile]_. This tells the compiler that the write has side effects and should not be optimized away.

[volatile]: https://en.wikipedia.org/wiki/Volatile_(computer_programming)

In order to use volatile writes for the VGA buffer, we use the [volatile][volatile crate] library. This _crate_ (this is how packages are called in the Rust world) provides a `Volatile` wrapper type with `read` and `write` methods. These methods internally use the [read_volatile] and [write_volatile] functions of the standard library and thus guarantee that the reads/writes are not optimized away.

[volatile crate]: https://docs.rs/volatile
[read_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.read_volatile.html
[write_volatile]: https://doc.rust-lang.org/nightly/core/ptr/fn.write_volatile.html

We can add a dependency on the `volatile` crate by adding it to the `dependencies` section of our `Cargo.toml`:

```toml
# in Cargo.toml

[dependencies]
volatile = "0.2.3"
```

The `0.2.3` is the [semantic] version number. For more information, see the [Specifying Dependencies] guide of the cargo documentation.

[semantic]: http://semver.org/
[Specifying Dependencies]: http://doc.crates.io/specifying-dependencies.html

Now we've declared that our project depends on the `volatile` crate and are able to import it in `src/main.rs`:

```rust
// in src/main.rs

extern crate volatile;
```

Let's use it to make writes to the VGA buffer volatile. We update our `Buffer` type as follows:

```rust
// in src/vga_buffer.rs

use volatile::Volatile;

struct Buffer {
    chars: [[Volatile<ScreenChar>; BUFFER_WIDTH]; BUFFER_HEIGHT],
}
```
Instead of a `ScreenChar`, we're now using a `Volatile<ScreenChar>`. (The `Volatile` type is [generic] and can wrap (almost) any type). This ensures that we can't accidentally write to it through a “normal” write. Instead, we have to use the `write` method now.

[generic]: https://doc.rust-lang.org/book/generics.html

This means that we have to update our `Writer::write_byte` method:

```rust
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

Instead of a normal assignment using `=`, we're now using the `write` method. This guarantees that the compiler will never optimize away this write.

### Formatting Macros
It would be nice to support Rust's formatting macros, too. That way, we can easily print different types like integers or floats. To support them, we need to implement the [`core::fmt::Write`] trait. The only required method of this trait is `write_str` that looks quite similar to our `write_string` method, just with a `fmt::Result` return type:

[`core::fmt::Write`]: https://doc.rust-lang.org/nightly/core/fmt/trait.Write.html

```rust
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

```rust
pub fn print_something() {
    use core::fmt::Write;
    let mut writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };

    writer.write_byte(b'H');
    writer.write_str("ello! ").unwrap();
    write!(writer, "The numbers are {} and {}", 42, 1.0/3.0).unwrap();
}
```

Now you should see a `Hello! The numbers are 42 and 0.3333333333333333` at the bottom of the screen. The `write!` call returns a `Result` which causes a warning if not used, so we call the [`unwrap`] function on it, which panics if an error occurs. This isn't a problem in our case, since writes to the VGA buffer never fail.

[`unwrap`]: https://doc.rust-lang.org/core/result/enum.Result.html#method.unwrap

### Newlines
Right now, we just ignore newlines and characters that don't fit into the line anymore. Instead we want to move every character one line up (the top line gets deleted) and start at the beginning of the last line again. To do this, we add an implementation for the `new_line` method of `Writer`:

```rust
impl Writer {
    fn new_line(&mut self) {
        for row in 1..BUFFER_HEIGHT {
            for col in 0..BUFFER_WIDTH {
                let character = self.buffer.chars[row][col].read();
                self.buffer.chars[row - 1][col].write(character);
            }
        }
        self.clear_row(BUFFER_HEIGHT-1);
        self.column_position = 0;
    }

    fn clear_row(&mut self, row: usize) {/* TODO */}
}
```
We iterate over all screen characters and move each characters one row up. Note that the range notation (`..`) is exclusive the upper bound. We also omit the 0th row (the first range starts at `1`) because it's the row that is shifted off screen.

To finish the newline code, we add the `clear_row` method:

```rust
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

```rust
pub static WRITER: Writer = Writer {
    column_position: 0,
    color_code: ColorCode::new(Color::Yellow, Color::Black),
    buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
};
```

However, if we try to compile it now, the following errors occur:

```
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

The problem is that Rust's const evaluator is not able to convert raw pointers to references at compile time. Maybe it will work someday when `const` functions become more powerful. But until then, we have to find another solution.

### Lazy Statics
Fortunately the `lazy_static` macro exists. Instead of evaluating a `static` at compile time, the macro performs the initialization when the `static` is referenced the first time. Thus, we can do almost everything in the initialization block and are even able to read runtime values.

Let's add the `lazy_static` crate to our project:

```rust
// in src/main.rs

#[macro_use]
extern crate lazy_static;
```

```toml
# in Cargo.toml

[dependencies.lazy_static]
version = "1.0"
features = ["spin_no_std"]
```
We need the `spin_no_std` feature, since we don't link the standard library.

With `lazy_static`, we can define our static `WRITER` without problems:

```rust
lazy_static! {
    pub static ref WRITER: Writer = Writer {
        column_position: 0,
        color_code: ColorCode::new(Color::Yellow, Color::Black),
        buffer: unsafe { &mut *(0xb8000 as *mut Buffer) },
    };
}
```

However, this `WRITER` is pretty useless since it is immutable. This means that we can't write anything to it (since all the write methods take `&mut self`). One possible solution would be to use a [mutable static]. But then every read and write to it would be unsafe since it could easily introduce data races and other bad things. Using `static mut` is highly discouraged, there were even proposals to [remove it][remove static mut]. But what are the alternatives? We could try to use a immutable static with a cell type like [RefCell] or even [UnsafeCell] that provides [interior mutability]. But these types aren't [Sync] \(with good reason), so we can't use them in statics.

[mutable static]: https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable
[remove static mut]: https://internals.rust-lang.org/t/pre-rfc-remove-static-mut/1437
[RefCell]: https://doc.rust-lang.org/nightly/core/cell/struct.RefCell.html
[UnsafeCell]: https://doc.rust-lang.org/nightly/core/cell/struct.UnsafeCell.html
[interior mutability]: https://doc.rust-lang.org/book/first-edition/mutability.html#interior-vs-exterior-mutability
[Sync]: https://doc.rust-lang.org/nightly/core/marker/trait.Sync.html

### Spinlocks
To get synchronized interior mutability, users of the standard library can use [Mutex]. It provides mutual exclusion by blocking threads when the resource is already locked. But our basic kernel does not have any blocking support or even a concept of threads, so we can't use it either. However there is a really basic kind of mutex in computer science that requires no operating system features: the [spinlock]. Instead of blocking, the threads simply try to lock it again and again in a tight loop and thus burn CPU time until the mutex is free again.

[Mutex]: https://doc.rust-lang.org/nightly/std/sync/struct.Mutex.html
[spinlock]: https://en.wikipedia.org/wiki/Spinlock

To use a spinning mutex, we can add the [spin crate] as a dependency:

[spin crate]: https://crates.io/crates/spin

```toml
# in Cargo.toml
[dependencies]
spin = "0.4.6"
```

```rust
// in src/main.rs
extern crate spin;
```

Then we can use the spinning Mutex to add safe [interior mutability] to our static `WRITER`:

```rust
// in src/vga_buffer.rs again
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

```rust
// in src/main.rs
#[no_mangle]
pub fn _start() -> ! {
    use core::fmt::Write;
    vga_buffer::WRITER.lock().write_str("Hello again").unwrap();
    write!(vga_buffer::WRITER.lock(), ", some numbers: {} {}", 42, 1.337).unwrap();

    loop {}
}
```
We need to import the `fmt::Write` trait in order to be able to use its functions.

### Safety
Note that we only have a single unsafe block in our code, which is needed to create a `Buffer` reference pointing to `0xb8000`. Afterwards, all operations are safe. Rust uses bounds checking for array accesses by default, so we can't accidentally write outside the buffer. Thus, we encoded the required conditions in the type system and are able to provide a safe interface to the outside.

### A println Macro
Now that we have a global writer, we can add a `println` macro that can be used from anywhere in the codebase. Rust's [macro syntax] is a bit strange, so we won't try to write a macro from scratch. Instead we look at the source of the [`println!` macro] in the standard library:

[macro syntax]: https://doc.rust-lang.org/nightly/book/macros.html
[`println!` macro]: https://doc.rust-lang.org/nightly/std/macro.println!.html

```rust
macro_rules! println {
    () => (print!("\n"));
    ($fmt:expr) => (print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => (print!(concat!($fmt, "\n"), $($arg)*));
}
```
Macros are defined through one or more rules, which are similar to `match` arms. The `println` macro has three rules: The first rule for is invocations without arguments (e.g `println!()`), the second rule is for invocations with a single argument (e.g. `println!("Hello")`) and the third rule is for invocations with additional parameters (e.g. `println!("{}{}", 4, 2)`).

First line just prints a `\n` symbol which literally means "don't print anything, just break the line".
Last two rules simply append a newline character (`\n`) to the format string and then invoke the [`print!` macro], which is defined as:

[`print!` macro]: https://doc.rust-lang.org/nightly/std/macro.print!.html

```rust
macro_rules! print {
    ($($arg:tt)*) => ($crate::io::_print(format_args!($($arg)*)));
}
```
The macro expands to a call of the [`_print` function] in the `io` module. The [`$crate` variable] ensures that the macro also works from outside the `std` crate. For example, it expands to `::std` when it's used in other crates.

The [`format_args` macro] builds a [fmt::Arguments] type from the passed arguments, which is passed to `_print`. The [`_print` function] of libstd calls `print_to`, which is rather complicated because it supports different `Stdout` devices. We don't need that complexity since we just want to print to the VGA buffer.

[`_print` function]: https://github.com/rust-lang/rust/blob/29f5c699b11a6a148f097f82eaa05202f8799bbc/src/libstd/io/stdio.rs#L698
[`$crate` variable]: https://doc.rust-lang.org/book/first-edition/macros.html#the-variable-crate
[`format_args` macro]: https://doc.rust-lang.org/nightly/std/macro.format_args.html
[fmt::Arguments]: https://doc.rust-lang.org/nightly/core/fmt/struct.Arguments.html

To print to the VGA buffer, we just copy the `println!` and `print!` macros, but modify them to use a `print` function:

```rust
// in src/vga_buffer.rs

macro_rules! print {
    ($($arg:tt)*) => ($crate::vga_buffer::print(format_args!($($arg)*)));
}

macro_rules! println {
    () => (print!("\n"));
    ($fmt:expr) => (print!(concat!($fmt, "\n")));
    ($fmt:expr, $($arg:tt)*) => (print!(concat!($fmt, "\n"), $($arg)*));
}

pub fn print(args: fmt::Arguments) {
    use core::fmt::Write;
    WRITER.lock().write_fmt(args).unwrap();
}
```
The `print` function locks our static `WRITER` and calls the `write_fmt` method on it. This method is from the `Write` trait, we need to import that trait. The additional `unwrap()` at the end panics if printing isn't successful. But since we always return `Ok` in `write_str`, that should not happen.

### Hello World using `println`
To use `println` in `main.rs`, we need to import the macros of the VGA buffer module first. Therefore we add a `#[macro_use]` attribute to the module declaration:

```rust
// in src/main.rs

#[macro_use]
mod vga_buffer;

#[no_mangle]
pub extern fn rust_main() {
    println!("Hello World{}", "!");

    loop {}
}
```

Since we imported the macros at crate level, they are available in all modules and thus provide an easy and safe interface to the VGA buffer.

As expected, we now see a _“Hello World!”_ on the screen:

![QEMU printing “Hello World!”](vga-hello-world.png)

## Summary
In this post we learned about the structure of the VGA text buffer and how it can be written through the memory mapping at address `0xb8000`. We created a Rust module that encapsulates the unsafety of writing to this memory mapped buffer and presents a safe and convenient interface to the outside.

We also saw how easy it is to add dependencies on third-party libraries, thanks to cargo. The two dependencies that we added, `lazy_static` and `spin`, are very useful in OS development and we will use them in more places in future posts.

## What's next?
In the next post, we will explore _CPU exceptions_. These exceptions are thrown by the CPU when something illegal happens, such as a division by zero or an access to an unmapped memory page (a so-called “page fault”). Being able to catch and examine these exceptions is very important for debugging future errors. Exception handling is also very similar to the handling of hardware interrupts, which is required for keyboard support.
