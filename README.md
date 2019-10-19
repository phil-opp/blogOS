# Blog OS

[![Build Status](https://travis-ci.org/phil-opp/blog_os.svg?branch=master)](https://travis-ci.org/phil-opp/blog_os)
[![Build Status](https://ci.appveyor.com/api/projects/status/phil-opp/blog_os)]()
[![Gitter](https://badges.gitter.im/phil-opp/blog_os.svg)](https://gitter.im/phil-opp/blog_os?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)
![GitHub code size in bytes](https://img.shields.io/github/languages/code-size/phil-opp/blog_os)
![](https://img.shields.io/badge/license-MIT%2FApache%202-brightgreen)
![](https://img.shields.io/github/v/tag/phil-opp/blog_os?include_prereleases)

This repository contains the source code for the _Writing an OS in Rust_ series at [os.phil-opp.com](https://os.phil-opp.com).

If you have questions, open an issue or chat with us [on Gitter](https://gitter.im/phil-opp/blog_os).

## Where is the code?

The code for each post lives in a separate git branch. This makes it possible to see the intermediate state after each post.

**The code for the latest post is available [here][latest-post].**

[latest-post]: https://github.com/phil-opp/blog_os/tree/post-10

You can find the branch for each post by following the `(source code)` link in the [post list](#posts) below. The branches are named `post-XX` where `XX` is the post number, for example `post-03` for the _VGA Text Mode_ post or `post-07` for the _Hardware Interrupts_ post. For build instructions, see the Readme of the respective branch.

You can check out a branch in a subdirectory using [git worktree]:

[git worktree]: https://git-scm.com/docs/git-worktree

```
git worktree add code post-10
```

The above command creates a subdirectory named `code` that contains the code for the 10th post ("Heap Allocation").

## Posts

The goal of this project is to provide step-by-step tutorials in individual blog posts. We currently have the following set of posts:

**Bare Bones:**

- [A Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-01))
- [A Minimal Rust Kernel](https://os.phil-opp.com/minimal-rust-kernel/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-02))
- [VGA Text Mode](https://os.phil-opp.com/vga-text-mode/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-03))
- [Testing](https://os.phil-opp.com/testing/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-04))

**Interrupts:**

- [CPU Exceptions](https://os.phil-opp.com/cpu-exceptions/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-05))
- [Double Faults](https://os.phil-opp.com/double-fault-exceptions/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-06))
- [Hardware Interrupts](https://os.phil-opp.com/hardware-interrupts/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-07))

**Memory Management:**

- [Introduction to Paging](https://os.phil-opp.com/paging-introduction/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-08))
- [Paging Implementation](https://os.phil-opp.com/paging-implementation/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-09))
- [Heap Allocation](https://os.phil-opp.com/heap-allocation/)
    ([source code](https://github.com/phil-opp/blog_os/tree/post-10))

## First Edition Posts

The current version of the blog is already the second edition. The first edition is outdated and no longer maintained, but might still be useful. The posts of the first edition are:

**Bare Bones:**

- [A Minimal x86 Kernel](https://os.phil-opp.com/multiboot-kernel.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_1))
- [Entering Long Mode](https://os.phil-opp.com/entering-longmode.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_2))
- [Set Up Rust](https://os.phil-opp.com/set-up-rust.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_3))
- [Printing to Screen](https://os.phil-opp.com/printing-to-screen.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_4))

**Memory Management:**

- [Allocating Frames](https://os.phil-opp.com/allocating-frames.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_5))
- [Page Tables](https://os.phil-opp.com/modifying-page-tables.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_6))
- [Remap the Kernel](https://os.phil-opp.com/remap-the-kernel.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_7))
- [Kernel Heap](https://os.phil-opp.com/kernel-heap.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_8))

**Exceptions:**

- [Handling Exceptions](https://os.phil-opp.com/handling-exceptions.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_9))
- [Double Faults](https://os.phil-opp.com/double-faults.html)
      ([source code](https://github.com/phil-opp/blog_os/tree/first_edition_post_10))

**Additional Resources:**

- [Cross Compile Binutils](https://os.phil-opp.com/cross-compile-binutils.html)
- [Cross Compile libcore](https://os.phil-opp.com/cross-compile-libcore.html)
- [Set Up GDB](https://os.phil-opp.com/set-up-gdb)
- [Handling Exceptions using Naked Functions](https://os.phil-opp.com/handling-exceptions-with-naked-fns.html)
    - [Catching Exceptions](https://os.phil-opp.com/catching-exceptions.html)
          ([source code](https://github.com/phil-opp/blog_os/tree/catching_exceptions))
    - [Better Exception Messages](https://os.phil-opp.com/better-exception-messages.html)
          ([source code](https://github.com/phil-opp/blog_os/tree/better_exception_messages))
    - [Returning from Exceptions](https://os.phil-opp.com/returning-from-exceptions.html)
          ([source code](https://github.com/phil-opp/blog_os/tree/returning_from_exceptions))

## License
The source code is dual-licensed under MIT or the Apache License (Version 2.0). This excludes the `blog` directory.
