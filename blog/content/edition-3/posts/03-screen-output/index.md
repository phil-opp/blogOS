+++
title = "Screen Output"
weight = 3
path = "screen-output"
date = 0000-01-01
draft = true

[extra]
chapter = "Basic I/O"
icon = '''<svg xmlns="http://www.w3.org/2000/svg" fill="currentColor" class="bi bi-display" viewBox="0 0 16 16">
  <path d="M0 4s0-2 2-2h12s2 0 2 2v6s0 2-2 2h-4c0 .667.083 1.167.25 1.5H11a.5.5 0 0 1 0 1H5a.5.5 0 0 1 0-1h.75c.167-.333.25-.833.25-1.5H2s-2 0-2-2V4zm1.398-.855a.758.758 0 0 0-.254.302A1.46 1.46 0 0 0 1 4.01V10c0 .325.078.502.145.602.07.105.17.188.302.254a1.464 1.464 0 0 0 .538.143L2.01 11H14c.325 0 .502-.078.602-.145a.758.758 0 0 0 .254-.302 1.464 1.464 0 0 0 .143-.538L15 9.99V4c0-.325-.078-.502-.145-.602a.757.757 0 0 0-.302-.254A1.46 1.46 0 0 0 13.99 3H2c-.325 0-.502.078-.602.145z"/>
</svg>'''
+++

In this post we focus on the [framebuffer], a special memory region that controls the screen output.
Using an [external crate], we will create functions for writing individual pixels, lines, and various shapes.
In the the second half of this post, we will explore text rendering and learn how to print the obligatory _["Hello, World!"]_.

[framebuffer]: https://en.wikipedia.org/wiki/Framebuffer
[external crate]: https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html
["Hello, World!"]: https://en.wikipedia.org/wiki/Hello_world

<!-- more -->

This blog is openly developed on [GitHub].
If you have any problems or questions, please open an issue there.
You can also leave comments [at the bottom].
The complete source code for this post can be found in the [`post-3.3`][post branch] branch.

[GitHub]: https://github.com/phil-opp/blog_os
[at the bottom]: #comments
<!-- fix for zola anchor checker (target is in template): <a id="comments"> -->
[post branch]: https://github.com/phil-opp/blog_os/tree/post-3.3

<!-- toc -->

## Bitmap Images

In the [previous post], we learned how to make our minimal kernel bootable.
Using the [`BootInfo`] provided by the bootloader, we were able to access a special memory region called the _[framebuffer]_, which controls the screen output.
We wrote some example code to display a gray background:

[previous post]: @/edition-3/posts/02-booting/index.md
[`BootInfo`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/struct.BootInfo.html

```rust
// in kernel/src/main.rs

fn kernel_main(boot_info: &'static mut BootInfo) -> ! {
    if let Some(framebuffer) = boot_info.framebuffer.as_mut() {
        for byte in framebuffer.buffer_mut() {
            *byte = 0x90;
        }
    }
    loop {}
}
```

The reason that the above code affects the screen output is because the graphics card interprets the framebuffer memory as a [bitmap] image.
A bitmap describes an image through a predefined number of bytes per pixel.
The pixels are laid out line by line, typically starting at the top.

[bitmap]: https://en.wikipedia.org/wiki/Bitmap
[RGB]: https://en.wikipedia.org/wiki/Rgb

For example, the pixels of an image with width 10 and height 3 would be typically stored in this order:

<table style = "width: fit-content;"><tbody>
  <tr><td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td></tr>
  <tr><td>10</td><td>11</td><td>12</td><td>13</td><td>14</td><td>15</td><td>16</td><td>17</td><td>18</td><td>19</td></tr>
  <tr><td>20</td><td>21</td><td>22</td><td>23</td><td>24</td><td>25</td><td>26</td><td>27</td><td>28</td><td>29</td></tr>
</tbody></table>

So top left pixel is stored at offset 0 in the bitmap array.
The pixel on its right is at pixel offset 1.
The first pixel of the next line starts at pixel offset `line_length`, which is 10 in this case.
The last line starts at pixel offset 20, which is `line_length * 2`.

### Padding

Depending on the hardware and GPU firmware, it is often more efficient to make lines start at well-aligned offsets.
Because of this, there is often some additional padding at the end of each line.
So the actual memory layout of the 10x3 example image might look like this, with the padding marked as yellow:

<table style = "width: fit-content;"><tbody>
  <tr><td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td style="background-color:yellow;">10</td><td style="background-color:yellow;">11</td></tr>
  <tr><td>12</td><td>13</td><td>14</td><td>15</td><td>16</td><td>17</td><td>18</td><td>19</td><td>20</td><td>21</td><td style="background-color:yellow;">22</td><td style="background-color:yellow;">23</td></tr>
  <tr><td>24</td><td>25</td><td>26</td><td>27</td><td>28</td><td>29</td><td>30</td><td>31</td><td>32</td><td>33</td><td style="background-color:yellow;">34</td><td style="background-color:yellow;">35</td></tr>
</tbody></table>

So now the second line starts at pixel offset 12.
The two pixels at the end of each line are considered as padding and ignored.
So if we want to set the first pixel of the second line, we need to be aware of the additional padding and set the pixel at offset 12 instead of offset 10.

The line length plus the padding bytes is typically called the _stride_ or _pitch_ of the buffer.
In the example above, the stride is 12 and the line length is 10.

Since the amount of padding depends on the hardware, the stride is only known at runtime.
The `bootloader` crate queries the framebuffer parameters from the UEFI or BIOS firmware and reports them as part of the `BootInfo`.
It provides the stride of the framebuffer, among other parameters, in form of a [`FrameBufferInfo`] struct that can be created using the [`FrameBuffer::info`] method.

[`FrameBufferInfo`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/struct.FrameBufferInfo.html
[`FrameBuffer::info`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/struct.FrameBuffer.html#method.info

### Color formats

The [`FrameBufferInfo`] also specifies the [`PixelFormat`] of the framebuffer, which also depends on the underlying hardware.
Using this information, we can set pixels to different colors.
For example, the [`PixelFormat::Rgb`] variant specifies that each pixel is represented in the [RGB color space], which stores the red, green, and blue parts of the pixel as separate bytes.
In this model, the color red would be represented as the three bytes `[255, 0, 0]`, or `0xff0000` in [hexadecimal representation].
The color yellow is represented the addition of red and green, which results in `[255, 255, 0]` (or `0xffff00` in hexadecimal representation).

[`PixelFormat`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/enum.PixelFormat.html
[`PixelFormat::Rgb`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/enum.PixelFormat.html#variant.Rgb
[RGB color space]: https://en.wikipedia.org/wiki/RGB_color_spaces
[hexadecimal representation]: https://en.wikipedia.org/wiki/RGB_color_model#Numeric_representations

While the `Rgb` format is most common, there are also framebuffers that use a different color format.
For example, the [`PixelFormat::Bgr`] stores the three colors in inverted order, i.e. blue first and red last.
There are also buffers that don't support colors at all and can represent only grayscale pixels.
The `bootloader_api` crate reports such buffers as [`PixelFormat::U8`].

[`PixelFormat::Bgr`]: https://docs.rs/bootloader_api/0.11.5/bootloader_api/info/enum.PixelFormat.html#variant.Bgr
[`PixelFormat::U8`]: https://docs.rs/bootloader_api/0.11.5/bootloader_api/info/enum.PixelFormat.html#variant.U8

Note that there might be some additional padding at the pixel-level as well.
For example, an `Rgb` pixel might be stored as 4 bytes instead of 3 to ensure 32-bit alignment.
The number of bytes per pixel is reported by the bootloader in the [`FrameBufferInfo::bytes_per_pixel`] field.

[`FrameBufferInfo::bytes_per_pixel`]: https://docs.rs/bootloader_api/0.11/bootloader_api/info/struct.FrameBufferInfo.html#structfield.bytes_per_pixel

## Setting specific Pixels

Based on this above details, we can now create a function to set a specific pixel to a certain color.
We start by creating a new `framebuffer` [module]:

[module]: https://doc.rust-lang.org/book/ch07-02-defining-modules-to-control-scope-and-privacy.html

```rust ,hl_lines=3-5
// in kernel/src/main.rs

// declare a submodule -> the compiler will automatically look
// for a file named `framebuffer.rs` or `framebuffer/mod.rs`
mod framebuffer;
```

In the new module, we create basic structs for representing pixel positions and colors:

```rust ,hl_lines=3-16
// in new kernel/src/framebuffer.rs file

use bootloader_api::info::FrameBuffer;

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Position {
    pub x: usize,
    pub y: usize,
}

impl From<Point> for Position {
    fn from(value: Point) -> Self {
        Self {
            x: value.x as usize,
            y: value.y as usize,
        }
    }
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Color {
    pub red: u8,
    pub green: u8,
    pub blue: u8,
}
```

By marking the structs and their fields as `pub`, we make them accessible from the parent `kernel` module.
We use the `#[derive]` attribute to implement the [`Debug`], [`Clone`], [`Copy`], [`PartialEq`], and [`Eq`] traits of Rust's core library.
These traits allow us to duplicate, compare, and print the structs.

[`Debug`]: https://doc.rust-lang.org/stable/core/fmt/trait.Debug.html
[`Clone`]: https://doc.rust-lang.org/stable/core/clone/trait.Clone.html
[`Copy`]: https://doc.rust-lang.org/stable/core/marker/trait.Copy.html
[`PartialEq`]: https://doc.rust-lang.org/stable/core/cmp/trait.PartialEq.html
[`Eq`]: https://doc.rust-lang.org/stable/core/cmp/trait.Eq.html

Next, we create a function for setting a specific pixel in the framebuffer to a given color:

```rust ,hl_lines=3 5-39
// in new kernel/src/framebuffer.rs file

use bootloader_api::info::PixelFormat;

pub fn set_pixel_in(framebuffer: &mut FrameBuffer, position: Position, color: Color) {
    let info = framebuffer.info();

    // calculate offset to first byte of pixel
    let byte_offset = {
        // use stride to calculate pixel offset of target line
        let line_offset = position.y * info.stride;
        // add x position to get the absolute pixel offset in buffer
        let pixel_offset = line_offset + position.x;
        // convert to byte offset
        pixel_offset * info.bytes_per_pixel
    };

    // set pixel based on color format
    let pixel_buffer = &mut framebuffer.buffer_mut()[byte_offset..];
    match info.pixel_format {
        PixelFormat::Rgb => {
            pixel_buffer[0] = color.red;
            pixel_buffer[1] = color.green;
            pixel_buffer[2] = color.blue;
        }
        PixelFormat::Bgr => {
            pixel_buffer[0] = color.blue;
            pixel_buffer[1] = color.green;
            pixel_buffer[2] = color.red;
        }
        PixelFormat::U8 => {
            // use a simple average-based grayscale transform
            let gray = color.red / 3 + color.green / 3 + color.blue / 3;
            pixel_buffer[0] = gray;
        }
        other => panic!("unknown pixel format {other:?}"),
    }
}
```

The first step is to calculate the byte offset within the framebuffer slice at which the pixel starts.
For this, we first calculate the pixel offset of the line by multiplying the `y` position with the stride of the framebuffer, i.e. its line width plus the line padding.
We then add the `x` position to get the absolute index of the pixel.
As the framebuffer slice is a byte slice, we need to transform the pixel index to a byte offset by multiplying it with the number of `bytes_per_pixel`.

[`FrameBuffer::buffer_mut`]: https://docs.rs/bootloader_api/0.11.5/bootloader_api/info/struct.FrameBuffer.html#method.buffer_mut

The second step is to set the pixel to the desired color.
We first use the [`FrameBuffer::buffer_mut`] method to get access to the actual bytes of the framebuffer in form of a slice.
Then, we use the slicing operator `[byte_offset..]` to get a sub-slice starting at the `byte_offset` of the target pixel.
As the write operation depends on the pixel format, we use a [`match`] statement:

[`match`]: https://doc.rust-lang.org/stable/std/keyword.match.html

- For `Rgb` framebuffers, we write three bytes; first red, then green, then blue.
- For `Bgr` framebuffers, we also write three bytes, but blue first and red last.
- For `U8` framebuffers, we first convert the color to grayscale by taking the average of the three color channels.
  Note that there are multiple [different ways to convert colors to grayscale], so you can also use different factors here.
- For all other framebuffer formats, we [panic] for now.

[different ways to convert colors to grayscale]: https://www.baeldung.com/cs/convert-rgb-to-grayscale#bd-convert-rgb-to-grayscale
[panic]: https://doc.rust-lang.org/stable/core/macro.panic.html

Let's try to use our new function to write a blue pixel in our `kernel_main` function:

```rust ,hl_lines=5-11
// in kernel/src/main.rs

fn kernel_main(boot_info: &'static mut BootInfo) -> ! {
    if let Some(framebuffer) = boot_info.framebuffer.as_mut() {
        let position = framebuffer::Position { x: 20, y: 100 };
        let color = framebuffer::Color {
            red: 0,
            green: 0,
            blue: 255,
        };
        framebuffer::set_pixel_in(framebuffer, position, color);
    }
    loop {}
}
```

When we run our code in QEMU using `cargo run --bin qemu-bios` (or `--bin qemu-uefi`) and look _very closely_, we can see the blue pixel.
It's really difficult to see, so I marked with an arrow below:

![QEMU the bootloader text output with one pixel set to blue. An annotated arrow points to the pixel](qemu-blue-pixel.png)

As this single pixel is too difficult to see, let's draw a filled square of 100x100 pixels instead:

```rust ,hl_lines=10-18
// in kernel/src/main.rs

fn kernel_main(boot_info: &'static mut BootInfo) -> ! {
    if let Some(framebuffer) = boot_info.framebuffer.as_mut() {
        let color = framebuffer::Color {
            red: 0,
            green: 0,
            blue: 255,
        };
        for x in 0..100 {
            for y in 0..100 {
                let position = framebuffer::Position {
                    x: 20 + x,
                    y: 100 + y,
                };
                framebuffer::set_pixel_in(framebuffer, position, color);
            }
        }
    }
    loop {}
}
```

Now we clearly see that our code works as intended:

![QEMU showing a blue square above the bootloader text output](qemu-blue-square.png)

Feel free to experiment with different positions and colors if you like.
You can also try to draw a circle instead of a square, or a line with a certain thickness.

As you can probably imagine, it would be a lot of work to draw more complex shapes this way.
One example for such complex shapes is _text_, i.e. the rendering of letters and punctuation.
Fortunately, there is the nice `no_std`-compatible [`embedded-graphics`] crate, which provides draw functions for text, various shapes, and image data.

[`embedded-graphics`]: https://docs.rs/embedded-graphics/latest/embedded_graphics/index.html

## The `embedded-graphics` crate



### Implementing `DrawTarget`

```rust ,hl_lines=3
// in kernel/src/framebuffer.rs
use embedded_graphics::geometry::Point;
use embedded_graphics::pixelcolor::Rgb888;
use embedded_graphics::prelude::OriginDimensions;
use embedded_graphics::prelude::Pixel;
use embedded_graphics::prelude::Size;

pub struct Display {
    framebuffer: &'static mut FrameBuffer,
}

impl Display {
    pub fn new(framebuffer: &'static mut FrameBuffer) -> Display {
        Self { framebuffer }
    }

    fn in_range(&mut self, point: Point) -> bool {
        let (width, height) = {
            let info = self.framebuffer.info();
            (info.width, info.height)
        };

        if point.x < width as i32 && point.x > 0 && point.y < height as i32 && point.y > 0 {
            true
        } else {
            false
        }
    }

    pub fn clear(&mut self) {
        self.framebuffer.buffer_mut().fill(0);
    }

    fn draw_pixel(&mut self, Pixel(point, color): Pixel<Rgb888>) {
        use embedded_graphics::prelude::RgbColor;

        if self.in_range(point) {
            let color = Color {
                red: color.r(),
                green: color.g(),
                blue: color.b(),
            };
            set_pixel_in(&mut self.framebuffer, Position::from(point), color)
        }
    }
}

impl embedded_graphics::draw_target::DrawTarget for Display {
    type Color = Rgb888;

    /// Drawing operations can never fail.
    type Error = core::convert::Infallible;

    fn draw_iter<I>(&mut self, pixels: I) -> Result<(), Self::Error>
    where
        I: IntoIterator<Item = Pixel<Self::Color>>,
    {
        for pixel in pixels.into_iter() {
            self.draw_pixel(pixel);
        }

        Ok(())
    }
}

impl OriginDimensions for Display {
    fn size(&self) -> Size {
        let width: usize = self.framebuffer.info().width.try_into().unwrap();
        let height: usize = self.framebuffer.info().height.try_into().unwrap();
        Size::new(width as u32, height as u32)
    }
}
```

```rust ,hl_lines=3
// in kernel/src/main.rs
use kernel::framebuffer::Display;

use embedded_graphics::{
    mono_font::{ascii::FONT_9X18, MonoTextStyle},
    pixelcolor::Rgb888,
    prelude::*,
    text::Text,
};

entry_point!(kernel_main);
fn kernel_main(boot_info: &'static mut BootInfo) -> ! {
    let mut display = Display::new(boot_info.framebuffer.as_mut().unwrap());

    // clear the log
    display.clear();

    let style = MonoTextStyle::new(&FONT_9X18, Rgb888::WHITE);

    // Create a text at position (20, 30) and draw it using the previously defined style
    Text::new("Hello Rust!", Point::new(20, 30), style).draw(&mut display).unwrap();

    loop {}
}
```

---






  draw shapes and pixels directly onto the framebuffer. That's fine and all, but how is one able to go from that to displaying text on the screen? Understanding this requires taking a deep dive into how characters are rendered behind the scenes.

When a key is pressed on the keyboard, it sends a character code to the CPU. It's the CPU's job at that point to then interpret the character code and match it with an image to draw on the screen. The image is then sent to either the GPU or the framebuffer (the latter in our case) to be drawn on the screen, and the user sees that image as a letter, number, CJK character, emoji, or whatever else he or she wanted to have displayed by pressing that key.

In most other programming languages, implementing this behind the scenes can be a daunting task. With Rust, however, we have a toolset at our disposal that can pave the way for setting up proper framebuffer logging using very little code of our own.

# The `log` crate

Rust developers used to writing user-mode code will recognize the `log` crate from a mile away:

```toml
# in Cargo.toml
[dependencies]
log = { version = "0.4.17", default-features = false }
```

This crate has both a set of macros for logging either to the console or to a log file for later reading and a trait — also called `Log` with a capital L — that can be implemented to provide a backend, called a `Logger` in Rust parlance. Loggers are provided by a myriad of crates for a wide variety of use cases, and some of them even run on bare metal. We already used one such extant logger in the UEFI booting module when we used the logger provided by the `uefi` crate to print text to the UEFI console. That won't work in the kernel, however, because UEFI boot services need to be active in order for the UEFI logger to be usable.

If you were paying attention to the post before that one, however, you may have noticed that the bootloader is itself able to log directly to the framebuffer as it did when we booted the barebones kernel for the first time, and unlike the UEFI console logger, this logger is usable long after UEFI boot services are exited. It's this logger, therefore, that provides the easiest means of implementation on our end.

## `bootloader-x86_64-common`

In version 0.11.x of the bootloader crate, each component is separate, unlike in 0.10.x where the bootloader was a huge monolith. This is fantastic as it means that a lot of the APIs that the bootloader uses behind the scenes are also free for kernels to use, including, of course, the logger. The set of APIs that the logger belongs to are in a crate called `bootloader-x86_64-common` which also contains some other useful abstractions related to things like memory management that will come in handy later:

```toml
# in Cargo.toml
[dependencies]
bootloader-x86_64-common = "0.11.3"
```

For now, however, only the logger will be used. If you are curious as to how this logger is written behind the scenes, however, don't worry; a sub-module of this chapter will include a tutorial on how to write a custom logger from scratch.

# Putting it all together

Before we use the bootloader's logger, we first need to initialize it. This requires creating a static instance, since it needs to live for as long as the kernel lives — which would mean for as long as the computer is powered on. Unfortunately, this is easier said than done, as Rust statics can be rather finicky — understandably so for security reasons. Luckily, there's a crate for this too.

## The `conquer_once` crate

Those used to using the standard library know that it provides a `OnceCell` which is exactly what it sounds like: you write to it only once, and then after that it's just there to use whenever. We're in a kernel and don't have access to the standard library, however, so is there a crate on crates.io that provides a replacement? Ah, yes there is:

```toml
# in Cargo.toml
[dependencies]
conquer-once = { version = "0.4.0", default-features = false }
```

Note that we need to add `default-features = false` to our `conquer-once` dependency —that's because the [`conquer-once` crate](https://crates.io/crates/conquer-once) tries to pull in the standard library by default, which in the kernel will result in compilation errors.

Now that we've added our two dependencies, it's time to use them:

```rust
// in src/main.rs
use conquer_once::spin::OnceCell;
use bootloader_x86_64_common::logger::LockedLogger;
// ...
pub(crate) static LOGGER: OnceCell<LockedLogger> = OnceCell::uninit();
```

By setting the logger up as a static `OnceCell` it becomes much easier to initialize. We use `pub(crate)` to ensure that the kernel can see it but nothing else can.

After this, it's time to actually initialize it. To do that, we use a function:

```rust
// in src/main.rs
use bootloader_api::info::FrameBufferInfo;
// ...
pub(crate) fn init_logger(buffer: &'static mut [u8], info: FrameBufferInfo) {
    let logger = LOGGER.get_or_init(move || LockedLogger::new(buffer, info, true, false));
    log::set_logger(logger).expect("Logger already set");
    log::set_max_level(log::LevelFilter::Trace);
    log::info!("Hello, Kernel Mode!");
}
```

This function takes two parameters: a byte slice representing a raw framebuffer and a `FrameBufferInfo` structure containing information about the first parameter. Getting those parameters, however, requires jumping through some hoops to satisfy the borrow checker:

```rust
// in src/main.rs
fn kernel_main(boot_info: &'static mut bootloader_api::BootInfo) -> ! {
    // ...
    // free the doubly wrapped framebuffer from the boot info struct
    let frame_buffer_optional = &mut boot_info.framebuffer;
    
    // free the wrapped framebuffer from the FFI-safe abstraction provided by bootloader_api
    let frame_buffer_option = frame_buffer_optional.as_mut();
    
    // unwrap the framebuffer
    let frame_buffer_struct = frame_buffer_option.unwrap();
    
    // extract the framebuffer info and, to satisfy the borrow checker, clone it
    let frame_buffer_info = frame_buffer.info().clone();
    
    // get the framebuffer's mutable raw byte slice
    let raw_frame_buffer = frame_buffer_struct.buffer_mut();
    
    // finally, initialize the logger using the last two variables
    init_logger(raw_frame_buffer, frame_buffer_info);
    // ...
}
```

Any one of these steps, if skipped, will cause the borrow checker to throw a hissy fit due to the use of the `move ||` closure by the initializer function. With this, however, you're done, and you'll know the logger has been initialized when you see "Hello, Kernel Mode!" printed on the screen.

<!-- more -->
<!-- toc -->

<!-- TODO: update relative link in 02-booting/uefi/index.md when this post is finished -->
