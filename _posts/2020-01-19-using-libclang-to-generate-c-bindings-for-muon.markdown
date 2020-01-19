---
layout: post
title:  Using libclang to generate C bindings for Muon
date:   2020-01-19 19:00:00 +0100
---

Muon aims to make it easy to use code that's written in other languages. Muon has had support for working with foreign C interfaces from the beginning, but using the feature can be a bit cumbersome. Foreign interface declarations (e.g. functions, structs, etc.) have to be specified manually, which is tedious. So let's automate this, by building "ffigen": a standalone tool that takes a .c or .h file and generates a Muon file with corresponding foreign interface declarations (a.k.a. "bindings").

## Parsing C

First of all, we need to figure out some way of parsing C code. C might seem like a "small" language (especially when compared to something like C++), but writing a fully functional C parser by ourselves would still be a huge task. We need a mature parser that can handle any C file (.c or .h) we throw at it. Fortunately, such a parser is available in the form of [libclang](https://clang.llvm.org/docs/Tooling.html), which is part of the LLVM project. It even has a C interface, which means that we can use it from Muon. For the time being, we'll need to manually declare any foreign functions that we use from it. By the end of this project, we hopefully don't need to do that anymore!

## Mapping C to Muon

C and Muon are both low level languages, so at a glance it seems pretty straightforward to map between the two. That is certainly the case for basic language constructs like functions and structs. However, when we dive into the details we find some problems. Some are easily solved, while others are more fundamental, and applicable to any language that wants to interoperate with C.

* **Global namespace vs struct/union/enum tag names.** In C, structs, enums and unions have their own namespace. For example, a struct can be declared as `struct abc { ... }` and then later be referred to as `struct abc`. In practice, people usually use a `typedef` to bring such a type into the global namespace, but certainly not always, so we need to support both cases. Also, things can break if we naively map both `struct abc` and `abc` (which might refer to something else) to `abc` in Muon.

* **Loosely defined primitive type sizes.** The C spec does not specify the exact size of various primitive types (e.g. `char`, `short`, `int`, `long`). Fortunately, most platforms agree on the size of most of these, so we can make a practical choice to map each primitive type to a Muon type that matches its usual size. There are some exceptions, such as `long`, which is different between Linux/macOS and Windows on 64-bit platforms. Solution: for now, we generate definitions that are only suited for a specific platform. Later, we may want to introduce a `c_long` type in Muon specifically for C interop; that way, a single definition can be used in those cases.

* **Preprocessor flags.** For some libraries, we may inevitably have to generate multiple files for each platform and architecture combination, as the C preprocessor allows for declarations to vary wildly between architectures/platforms through the use of `#ifdef`s. Solution: for now, we accept that each generated foreign definition file is only suited for a specific architecture/platform. Later, we may want to parse the C file multiple times with different architecture settings, e.g. to discover which types could be mapped to Muon's `ssize` or `usize` types (machine word sized integers), so we can perhaps avoid the need for architecture specific definitions in some cases.

* **Preprocessor constant definitions.** In C, it is common for people to use `#define` for "constant" declarations. Unfortunately, libclang is not able to directly tell us the value of such a constant. When libclang parses a C file, symbols are replaced by the preprocessor, and we only see the final result. libclang does provide an extra flag to tell us about `#define`s as it encounters them, but we just get a token stream of the definition. For constants, the token stream will usually be a single literal token, although more complex expressions are not uncommon (e.g. binary OR-ing a bunch of flags). We'd like to avoid writing a C expression parser and evaluator, so our solution is a bit of a hack: we ask libclang to parse the C file to discover any `#define`d constants, then we generate additional C code in the form of `const some_type some_temp_name = PREPROCESSOR_CONSTANT;`, which we then feed through libclang again to get the final value, which is now possible because it is assigned to a "real" constant.

* **Preprocessor macros.** Macros suffer from the same problems as preprocessor constant definitions. For now, we don't support them. Later, we could parse each macro definition's token stream and reconstruct an equivalent Muon function. C macros can get pretty crazy though, so this won't be possible for everything.

* **Enum backing type difference.** In C, enums are backed by a signed 32-bit integer. In Muon, they are backed by an unsigned integer. Solution: convert negative enum values to positive, but allow users to override this.

* **`char *` vs strings.** In C, a `char *` can be used to represent a sequence of bytes, as well as a string. Muon has `*byte` (or `*sbyte`) and `cstring`, so the best way to map a `char *` is not always clear. Solution: map to byte pointer by default, but allow users to override this.

* **Bitfields.** Muon does not support bit fields. Solution: generate padding fields that span the same number of bytes, so that the parent struct and other non-bitfield fields can still be used.

* **Variadic functions.** Solution: supported in Muon for foreign functions (via the `#VarArgs` attribute).

* **Rarely used types.** For example: `long double`. Solution: just ignore for now.

Finally, some problems are due to Muon not yet being done. For example, Muon does not yet support inline fixed size arrays, unions, and function pointers with custom calling conventions. For now, we can work around those issues via some hacks, like unrolling array struct members, mapping only specific variants of unions, and mapping function pointers to a general `pointer` type. Once Muon has the required features, we can update ffigen to fix these issues.

## Putting it all together

That was quite a bit of stuff! Building ffigen was certainly an iterative process: trying to convert various C APIs, seeing what problems came up, and then addressing them.

As mentioned above, there are some scenarios where we want the user to be able to customize the foreign interface generation process. We handle this via a so-called [rules file](https://github.com/nickmqb/muon/blob/master/ffigen/README.md#rules-file), that allows users to specify which symbols to include/exclude, as well as how each symbol should be mapped.

The implementation of ffigen is pretty straightforward. In a nutshell, we use libclang's `clang_parseTranslationUnit` function to get a `CXTranslationUnit`. We obtain a `CXCursor` via `clang_getTranslationUnitCursor`, then iterate over the translation unit using `clang_visitChildren`. For each AST node, we call various other `clang_*` functions (e.g. `clang_getCursorKind`, `clang_getCursorSpelling`, etc.) to learn about the node, then map the node, and finally write the definition to an output file.

## Conclusion

With that, it's time to put ffigen to the test. Let's generate a foreign interface for libclang, swap out our manually created libclang declarations with auto generated ones, and... it works! The [rules file is pretty short](https://github.com/nickmqb/muon/blob/master/ffigen/libclang.rules), and the [generated Muon file can be seen here](https://github.com/nickmqb/muon/blob/master/ffigen/libclang_linux_macos.mu). Other libraries that I have (lightly) tested so far include: the C stdlib, Win32, POSIX and SDL.

If you'd like to give ffigen a try, [documentation can be found here](https://github.com/nickmqb/muon/blob/master/ffigen/README.md).

If you liked this post, [consider following me on Twitter](https://twitter.com/nickmqb).
