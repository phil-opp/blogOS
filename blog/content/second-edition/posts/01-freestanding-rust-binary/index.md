+++
title = "A Freestanding Rust Binary"
order = 1
path = "freestanding-rust-binary"
date = 2018-02-10
template = "second-edition/page.html"
+++

This post describes how to create a Rust executable that does not link the standard library. This makes it possible to run Rust code on the [bare metal] without an underlying operating system.

[bare metal]: https://en.wikipedia.org/wiki/Bare_machine

<!-- more -->

## Introduction
To write an operating system kernel, we need code that does not depend on any operating system features. This means that we can't use threads, files, heap memory, the network, random numbers, standard output, or any other features requiring OS abstractions or specific hardware. Which makes sense, since we're trying to write our own OS and our own drivers.

This means that we can't use most of the [Rust standard library], but there are a lot of Rust features that we _can_ use. For example, we can use [iterators], [closures], [pattern matching], [option] and [result], [string formatting], and of course the [ownership system]. These features make it possible to write a kernel in a very expressive, high level way without worrying about [undefined behavior] or [memory safety].

[option]: https://doc.rust-lang.org/core/option/
[result]:https://doc.rust-lang.org/core/result/
[Rust standard library]: https://doc.rust-lang.org/std/
[iterators]: https://doc.rust-lang.org/book/second-edition/ch13-02-iterators.html
[closures]: https://doc.rust-lang.org/book/second-edition/ch13-01-closures.html
[pattern matching]: https://doc.rust-lang.org/book/second-edition/ch06-00-enums.html
[string formatting]: https://doc.rust-lang.org/core/macro.write.html
[ownership system]: https://doc.rust-lang.org/book/second-edition/ch04-00-understanding-ownership.html
[undefined behavior]: https://www.nayuki.io/page/undefined-behavior-in-c-and-cplusplus-programs
[memory safety]: https://tonyarcieri.com/it-s-time-for-a-memory-safety-intervention

In order to create an OS kernel in Rust, we need to create an executable that can be run without an underlying operating system. Such an executable is often called a “freestanding” or “bare-metal” executable.

This post describes the necessary steps to create a freestanding Rust binary and explains why the steps are needed. If you're just interested in a minimal example, you can **[jump to the summary](#summary)**.

## Disabling the Standard Library
By default, all Rust crates link the [standard library], which depends on the operating system for features such as threads, files, or networking. It also depends on the C standard library `libc`, which closely interacts with OS services. Since our plan is to write an operating system, we can not use any OS-dependent libraries. So we have to disable the automatic inclusion of the standard library through the [`no_std` attribute].

[standard library]: https://doc.rust-lang.org/std/
[`no_std` attribute]: https://doc.rust-lang.org/book/first-edition/using-rust-without-the-standard-library.html

We start by creating a new cargo application project. The easiest way to do this is through the command line:

```
> cargo new blog_os --bin
```

I named the project `blog_os`, but of course you can choose your own name. The `--bin` flag specifies that we want to create an executable binary (in contrast to a library). When we run the command, cargo creates the following directory structure for us:

```
blog_os
├── Cargo.toml
└── src
    └── main.rs
```

The `Cargo.toml` contains the crate configuration, for example the crate name, the author, the [semantic version] number, and dependencies. The `src/main.rs` file contains the root module of our crate and our `main` function. You can compile your crate through `cargo build` and then run the compiled `blog_os` binary in the `target/debug` subfolder.

[semantic version]: http://semver.org/

### The `no_std` Attribute

Right now our crate implicitly links the standard library. Let's try to disable this by adding the [`no_std` attribute]:

```rust
// main.rs

#![no_std]

fn main() {
    println!("Hello, world!");
}
```

When we try to build it now (by running `cargo build`), the following error occurs:

```
error: cannot find macro `println!` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

The reason for this error is that the [`println` macro] is part of the standard library, which we no longer include. So we can no longer print things. This makes sense, since `println` writes to [standard output], which is a special file descriptor provided by the operating system.

[`println` macro]: https://doc.rust-lang.org/std/macro.println.html
[standard output]: https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29

So let's remove the printing and try again with an empty main function:

```rust
// main.rs

#![no_std]

fn main() {}
```

```
> cargo build
error: language item required, but not found: `panic_fmt`
error: language item required, but not found: `eh_personality`
```

Now the compiler is missing some _language items_. Language items are special pluggable functions that the compiler invokes on certain conditions, for example when the application [panics]. Normally, these items are provided by the standard library, but we disabled it. So we need to provide our own implementations.

[panics]: https://doc.rust-lang.org/stable/book/second-edition/ch09-01-unrecoverable-errors-with-panic.html

### Enabling Unstable Features

Implementing language items is unstable and protected by a so-called _feature gate_. A feature gate is a special attribute that you have to specify at the top of your `main.rs` in order to use the corresponding feature. By doing this you basically say: “I know that this feature is unstable and that it might stop working without any warnings. I want to use it anyway.”

To limit the use of unstable features, the feature gates are not available in the stable or beta Rust compilers, only [nightly Rust] makes it possible to opt-in. This means that you have to use a nightly compiler for OS development for the near future (since we need to implement unstable language items). To install a nightly compiler using [rustup], you just need to run `rustup default nightly` (for more information check out [rustup's documentation]).

[nightly Rust]: https://doc.rust-lang.org/book/first-edition/release-channels.html
[rustup]: https://rustup.rs/
[rustup's documentation]: https://github.com/rust-lang-nursery/rustup.rs#rustup-the-rust-toolchain-installer

After installing a nightly Rust compiler, you can enable the unstable `lang_items` feature by inserting `#![feature(lang_items)]` right at the top of `main.rs`.

### Implementing the Language Items

To create a `no_std` binary, we have to implement the `panic_fmt` and the `eh_personality` language items. The `panic_fmt` item specifies a function that should be invoked when a panic occurs. This function should format an error message (hence the `_fmt` suffix) and then invoke the panic routine. In our case, there is not much we can do, since we can neither print anything nor do we have a panic routine. So we just loop indefinitely:

```rust
#![feature(lang_items)]
#![no_std]

fn main() {}

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(_msg: core::fmt::Arguments,
    _file: &'static str, _line: u32, _column: u32) -> !
{
    loop {}
}
```

The function signature is taken from the [unstable Rust book]. The signature isn't verified by the compiler, so implement it carefully.

[unstable Rust book]: https://doc.rust-lang.org/unstable-book/language-features/lang-items.html#writing-an-executable-without-stdlib

Note that there is already an accepted RFC for a stable panic mechanism, which is only waiting for an implementation. See the [tracking issue](https://github.com/rust-lang/rust/issues/44489) for more information.

Instead of implementing the second language item, `eh_personality`, we remove the need for it by disabling unwinding.

### Disabling Unwinding

The `eh_personality` language item is used for implementing [stack unwinding]. By default, Rust uses unwinding to run the destructors of all live stack variables in case of a panic. This ensures that all used memory is freed and allows the parent thread to catch the panic and continue execution. Unwinding, however, is a complicated process and requires some OS specific libraries (e.g. [libunwind] on Linux or [structured exception handling] on Windows), so we don't want to use it for our operating system.

[stack unwinding]: http://www.bogotobogo.com/cplusplus/stackunwinding.php
[libunwind]: http://www.nongnu.org/libunwind/
[structured exception handling]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms680657(v=vs.85).aspx

There are other use cases as well for which unwinding is undesirable, so Rust provides an option to [abort on panic] instead. This disables the generation of unwinding symbol information and thus considerably reduces binary size. There are multiple places where we can disable unwinding. The easiest way is to add the following lines to our `Cargo.toml`:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

This sets the panic strategy to `abort` for both the `dev` profile (used for `cargo build`) and the `release` profile (used for `cargo build --release`). Now the `eh_personality` language item should no longer be required.

[abort on panic]: https://github.com/rust-lang/rust/pull/32900

However, if we try to compile it now, another language item is required:

```
> cargo build
error: requires `start` lang_item
```

### The `start` attribute

One might think that the `main` function is the first function called when you run a program. However, most languages have a [runtime system], which is responsible for things such as garbage collection (e.g. in Java) or software threads (e.g. goroutines in Go). This runtime needs to be called before `main`, since it needs to initialize itself.

[runtime system]: https://en.wikipedia.org/wiki/Runtime_system

In a typical Rust binary that links the standard library, execution starts in a C runtime library called `crt0` (“C runtime zero”), which sets up the environment for a C application. This includes creating a stack and placing the arguments in the right registers. The C runtime then invokes the [entry point of the Rust runtime][rt::lang_start], which is marked by the `start` language item. Rust only has a very minimal runtime, which takes care of some small things such as setting up stack overflow guards or printing a backtrace on panic. The runtime then finally calls the `main` function.

[rt::lang_start]: https://github.com/rust-lang/rust/blob/bb4d1491466d8239a7a5fd68bd605e3276e97afb/src/libstd/rt.rs#L32-L73

Our freestanding executable does not have access to the Rust runtime and `crt0`, so we need to define our own entry point. Implementing the `start` language item wouldn't help, since it would still require `crt0`. Instead, we need to overwrite the `crt0` entry point directly.

### Overwriting the Entry Point
To tell the Rust compiler that we don't want to use the normal entry point chain, we add the `#![no_main]` attribute.

```rust
#![feature(lang_items)]
#![no_std]
#![no_main]

#[lang = "panic_fmt"]
#[no_mangle]
pub extern fn rust_begin_panic(_msg: core::fmt::Arguments,
    _file: &'static str, _line: u32, _column: u32) -> !
{
    loop {}
}
```

You might notice that we removed the `main` function. The reason is that a `main` doesn't make sense without an underlying runtime that calls it. Instead, we are now overwriting the operating system entry point.

The entry point convention depends on your operating system. I recommend you to read the Linux section even if you're on a different OS because we will use this convention for our kernel.

#### Linux
On Linux, the default entry point is called `_start`. The linker just looks for a function with that name and sets this function as entry point the executable. So to overwrite the entry point, we define our own `_start` function:

```rust
#[no_mangle]
pub extern fn _start() -> ! {
    loop {}
}
```

It's important that we disable the [name mangling] through the `no_mangle` attribute, otherwise the compiler would generate some cryptic `_ZN3blog_os4_start7hb173fedf945531caE` symbol that the linker wouldn't recognize.

[name mangling]: https://en.wikipedia.org/wiki/Name_mangling

The `!` return type means that the function is diverging, i.e. not allowed to ever return. This is required because the entry point is not called by any function, but invoked directly by the operating system or bootloader. So instead of returning, the entry point should e.g. invoke the [`exit` system call] of the operating system. In our case, shutting down the machine could be a reasonable action, since there's nothing left to do if a freestanding binary returns. For now, we fulfill the requirement by looping endlessly.

[`exit` system call]: https://en.wikipedia.org/wiki/Exit_(system_call)

If we try to build it now, an ugly linker error occurs:

```
error: linking with `cc` failed: exit code: 1
  |
  = note: "cc" "-Wl,--as-needed" "-Wl,-z,noexecstack" "-m64" "-L"
    "/…/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib"
    "/…/blog_os/target/debug/deps/blog_os-f7d4ca7f1e3c3a09.0.o" […]
    "-o" "/…/blog_os/target/debug/deps/blog_os-f7d4ca7f1e3c3a09"
    "-Wl,--gc-sections" "-pie" "-Wl,-z,relro,-z,now" "-nodefaultlibs"
    "-L" "/…/blog_os/target/debug/deps"
    "-L" "/…/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib"
    "-Wl,-Bstatic"
    "/…/.rustup/toolchains/nightly-x86_64-unknown-linux-gnu/lib/rustlib/x86_64-unknown-linux-gnu/lib/libcore-dd5bba80e2402629.rlib"
    "-Wl,-Bdynamic"
  = note: /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x12): undefined reference to `__libc_csu_fini'
          /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x19): undefined reference to `__libc_csu_init'
          /usr/lib/gcc/x86_64-linux-gnu/5/../../../x86_64-linux-gnu/Scrt1.o: In function `_start':
          (.text+0x25): undefined reference to `__libc_start_main'
          collect2: error: ld returned 1 exit status

```

The problem is that we still link the startup routine of the C runtime, which requires some symbols of the C standard library `libc`, which we don't link due to the `no_std` attribute. So we need to get rid of the C startup routine. We can do that by passing the `-nostartfiles` flag to the linker.

One way to pass linker attributes via cargo is the `cargo rustc` command. The command behaves exactly like `cargo build`, but allows to pass options to `rustc`, the underlying Rust compiler. `rustc` has the (unstable) `-Z pre-link-arg` flag, which passes an argument to the linker. Combined, our new build command looks like this:

```
> cargo rustc -- -Z pre-link-arg=-nostartfiles
```

With this command, our crate finally builds as a freestanding executable!

#### Windows
On Windows, the linker requires two entry points: `WinMain` and `WinMainCRTStartup`, [depending on the used subsystem](https://msdn.microsoft.com/en-us/library/f9t8842e.aspx). Like on Linux, we overwrite the entry points by defining `no_mangle` functions:

```rust
#[no_mangle]
pub extern fn WinMainCRTStartup() -> ! {
    WinMain();
}

#[no_mangle]
pub extern fn WinMain() -> ! {
    loop {}
}
```

We just call `WinMain` from `WinMainCRTStartup` to avoid any ambiguity which function is called.

#### macOS
macOS [does not support statically linked binaries], so we have to link the `libSystem` library. The entry point is called `main`:

[does not support statically linked binaries]: https://developer.apple.com/library/content/qa/qa1118/_index.html

```rust
#[no_mangle]
pub extern fn main() -> ! {
    loop {}
}
```

To build it and link `libSystem`, we execute:

```
> cargo rustc -- -Z pre-link-arg=-lSystem
```


## Summary

A minimal freestanding Rust binary looks like this:

`src/main.rs`:

```rust
#![feature(lang_items)] // required for defining the panic handler
#![no_std] // don't link the Rust standard library
#![no_main] // disable all Rust-level entry points

#[lang = "panic_fmt"] // define a function that should be called on panic
#[no_mangle]
pub extern fn rust_begin_panic(_msg: core::fmt::Arguments,
    _file: &'static str, _line: u32, _column: u32) -> !
{
    loop {}
}

// On Linux:
#[no_mangle] // don't mangle the name of this function
pub extern fn _start() -> ! {
    // this function is the entry point, since the linker looks for a function
    // named `_start` by default
    loop {}
}

// On Windows:
#[no_mangle]
pub extern fn WinMainCRTStartup() -> ! {
    WinMain();
}

#[no_mangle]
pub extern fn WinMain() -> ! {
    loop {}
}

// On macOS:

#[no_mangle]
pub extern fn main() -> ! {
    loop {}
}
```

`Cargo.toml`:

```toml
[package]
name = "crate_name"
version = "0.1.0"
authors = ["Author Name <author@example.com>"]

# the profile used for `cargo build`
[profile.dev]
panic = "abort" # disable stack unwinding on panic

# the profile used for `cargo build --release`
[profile.release]
panic = "abort" # disable stack unwinding on panic
```

It can be compiled with:

```bash
# Linux
> cargo rustc -- -Z pre-link-arg=-nostartfiles
# Windows
> cargo build
# macOS
> cargo rustc -- -Z pre-link-arg=-lSystem
```

Note that this is just a minimal example of a freestanding Rust binary. This binary expects various things, for example that a stack is initialized when the `_start` function is called. **So for any real use of such a binary, more steps are required**.

## What's next?

The [next post] builds upon our minimal freestanding binary by explaining the steps needed for creating a minimal operating system kernel. It explains how to configure the kernel for the target system, how to start it using a bootloader, and how to print something to the screen.

[next post]: ./second-edition/posts/02-minimal-rust-kernel/index.md
