# 编译器自举

本小节与自举过程有关。

## 什么是自举？他是怎么工作的？ bootstrapping?

[Bootstrapping] is the process of using a compiler to compile itself.
More accurately, it means using an older compiler to compile a newer version
of the same compiler.

This raises a chicken-and-egg paradox: where did the first compiler come from?
It must have been written in a different language. In Rust's case it was
[written in OCaml][ocaml-compiler]. However it was abandoned long ago and the
only way to build a modern version of rustc is a slightly less modern
version.

This is exactly how `x.py` works: it downloads the current `beta` release of
rustc, then uses it to compile the new compiler. The beta release is
called `stage0` and the newly built compiler is `stage1` (or `stage0
artifacts`). To get the full benefits of the new compiler (e.g. optimizations
and new features), the `stage1` compiler then compiles _itself_ again. This
last compiler is called `stage2` (or `stage1 artifacts`).

The `stage2` compiler is the one distributed with `rustup` and all other
install methods. However, it takes a very long time to build because one must
first build the new compiler with an older compiler and then use that to
build the new compiler with itself. For development, you usually only want
the `stage1` compiler: `x.py build --stage 1 library/std`.

## Complications of bootstrapping

Since the build system uses the current beta compiler to build the stage-1
bootstrapping compiler, the compiler source code can't use some features
until they reach beta (because otherwise the beta compiler doesn't support
them). On the other hand, for [compiler intrinsics][intrinsics] and internal
features, the features _have_ to be used. Additionally, the compiler makes
heavy use of nightly features (`#![feature(...)]`). How can we resolve this
problem?

There are two methods used:
1. The build system sets `--cfg bootstrap` when building with `stage0`, so we
can use `cfg(not(bootstrap))` to only use features when built with `stage1`.
This is useful for e.g. features that were just stabilized, which require
`#![feature(...)]` when built with `stage0`, but not for `stage1`.
2. The build system sets `RUSTC_BOOTSTRAP=1`. This special variable means to
_break the stability guarantees_ of rust: Allow using `#![feature(...)]` with
a compiler that's not nightly. This should never be used except when
bootstrapping the compiler.

[Bootstrapping]: https://en.wikipedia.org/wiki/Bootstrapping_(compilers)
[intrinsics]: ../appendix/glossary.md#intrinsic
[ocaml-compiler]: https://github.com/rust-lang/rust/tree/ef75860a0a72f79f97216f8aaa5b388d98da6480/src/boot

## Contributing to bootstrap

When you use the bootstrap system, you'll call it through `x.py`.
However, most of the code lives in `src/bootstrap`.
`bootstrap` has a difficult problem: it is written in Rust, but yet it is run
before the rust compiler is built! To work around this, there are two
components of bootstrap: the main one written in rust, and `bootstrap.py`.
`bootstrap.py` is what gets run by x.py. It takes care of downloading the
`stage0` compiler, which will then build the bootstrap binary written in
Rust.

Because there are two separate codebases behind `x.py`, they need to
be kept in sync. In particular, both `bootstrap.py` and the bootstrap binary
parse `config.toml` and read the same command line arguments. `bootstrap.py`
keeps these in sync by setting various environment variables, and the
programs sometimes to have add arguments that are explicitly ignored, to be
read by the other.

### Adding a setting to config.toml

This section is a work in progress. In the meantime, you can see an example
contribution [here][bootstrap-build].

[bootstrap-build]: https://github.com/rust-lang/rust/pull/71994

## Stages of bootstrap

This is a detailed look into the separate bootstrap stages. When running
`x.py` you will see output such as:

```txt
Building stage0 std artifacts
Copying stage0 std from stage0
Building stage0 compiler artifacts
Copying stage0 rustc from stage0
Building LLVM for x86_64-apple-darwin
Building stage0 codegen artifacts
Assembling stage1 compiler
Building stage1 std artifacts
Copying stage1 std from stage1
Building stage1 compiler artifacts
Copying stage1 rustc from stage1
Building stage1 codegen artifacts
Assembling stage2 compiler
Uplifting stage1 std
Copying stage2 std from stage1
Generating unstable book md files
Building stage0 tool unstable-book-gen
Building stage0 tool rustbook
Documenting standalone
Building rustdoc for stage2
Documenting book redirect pages
Documenting stage2 std
Building rustdoc for stage1
Documenting stage2 whitelisted compiler
Documenting stage2 compiler
Documenting stage2 rustdoc
Documenting error index
Uplifting stage1 rustc
Copying stage2 rustc from stage1
Building stage2 tool error_index_generator
```

在这里可以更深入地了解`x.py`的各个阶段：

<img alt="A diagram of the rustc compilation phases" src="../img/rustc_stages.svg" class="center" />

请记住，此图只是一个简化，即`rustdoc`可以在不同阶段构建，当传递诸如`--keep-stage`之类的标志或存在和宿主机类型不同的目标时，该过程会有所不同。

下表列出了各种阶段操作的输出：

| Stage 0 动作                                | Output                                       |
| ------------------------------------------- | -------------------------------------------- |
| 提取`beta`                                  | `build/HOST/stage0`                          |
| `stage0` 构建 `bootstrap`                   | `build/bootstrap`                            |
| `stage0` 构建 `test`/`libstd`                      | `build/HOST/stage0-std/TARGET`               |
| 复制 `stage0-std` (HOST only)               | `build/HOST/stage0-sysroot/lib/rustlib/HOST` |
| `stage0` 使用`stage0-sysroot`构建 `rustc`   | `build/HOST/stage0-rustc/HOST`               |
| 复制 `stage0-rustc `（可执行文件除外）      | `build/HOST/stage0-sysroot/lib/rustlib/HOST` |
| 构建 `llvm`                                 | `build/HOST/llvm`                            |
| `stage0` 使用`stage0-sysroot`构建 `codegen` | `build/HOST/stage0-codegen/HOST`             |
| `stage0` 使用`stage0-sysroot`构建 `rustdoc` | `build/HOST/stage0-tools/HOST`               |

`--stage=0` 到此为止。

| Stage 1 Action                                      | Output                                |
|-----------------------------------------------------|---------------------------------------|
| copy (uplift) `stage0-rustc` executable to `stage1` | `build/HOST/stage1/bin`               |
| copy (uplift) `stage0-codegen` to `stage1`          | `build/HOST/stage1/lib`               |
| copy (uplift) `stage0-sysroot` to `stage1`          | `build/HOST/stage1/lib`               |
| `stage1` builds `libstd`                            | `build/HOST/stage1-std/TARGET`        |
| copy `stage1-std` (HOST only)                       | `build/HOST/stage1/lib/rustlib/HOST`  |
| `stage1` builds `rustc`                             | `build/HOST/stage1-rustc/HOST`        |
| copy `stage1-rustc` (except executable)             | `build/HOST/stage1/lib/rustlib/HOST`  |
| `stage1` builds `codegen`                           | `build/HOST/stage1-codegen/HOST`      |

`--stage=1` 到此为止。

| Stage 2 动作                          | Output                                                       |
| ------------------------------------- | ------------------------------------------------------------ |
| 复制 (提升) `stage1-rustc` 可执行文件 | `build/HOST/stage2/bin`                                      |
| 复制 (提升) `stage1-sysroot`          | `build/HOST/stage2/lib and build/HOST/stage2/lib/rustlib/HOST` |
| `stage2` 构建 `libstd` (除 HOST?)     | `build/HOST/stage2-std/TARGET`                               |
| 复制 `stage2-std` (非 HOST 目标)      | `build/HOST/stage2/lib/rustlib/TARGET`                       |
| `stage2` 构建 `rustdoc`               | `build/HOST/stage2-tools/HOST`                               |
| 复制 `rustdoc`                        | `build/HOST/stage2/bin`                                      |

`--stage=2` 到此为止。

注意，`x.py`使用的约定是：

- “stage N 产品”是由stage N编译器产生的制品。
- “stage (N+1)编译器”由“stage N 产品”组成。
- “--stage N”标志表示使用stage N构建。

简而言之，_stage 0使用stage0编译器创建stage0产品，随后将其提升为stage1_。

每次编译任何主要产品（`std`和`rustc`）时，都会执行两个步骤。
当`std`由N级编译器编译时，该`std`将链接到由N级编译器构建的程序（包括稍后构建的`rustc`）。 stage (N+1)编译器还将使用它与自身链接。
如果有人认为stage (N+1)编译器“只是”我们正在使用阶段N编译器构建的另一个程序，那么这有点直观。 
在某些方面，可以将`rustc`（二进制文件，而不是`rustbuild`步骤）视为少数`no_core`二进制文件之一。

因此，“stage0 std制品”实际上是下载的stage0编译器的输出，并且将用于stage0编译器构建的任何内容：
例如 `rustc`制品。 当它宣布正在“构建stage1 std制品”时，它已进入下一个自举阶段。 在以后的阶段中，这种模式仍在继续。

还要注意，根据stage的不同，构建主机`std`和目标`std`的情况有所不同（例如，在表格中看到stage2仅构建非主机`std`目标。
这是因为在stage2期间，主机`std`是从stage 1`std`提升过来的——
特别地，当宣布“ Building stage 1 artifacts”时，它也随后被复制到stage2中（编译器的`libdir `和`sysroot`））。

The `rustc` generated by the stage0 compiler is linked to the freshly-built
`std`, which means that for the most part only `std` needs to be cfg-gated,
so that `rustc` can use featured added to std immediately after their addition,
without need for them to get into the downloaded beta. The `std` built by the
`stage1/bin/rustc` compiler, also known as "stage1 std artifacts", is not
necessarily ABI-compatible with that compiler.
That is, the `rustc` binary most likely could not use this `std` itself.
It is however ABI-compatible with any programs that the `stage1/bin/rustc`
binary builds (including itself), so in that sense they're paired.
对于编译器的任何有用的工作，这个`std`都是非常必要的。
具体来说，它用作新编译的编译器所编译程序的`std`
（因此，当您编译`fn main() {}`时，它将链接到使用`x.py build --stage 1 src/libstd`编译的最后一个`std`）。

由stage0编译器生成的`rustc`链接到新构建的`libstd`，这意味着在大多数情况下仅需要对`std`进行cfg门控，以便`rustc`可以立即使用添加到std的功能。
添加后，无需进入下载的Beta。由`stage1/bin/rustc`编译器构建的`libstd`，也称“stage1 std”构件，不一定与该编译器具有ABI兼容性。
也就是说，`rustc`二进制文件很可能无法使用此`std`本身。
然而，它与`stage1/bin/rustc`二进制文件所构建的任何程序（包括其自身）都具有ABI兼容性，因此从某种意义上讲，它们是配对的。


这也是`--keep-stage 1 src/libstd`起作用的地方。 
由于对编译器的大多数更改实际上并未更改ABI，因此，一旦在阶段1中生成了`libstd`，您就可以将其与其他编译器一起使用。
如果ABI没变，那就很好了，不需要花费时间重新编译`std`。
`--keep-stage`假设先前的编译没有问题，然后将这些制品复制到适当的位置，从而跳过cargo调用。

我们首先构建`std`，然后构建`rustc`的原因基本上是因为我们要最小化`rustc`代码中的`cfg(stage0)`。
当前`rustc`总是与“新的”`std`链接，因此它不必关心std的差异。它可以假定std尽可能新。

我们需要两次构建它的原因是因为ABI兼容性。 Beta编译器具有自己的ABI，而`stage1/bin/rustc`编译器将使用新的ABI生成程序/库。
我们曾经要编译3次，但是由于我们假设ABI在代码库中是恒定的，
我们假定“stage2”编译器生成的库（由`stage1/bin/rustc`编译器产生）与`stage1/bin/rustc`编译器产生的库ABI兼容。
这意味着我们可以跳过最后一次编译 —— 只需使用`stage2/bin/rustc`编译器自身所使用的库。

这个`stage2/bin/rustc`编译器和`stage 1 {std, rustc}`一起被交付给最终用户。

## Passing stage-specific flags to `rustc`

`x.py` allows you to pass stage-specific flags to `rustc` when bootstrapping.
The `RUSTFLAGS_STAGE_0`, `RUSTFLAGS_STAGE_1` and `RUSTFLAGS_STAGE_2`
environment variables pass the given flags when building stage 0, 1, and 2
artifacts respectively.

Additionally, the `RUSTFLAGS_STAGE_NOT_0` variable, as its name suggests, pass
the given arguments if the stage is not 0.

## Environment Variables

## 环境变量

在自举过程中，使用了很多编译器内部的环境变量。 如果您尝试运行`rustc`的中间版本，有时可能需要手动设置其中一些环境变量。 否则，您将得到如下错误：

```text
thread 'main' panicked at 'RUSTC_STAGE was not set: NotPresent', library/core/src/result.rs:1165:5
```

如果`./stageN/bin/rustc`给出了有关环境变量的错误，那通常意味着有些不对劲
——或者您正在尝试编译例如`librustc`或`libstd`或依赖于环境变量的东西。 在极少数情况下，您才会需要在这种情况下调用`rustc`，
您可以通过在`x.py`命令中添加以下标志来找到环境变量值：`--on-fail=print-env`。