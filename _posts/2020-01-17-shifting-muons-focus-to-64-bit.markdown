---
layout: post
title:  Shifting Muon's focus to 64-bit
date:   2020-01-17 17:00:00 +0100
---

The Muon compiler now targets 64-bit architectures by default. It was an easy change to make (essentially, I just made the command line flag `-m64` the default, instead of `-m32`), but it's an important one for the long term future of the language. The change brings Muon in line with other compilers like GCC and clang, which also target 64-bit architectures by default.

For the time being, we still support 32-bit via the C backend, though the LLVM backend (coming later this year) will only support 64-bit, at least initially.

The main benefit is that we can improve the language faster this way. Fortunately, desktops have been 64-bit architectures for a long time, and 32-bit phones [seem to be getting pretty rare these days](https://www.arm.com/blogs/blueprint/android-64bit-future-mobile). There are some cases where 32-bit code is faster (e.g. due to smaller pointers, resulting in better CPU cache usage), but ultimately the right solution there probably consists of some hybrid mode where we target a 64-bit architecture, but restrict ourselves to a 32-bit (or some kind of relative) address space. 32-bit will remain important for various embedded applications, so it is still a goal, just something we will tackle further down the road.

In other news, I've been working on a tool to generate foreign function declarations, called ffigen. A blog post on that is coming soon!

If you liked this post, [consider following me on twitter](https://twitter.com/nickmqb).
