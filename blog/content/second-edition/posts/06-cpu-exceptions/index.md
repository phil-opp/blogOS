+++
title = "CPU Exceptions"
order = 6
path = "cpu-exceptions"
date  = 2018-06-17
template = "second-edition/page.html"
+++

CPU exceptions occur in various erroneous situations, for example when accessing an invalid memory address or when dividing by zero. To react to them we have to set up an _interrupt descriptor table_ that provides handler functions. At the end of this post, our kernel will be able to catch [breakpoint exceptions] and to resume normal execution afterwards.

[breakpoint exceptions]: http://wiki.osdev.org/Exceptions#Breakpoint

<!-- more -->

This blog is openly developed on [Github]. If you have any problems or questions, please open an issue there. You can also leave comments [at the bottom].

[Github]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments

## Overview
An exception signals that something is wrong with the current instruction. For example, the CPU issues an exception if the current instruction tries to divide by 0. When an exception occurs, the CPU interrupts its current work and immediately calls a specific exception handler function, depending on the exception type.

On x86 there are about 20 different CPU exception types. The most important are:

- **Page Fault**: A page fault occurs on illegal memory accesses. For example, if the current instruction tries to read from an unmapped page or tries to write to a read-only page.
- **Invalid Opcode**: This exception occurs when the current instruction is invalid, for example when we try to use newer [SSE instructions] on an old CPU that does not support them.
- **General Protection Fault**: This is the exception with the broadest range of causes. It occurs on various kinds of access violations such as trying to executing a privileged instruction in user level code or writing reserved fields in configuration registers.
- **Double Fault**: When an exception occurs, the CPU tries to call the corresponding handler function. If another exception occurs _while calling the exception handler_, the CPU raises a double fault exception. This exception also occurs when there is no handler function registered for an exception.
- **Triple Fault**: If an exception occurs while the CPU tries to call the double fault handler function, it issues a fatal _triple fault_. We can't catch or handle a triple fault. Most processors react by resetting themselves and rebooting the operating system.

[SSE instructions]: https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions

For the full list of exceptions check out the [OSDev wiki][exceptions].

[exceptions]: http://wiki.osdev.org/Exceptions

### The Interrupt Descriptor Table
In order to catch and handle exceptions, we have to set up a so-called _Interrupt Descriptor Table_ (IDT). In this table we can specify a handler function for each CPU exception. The hardware uses this table directly, so we need to follow a predefined format. Each entry must have the following 16-byte structure:

Type| Name                     | Description
----|--------------------------|-----------------------------------
u16 | Function Pointer [0:15]  | The lower bits of the pointer to the handler function.
u16 | GDT selector             | Selector of a code segment in the [global descriptor table].
u16 | Options                  | (see below)
u16 | Function Pointer [16:31] | The middle bits of the pointer to the handler function.
u32 | Function Pointer [32:63] | The remaining bits of the pointer to the handler function.
u32 | Reserved                 |

[global descriptor table]: https://en.wikipedia.org/wiki/Global_Descriptor_Table

The options field has the following format:

Bits  | Name                              | Description
------|-----------------------------------|-----------------------------------
0-2   | Interrupt Stack Table Index       | 0: Don't switch stacks, 1-7: Switch to the n-th stack in the Interrupt Stack Table when this handler is called.
3-7   | Reserved              |
8     | 0: Interrupt Gate, 1: Trap Gate   | If this bit is 0, interrupts are disabled when this handler is called.
9-11  | must be one                       |
12    | must be zero                      |
13‑14 | Descriptor Privilege Level (DPL)  | The minimal privilege level required for calling this handler.
15    | Present                           |

Each exception has a predefined IDT index. For example the invalid opcode exception has table index 6 and the page fault exception has table index 14. Thus, the hardware can automatically load the corresponding IDT entry for each exception. The [Exception Table][exceptions] in the OSDev wiki shows the IDT indexes of all exceptions in the “Vector nr.” column.

When an exception occurs, the CPU roughly does the following:

1. Push some registers on the stack, including the instruction pointer and the [RFLAGS] register. (We will use these values later in this post.)
2. Read the corresponding entry from the Interrupt Descriptor Table (IDT). For example, the CPU reads the 14-th entry when a page fault occurs.
3. Check if the entry is present. Raise a double fault if not.
4. Disable hardware interrupts if the entry is an interrupt gate (bit 40 not set).
5. Load the specified [GDT] selector into the CS segment.
6. Jump to the specified handler function.

[RFLAGS]: https://en.wikipedia.org/wiki/FLAGS_register
[GDT]: https://en.wikipedia.org/wiki/Global_Descriptor_Table

Don't worry about steps 4 and 5 for now, we will learn about the global descriptor table and hardware interrupts in future posts.

## An IDT Type
Instead of creating our own IDT type, we will use the [`InterruptDescriptorTable` struct] of the `x86_64` crate, which looks like this:

[`InterruptDescriptorTable` struct]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/struct.InterruptDescriptorTable.html

``` rust
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
    // some fields omitted
}
```

The fields have the type [`idt::Entry<F>`], which is a struct that represents the fields of an IDT entry (see the table above). The type parameter `F` defines the expected handler function type. We see that some entries require a [`HandlerFunc`] and some entries require a [`HandlerFuncWithErrCode`]. The page fault even has its own special type: [`PageFaultHandlerFunc`].

[`idt::Entry<F>`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/struct.Entry.html
[`HandlerFunc`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/type.HandlerFunc.html
[`HandlerFuncWithErrCode`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/type.HandlerFuncWithErrCode.html
[`PageFaultHandlerFunc`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/type.PageFaultHandlerFunc.html

Let's look at the `HandlerFunc` type first:

```rust
type HandlerFunc = extern "x86-interrupt" fn(_: &mut ExceptionStackFrame);
```

It's a [type alias] for an `extern "x86-interrupt" fn` type. The `extern` keyword defines a function with a [foreign calling convention] and is often used to communicate with C code (`extern "C" fn`). But what is the `x86-interrupt` calling convention?

[type alias]: https://doc.rust-lang.org/book/second-edition/ch19-04-advanced-types.html#type-aliases-create-type-synonyms
[foreign calling convention]: https://doc.rust-lang.org/book/first-edition/ffi.html#foreign-calling-conventions

## The Interrupt Calling Convention
Exceptions are quite similar to function calls: The CPU jumps to the first instruction of the called function and executes it. Afterwards the CPU jumps to the return address and continues the execution of the parent function.

However, there is a major difference between exceptions and function calls: A function call is invoked voluntary by a compiler inserted `call` instruction, while an exception might occur at _any_ instruction. In order to understand the consequences of this difference, we need to examine function calls in more detail.

[Calling conventions] specify the details of a function call. For example, they specify where function parameters are placed (e.g. in registers or on the stack) and how results are returned. On x86_64 Linux, the following rules apply for C functions (specified in the [System V ABI]):

[Calling conventions]: https://en.wikipedia.org/wiki/Calling_convention
[System V ABI]: http://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf

- the first six integer arguments are passed in registers `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`
- additional arguments are passed on the stack
- results are returned in `rax` and `rdx`

Note that Rust does not follow the C ABI (in fact, [there isn't even a Rust ABI yet][rust abi]), so these rules apply only to functions declared as `extern "C" fn`.

[rust abi]: https://github.com/rust-lang/rfcs/issues/600

### Preserved and Scratch Registers
The calling convention divides the registers in two parts: _preserved_ and _scratch_ registers.

The values of _preserved_ registers must remain unchanged across function calls. So a called function (the _“callee”_) is only allowed to overwrite these registers if it restores their original values before returning. Therefore these registers are called _“callee-saved”_. A common pattern is to save these registers to the stack at the function's beginning and restore them just before returning.

In contrast, a called function is allowed to overwrite _scratch_ registers without restrictions. If the caller wants to preserve the value of a scratch register across a function call, it needs to backup and restore it before the function call (e.g. by pushing it to the stack). So the scratch registers are _caller-saved_.

On x86_64, the C calling convention specifies the following preserved and scratch registers:

preserved registers | scratch registers
---|---
`rbp`, `rbx`, `rsp`, `r12`, `r13`, `r14`, `r15` | `rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`, `r9`, `r10`, `r11`
_callee-saved_ | _caller-saved_

The compiler knows these rules, so it generates the code accordingly. For example, most functions begin with a `push rbp`, which backups `rbp` on the stack (because it's a callee-saved register).

### Preserving all Registers
In contrast to function calls, exceptions can occur on _any_ instruction. In most cases we don't even know at compile time if the generated code will cause an exception. For example, the compiler can't know if an instruction causes a stack overflow or a page fault.

Since we don't know when an exception occurs, we can't backup any registers before. This means that we can't use a calling convention that relies on caller-saved registers for exception handlers. Instead, we need a calling convention means that preserves _all registers_. The `x86-interrupt` calling convention is such a calling convention, so it guarantees that all register values are restored to their original values on function return.

### The Exception Stack Frame
On a normal function call (using the `call` instruction), the CPU pushes the return address before jumping to the target function. On function return (using the `ret` instruction), the CPU pops this return address and jumps to it. So the stack frame of a normal function call looks like this:

![function stack frame](function-stack-frame.svg)

For exception and interrupt handlers, however, pushing a return address would not suffice, since interrupt handlers often run in a different context (stack pointer, CPU flags, etc.). Instead, the CPU performs the following steps when an interrupt occurs:

1. **Aligning the stack pointer**: An interrupt can occur at any instructions, so the stack pointer can have any value, too. However, some CPU instructions (e.g. some SSE instructions) require that the stack pointer is aligned on a 16 byte boundary, therefore the CPU performs such an alignment right after the interrupt.
2. **Switching stacks** (in some cases): A stack switch occurs when the CPU privilege level changes, for example when a CPU exception occurs in an user mode program. It is also possible to configure stack switches for specific interrupts using the so-called _Interrupt Stack Table_ (described in the next post).
3. **Pushing the old stack pointer**: The CPU pushes the values of the stack pointer (`rsp`) and the stack segment (`ss`) registers at the time when the interrupt occured (before the alignment). This makes it possible to restore the original stack pointer when returning from an interrupt handler.
4. **Pushing and updating the `RFLAGS` register**: The [`RFLAGS`] register contains various control and status bits. On interrupt entry, the CPU changes some bits and pushes the old value.
5. **Pushing the instruction pointer**: Before jumping to the interrupt handler function, the CPU pushes the instruction pointer (`rip`) and the code segment (`cs`). This is comparable to the return address push of a normal function call.
6. **Pushing an error code** (for some exceptions): For some specific exceptions such as page faults, the CPU pushes an error code, which describes the cause of the exception.
7. **Invoking the interrupt handler**: The CPU reads the address and the segment descriptor of the interrupt handler function from the corresponding field in the IDT. It then invokes this handler by loading the values into the `rip` and `cs` registers.

[`RFLAGS`]: https://en.wikipedia.org/wiki/FLAGS_register

So the _exception stack frame_ looks like this:

![exception stack frame](exception-stack-frame.svg)

In the `x86_64` crate, the exception stack frame is represented by the [`ExceptionStackFrame`] struct. It is passed to interrupt handlers as `&mut` and can be used to retrieve additional information about the exception's cause. The struct contains no error code field, since only some few exceptions push an error code. These exceptions use the separate [`HandlerFuncWithErrCode`] function type, which has an additional `error_code` argument.

[`ExceptionStackFrame`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/struct.ExceptionStackFrame.html

### Behind the Scenes
The `x86-interrupt` calling convention is a powerful abstraction that hides almost all of the messy details of the exception handling process. However, sometimes it's useful to know what's happening behind the curtain. Here is a short overview of the things that the `x86-interrupt` calling convention takes care of:

- **Retrieving the arguments**: Most calling conventions expect that the arguments are passed in registers. This is not possible for exception handlers, since we must not overwrite any register values before backing them up on the stack. Instead, the `x86-interrupt` calling convention is aware that the arguments already lie on the stack at a specific offset.
- **Returning using `iretq`**: Since the exception stack frame completely differs from stack frames of normal function calls, we can't return from handlers functions through the normal `ret` instruction. Instead, the `iretq` instruction must be used.
- **Handling the error code**: The error code, which is pushed for some exceptions, makes things much more complex. It changes the stack alignment (see the next point) and needs to be popped off the stack before returning. The `x86-interrupt` calling convention handles all that complexity. However, it doesn't know which handler function is used for which exception, so it needs to deduce that information from the number of function arguments. That means that the programmer is still responsible to use the correct function type for each exception. Luckily, the `InterruptDescriptorTable` type defined by the `x86_64` crate ensures that the correct function types are used.
- **Aligning the stack**: There are some instructions (especially SSE instructions) that require a 16-byte stack alignment. The CPU ensures this alignment whenever an exception occurs, but for some exceptions it destroys it again later when it pushes an error code. The `x86-interrupt` calling convention takes care of this by realigning the stack in this case.

If you are interested in more details: We also have a series of posts that explains exception handling using [naked functions] linked [at the end of this post][too-much-magic].

[naked functions]: https://github.com/rust-lang/rfcs/blob/master/text/1201-naked-fns.md
[too-much-magic]: #too-much-magic

## Implementation
Now that we've understood the theory, it's time to handle CPU exceptions in our kernel. We'll start by creating a new interrupts module in `src/interrupts.rs`, that first creates an `init_idt` function that creates a new `InterruptDescriptorTable`:

``` rust
// in src/lib.rs

pub mod interrupts;

// in src/interrupts.rs

extern crate x86_64;
use x86_64::structures::idt::InterruptDescriptorTable;

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
}
```

Now we can add handler functions. We start by adding a handler for the [breakpoint exception]. The breakpoint exception is the perfect exception to test exception handling. Its only purpose is to temporarily pause a program when the breakpoint instruction `int3` is executed.

[breakpoint exception]: http://wiki.osdev.org/Exceptions#Breakpoint

The breakpoint exception is commonly used in debuggers: When the user sets a breakpoint, the debugger overwrites the corresponding instruction with the `int3` instruction so that the CPU throws the breakpoint exception when it reaches that line. When the user wants to continue the program, the debugger replaces the `int3` instruction with the original instruction again and continues the program. For more details, see the ["_How debuggers work_"] series.

["_How debuggers work_"]: http://eli.thegreenplace.net/2011/01/27/how-debuggers-work-part-2-breakpoints

For our use case, we don't need to overwrite any instructions. Instead, we just want to print a message when the breakpoint instruction is executed and then continue the program. So let's create a simple `breakpoint_handler` function and add it to our IDT:

```rust
// in src/interrupts.rs

use x86_64::structures::idt::{InterruptDescriptorTable, ExceptionStackFrame};

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
    idt.breakpoint.set_handler_fn(breakpoint_handler);
}

extern "x86-interrupt" fn breakpoint_handler(
    stack_frame: &mut ExceptionStackFrame)
{
    println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
}
```

Our handler just outputs a message and pretty-prints the exception stack frame.

When we try to compile it, the following errors occur:

```
error[E0658]: x86-interrupt ABI is experimental and subject to change (see issue #40180)
  --> src/main.rs:53:1
   |
53 | / extern "x86-interrupt" fn breakpoint_handler(stack_frame: &mut ExceptionStackFrame) {
54 | |     println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
55 | | }
   | |_^
   |
   = help: add #![feature(abi_x86_interrupt)] to the crate attributes to enable
```

This error occurs because the `x86-interrupt` calling convention is still unstable. To use it anyway, we have to explicitly enable it by adding `#![feature(abi_x86_interrupt)]` on the top of our `lib.rs`.

```
error: cannot find macro `println!` in this scope
  --> src/interrupts.rs:40:5
   |
40 |     println!("EXCEPTION: BREAKPOINT\n{:#?}", stack_frame);
   |     ^^^^^^^
   |
   = help: have you added the `#[macro_use]` on the module/import?
```

This happened because we forgot to add `#[macro_use]` before our import of the `vga_buffer` module.

```rust
// in src/lib.rs

pub mod gdt;
pub mod interrupts;
pub mod serial;
#[macro_use] // new
pub mod vga_buffer;
```

However, after adding `#[macro_use]` before the module import, we still get the same error. Sometimes this can be confusing, but it's actually a quirk of how Rust's macro system works. Macros _must be defined_ before you can use them. This is one case where import order matters in Rust. We can easily fix this by ensuring the order of our imports places the macros first:

```rust
// in src/lib.rs

#[macro_use]
pub mod vga_buffer; // import this before other modules so its macros may be used
pub mod gdt;
pub mod interrupts;
pub mod serial;
```

Now we can use our `print!` and `println!` macros in `interrupts.rs`. If you'd like to know more about the ins and outs of macros and how they differ from functions [you can find more information here][in-depth-rust-macros].

[in-depth-rust-macros]: https://doc.rust-lang.org/book/second-edition/appendix-04-macros.html

### Loading the IDT
In order that the CPU uses our new interrupt descriptor table, we need to load it using the [`lidt`] instruction. The `InterruptDescriptorTable` struct of the `x86_64` provides a [`load`][InterruptDescriptorTable::load] method function for that. Let's try to use it:

[`lidt`]: http://x86.renejeschke.de/html/file_module_x86_id_156.html
[InterruptDescriptorTable::load]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/struct.InterruptDescriptorTable.html#method.load

```rust
// in src/interrupts.rs

pub fn init_idt() {
    let mut idt = InterruptDescriptorTable::new();
    idt.breakpoint.set_handler_fn(breakpoint_handler);
    idt.load();
}
```

When we try to compile it now, the following error occurs:

```
error: `idt` does not live long enough
  --> src/interrupts/mod.rs:43:5
   |
43 |     idt.load();
   |     ^^^ does not live long enough
44 | }
   | - borrowed value only lives until here
   |
   = note: borrowed value must be valid for the static lifetime...
```

So the `load` methods expects a `&'static self`, that is a reference that is valid for the complete runtime of the program. The reason is that the CPU will access this table on every interrupt until we load a different IDT. So using a shorter lifetime than `'static` could lead to use-after-free bugs.

In fact, this is exactly what happens here. Our `idt` is created on the stack, so it is only valid inside the `init` function. Afterwards the stack memory is reused for other functions, so the CPU would interpret random stack memory as IDT. Luckily, the `InterruptDescriptorTable::load` method encodes this lifetime requirement in its function definition, so that the Rust compiler is able to prevent this possible bug at compile time.

In order to fix this problem, we need to store our `idt` at a place where it has a `'static` lifetime. To achieve this we could allocate our IDT on the heap using [`Box`] and then convert it to a `'static` reference, but we are writing an OS kernel and thus don't have a heap (yet).

[`Box`]: https://doc.rust-lang.org/std/boxed/struct.Box.html


As an alternative we could try to store the IDT as a `static`:

```rust
static IDT: InterruptDescriptorTable = InterruptDescriptorTable::new();

pub fn init_idt() {
    IDT.breakpoint.set_handler_fn(breakpoint_handler);
    IDT.load();
}
```

However, there is a problem: Statics are immutable, so we can't modify the breakpoint entry from our `init` function. We could solve this problem by using a [`static mut`]:

[`static mut`]: https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#accessing-or-modifying-a-mutable-static-variable

```rust
static mut IDT: InterruptDescriptorTable = InterruptDescriptorTable::new();

pub fn init_idt() {
    unsafe {
        IDT.breakpoint.set_handler_fn(breakpoint_handler);
        IDT.load();
    }
}
```

This variant compiles without errors but it's far from idiomatic. `static mut`s are very prone to data races, so we need an [`unsafe` block] on each access.

[`unsafe` block]: https://doc.rust-lang.org/book/second-edition/ch19-01-unsafe-rust.html#unsafe-superpowers

#### Lazy Statics to the Rescue
Fortunately the `lazy_static` macro exists. Instead of evaluating a `static` at compile time, the macro performs the initialization when the `static` is referenced the first time. Thus, we can do almost everything in the initialization block and are even able to read runtime values.

We already imported the `lazy_static` crate when we [created an abstraction for the VGA text buffer][vga text buffer lazy static]. So we can directly use the `lazy_static!` macro to create our static IDT:

[vga text buffer lazy static]: ./second-edition/posts/03-vga-text-buffer/index.md#lazy-statics

```rust
// in src/interrupts.rs

lazy_static! {
    static ref IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        idt
    };
}

pub fn init_idt() {
    IDT.load();
}
```

Note how this solution requires no `unsafe` blocks. The `lazy_static!` macro does use `unsafe` behind the scenes, but it is abstracted away in a safe interface.

### Testing it
Now we should be able to handle breakpoint exceptions! Let's try it in our `_start` function:

```rust
// in src/main.rs

#[cfg(not(test))]
#[no_mangle]
pub extern "C" fn _start() -> ! {
    println!("Hello World{}", "!");

    blog_os::interrupts::init_idt();

    // invoke a breakpoint exception
    x86_64::instructions::int3();

    println!("It did not crash!");
    loop {}
}
```

When we run it in QEMU now (using `bootimage run`), we see the following:

![QEMU printing `EXCEPTION: BREAKPOINT` and the exception stack frame](qemu-breakpoint-exception.png)

It works! The CPU successfully invokes our breakpoint handler, which prints the message, and then returns back to the `_start` function, where the `It did not crash!` message is printed.

We see that the exception stack frame tells us the instruction and stack pointers at the time when the exception occured. This information is very useful when debugging unexpected exceptions.

### Adding a Test

Let's create an integration test that ensures that the above continues to work. For that we create a file named `test-exception-breakpoint.rs`:

```rust
// in src/bin/test-exception-breakpoint.rs

[…]
use core::sync::atomic::{AtomicUsize, Ordering};

static BREAKPOINT_HANDLER_CALLED: AtomicUsize = AtomicUsize::new(0);

#[no_mangle]
pub extern "C" fn _start() -> ! {
    init_test_idt();

    // invoke a breakpoint exception
    x86_64::instructions::int3();

    match BREAKPOINT_HANDLER_CALLED.load(Ordering::SeqCst) {
        1 => serial_println!("ok"),
        0 => {
            serial_println!("failed");
            serial_println!("Breakpoint handler was not called.");
        }
        other => {
            serial_println!("failed");
            serial_println!("Breakpoint handler was called {} times", other);
        }
    }

    unsafe { exit_qemu(); }
    loop {}
}


lazy_static! {
    static ref TEST_IDT: InterruptDescriptorTable = {
        let mut idt = InterruptDescriptorTable::new();
        idt.breakpoint.set_handler_fn(breakpoint_handler);
        idt
    };
}

pub fn init_test_idt() {
    TEST_IDT.load();
}

extern "x86-interrupt" fn breakpoint_handler(
    _stack_frame: &mut ExceptionStackFrame)
{
    BREAKPOINT_HANDLER_CALLED.fetch_add(1, Ordering::SeqCst);
}

[…]
```

For space reasons we don't show the full content here. You can find the full file [on Github](https://github.com/phil-opp/blog_os/blob/master/src/bin/test-exception-breakpoint.rs).

It is similar to our `main.rs`, but uses a custom IDT called `TEST_IDT` and different `_start` and `breakpoint_handler` functions. The most interesting part is the `BREAKPOINT_HANDLER_CALLED` static. It is an [`AtomicUsize`], an integer type that can be safely concurrently modified because all of its operations are atomic. We increment it when the `breakpoint_handler` is called and verify in our `_start` function that the handler was called exactly once.

[`AtomicUsize`]: https://doc.rust-lang.org/core/sync/atomic/struct.AtomicUsize.html

The [`Ordering`] parameter specifies the desired guarantees of the atomic operations. The `SeqCst` ordering means “sequential consistent” and gives the strongest guarantees. It is a good default, because weaker orderings can have undesired effects.

[`Ordering`]: https://doc.rust-lang.org/core/sync/atomic/enum.Ordering.html

### Fixing `cargo test` on Windows

The `x86-interrupt` calling convention has an annoying problem: There is a bug in LLVM that leads to a "LLVM ERROR: offset is not a multiple of 16" when compiling a function with the `x86-interrupt` calling convention _for a Windows target_. Normally this is no problem, since we only compile for our custom `x86_64-blog_os.json` target. But `cargo test` compiles our crate for the host system, so the error occurs if the host system is Windows.

To fix this problem, we add a conditional compilation attribute, so that the `x86-interrupt` functions are not compiled on Windows systems. We don't have any unit tests that rely on the `interrupts` module, so we can simply skip compilation of the whole module:

```rust
// in src/interrupts.rs

// LLVM throws an error if a function with the
// x86-interrupt calling convention is compiled
// for a Windows system.
#![cfg(not(windows))]
```

The bang ("!") after the hash ("#") indicates that this is an [inner attribute] and applies to the module we're in. Without the bang it would only apply to the next item in the file. Note that inner attributes must be right at the beginning of a module.

[inner attribute]: https://doc.rust-lang.org/reference/attributes.html

We could achieve the same effect by using an _outer_ attribute (without a bang) in our `lib.rs`:

```rust
// in src/lib.rs

#[cfg(not(windows))] // no bang ("!") after the hash ("#")
pub mod interrupts;
```

Both variants have the exact same effects, so it comes down to personal preference which one to use. I prefer the inner attribute in this case because it does not clutter our `lib.rs` with a workaround for a LLVM bug, but either way is fine.

## Too much Magic?
The `x86-interrupt` calling convention and the [`InterruptDescriptorTable`] type made the exception handling process relatively straightforward and painless. If this was too much magic for you and you like to learn all the gory details of exception handling, we got you covered: Our [“Handling Exceptions with Naked Functions”] series shows how to handle exceptions without the `x86-interrupt` calling convention and also creates its own IDT type. Historically, these posts were the main exception handling posts before the `x86-interrupt` calling convention and the `x86_64` crate existed. Note that these posts are based on the [first edition] of this blog and might be out of date.

[“Handling Exceptions with Naked Functions”]: ./first-edition/extra/naked-exceptions/_index.md
[`InterruptDescriptorTable`]: https://docs.rs/x86_64/0.2.8/x86_64/structures/idt/struct.InterruptDescriptorTable.html
[first edition]: ./first-edition/_index.md

## What's next?
We've successfully caught our first exception and returned from it! The next step is to ensure that we catch all exceptions, because an uncaught exception causes a fatal [triple fault], which leads to a system reset. The next post explains how we can avoid this by correctly catching [double faults].

[triple fault]: http://wiki.osdev.org/Triple_Fault
[double faults]: http://wiki.osdev.org/Double_Fault#Double_Fault
