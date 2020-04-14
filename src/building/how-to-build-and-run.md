# 如何构建并运行编译器

编译器使用 `x.py` 工具构建。您将需要安装Python才能运行它。 但是在此之前，如果您打算修改`rustc`的代码，则需要调整编译器的配置。因为默认配置面向以用户而不是开发人员来进行构建。

## 创建一个 config.toml

先将 [`config.toml.example`] 复制为 `config.toml`:

[`config.toml.example`]: https://github.com/rust-lang/rust/blob/master/config.toml.example

```bash
> cd $RUST_CHECKOUT
> cp config.toml.example config.toml
```

然后，您将需要打开这个文件并更改以下设置（根据需求不同可能也要修改其他的设置，例如`llvm.ccache`）：

```toml
[llvm]
# Enables LLVM assertions, which will check that the LLVM bitcode generated
# by the compiler is internally consistent. These are particularly helpful
# if you edit `codegen`.
assertions = true

[rust]
# This will make your build more parallel; it costs a bit of runtime
# performance perhaps (less inlining) but it's worth it.
codegen-units = 0

# This enables full debuginfo and debug assertions. The line debuginfo is also
# enabled by `debuginfo-level = 1`. Full debuginfo is also enabled by
# `debuginfo-level = 2`. Debug assertions can also be enabled with
# `debug-assertions = true`. Note that `debug = true` will make your build
# slower, so you may want to try individually enabling debuginfo and assertions
# or enable only line debuginfo which is basically free.
debug = true
```

如果您已经构建过了`rustc`，那么您可能必须执行`rm -rf build`才能使配置更改生效。 
请注意，`./x.py clean` 不会导致重新构建LLVM。
因此，如果您的配置更改影响LLVM，则在重新构建之前，您将需要手动`rm -rf build /`。

## `x.py`是什么？

`x.py`是用于编排`rustc`存储库的各种构建的脚本。
该脚本可以构建文档，运行测试并编译`rustc`。
现在它替代了以前的makefile，是构建`rustc`的首选方法。下面是利用`x.py`来有效处理各种任务的常见方式。

本章侧重于提高生产力的基础知识，但是，如果您想了解有关`x.py`的更多信息，请在[此处](https://github.com/rust-lang/rust/blob/master/src/bootstrap/README.md)阅读其README.md。

## 自举

要记住的一件事是`rustc`是一个自举式编译器。
也就是说，由于`rustc`是用Rust编写的，因此我们需要使用较旧版本的编译器来编译较新的版本。
特别是，新版本的编译器以及构建该编译器所需的一些组件，例如`libstd`和其他工具，可能在内部使用一些unstable的特性，因此需要能使用这些unstable特性的特定版本。

因此编译`rustc`需要分阶段完成：

- **Stage 0**：stage0中使用的编译器通常是当前最新的的beta 版本`rustc`编译器及其关联的动态库（您也可以将x.py配置为使用其他版本的编译器）。
  此stage 0编译器仅用于编译`rustbuild`，`std`和`rustc`。

  编译`rustc`时，此stage0编译器使用新编译的`std`。

  这里有两个概念：一个编译器（及其依赖）及其“目标”或“对象”库（`std`和`rustc`）。两者均会在此阶段出现，但以交错方式进行。

- **Stage 1**：然后使用stage0编译器编译你的代码库中的代码，以生成新的stage1编译器。
  
  但是，它是使用较旧的编译器（stage0）构建的，因此为了优化stage1编译器，我们需要进入下一阶段。
  
  - 从理论上讲，stage1编译器在功能上与stage2编译器相同，但实际上它们存在细微差别。
  
    特别是，stage1中使用的编译器本身是由stage0编译器构建的，而不是由您的工作目录中的源构建的。
  
    这意味着，在编译器源代码中使用的符号名称可能与stage1编译器生成的符号名称不匹配。
  
    这在使用动态链接时非常重要（例如，带有derive的代码）。有时这意味着某些测试在与stage1运行时不起作用。
  
- **Stage 2**：我们使用stage1中得到的编译器重新构建其自身，以产生具有所有最新优化的stage2编译器。 
（默认情况下，我们复制stage1中的库供stage2编译器使用，因为它们应该是相同的。）

- （可选）**Stage 3**：要完全检查我们的新编译器，我们可以使用stage2编译器来构建库。除非出现故障，否则结果应与之前相同。
  要了解有关自举过程的更多信息，请[阅读本章][bootstrap]。

[bootstrap]: ./bootstrapping.md

## 构建编译器

要完整构建编译器，请运行`./x.py build`。 这将完成上述整个引导过程，并从您的源代码中生成可用的编译器工具链。 这需要很长时间，因此通常不需要真的运行这条命令（稍后会详细介绍）。

您可以将许多标志传递给`x.py`的build命令，这些标志可以减少编译时间或适应您可能需要更改的其他内容。 他们是：

```txt
Options:
    -v, --verbose       use verbose output (-vv for very verbose)
    -i, --incremental   use incremental compilation
        --config FILE   TOML configuration file for build
        --build BUILD   build target of the stage0 compiler
        --host HOST     host targets to build
        --target TARGET target targets to build
        --on-fail CMD   command to run on failure
        --stage N       stage to build
        --keep-stage N  stage to keep without recompiling
        --src DIR       path to the root of the rust checkout
    -j, --jobs JOBS     number of jobs to run in parallel
    -h, --help          print this help message
```

对于一些hacking，通常构建stage 1编译器就足够了，但是对于最终测试和发布，则使用stage 2编译器。

`./x.py check`可以快速构建rust编译器。 当您执行某种“基于类型的重构”（例如重命名方法或更改某些函数的签名）时，它特别有用。

<a name=command></a>

在创建了config.toml之后，就可以运行`x.py`。 这里有很多选项，但让我们从构建本地rust的最佳命令开始：

```bash
./x.py build -i --stage 1 src/libstd
```

*看起来*好像它仅构建`libstd`，但事实并非如此。该命令的作用如下：

- 使用stage0编译器构建`libstd`（使用增量）
- 使用stage0编译器构建`librustc`（使用增量）
  - 这产生了stage1编译器
- 使用stage1编译器构建`libstd`（不能使用增量式）

最终产品 (stage1编译器+使用该编译器构建的库)是构建其他rust程序所需要的（除非使用`#![no_std]`或`#![no_core]`）。

该命令包括`-i`开关，该开关启用增量编译。这将用于加快该过程的前两个步骤：特别是，如果您进行了较小的更改，我们应该能够使用您上一次编译的结果来更快地生成stage1编译器。

不幸的是，不能使用增量来加速stage1库的构建。
这是因为增量仅在连续运行*同一*编译器两次时才起作用。
在这种情况下，我们每次都会构建一个新的*stage1编译器*。
因此，旧的增量结果可能不适用。
**您可能会发现构建stage1 `libstd`对您来说是一个瓶颈** —— 但不要担心，这有一个（hacky的）解决方法。请参阅下面[“推荐的工作流程”](./suggested.md)部分。

请注意，这整个命令只是为您提供完整rustc构建的一部分。完整的rustc构建（即`./x.py build`命令）还有很多步骤：

- 使用stage1编译器构建librustc和rustc。
  - 此处生成的编译器为stage2编译器。
- 使用stage2编译器构建`libstd`。
- 使用stage2编译器构建`librustdoc`和其他内容。

## 构建特定组件

只构建 libcore 库

```bash
./x.py build src/libcore
```

只构建 libcore 和 libproc_macro 库

```bash
./x.py build src/libcore src/libproc_macro
```

只构建到 Stage 1 为止的 libcore

```bash
./x.py build src/libcore --stage 1
```

有时您可能只想测试您正在处理的部分是否可以编译。
使用这些命令，您可以在进行较大的构建之前进行测试，以确保它可以与编译器一起使用。
如前所示，您还可以在末尾传递标志，例如--stage。

## 创建一个rustup工具链

成功构建rustc之后，您在构建目录中已经创建了一堆文件。 
为了实际运行生成的`rustc`，我们建议创建两个rustup工具链。 
第一个将运行stage1编译器（上面构建的结果）。
第二个将执行stage2编译器（我们尚未构建这个编译器，但是您可能需要在某个时候构建它；例如，如果您想运行整个测试套件）。

```bash
rustup toolchain link stage1 build/<host-triple>/stage1
rustup toolchain link stage2 build/<host-triple>/stage2
```

 `<host-triple>` 一般来说是以下三者之一:

- Linux: `x86_64-unknown-linux-gnu`
- Mac: `x86_64-apple-darwin`
- Windows: `x86_64-pc-windows-msvc`

现在，您可以运行构建出的`rustc`。 如果使用`-vV`运行，则应该看到以`-dev`结尾的版本号，表示从本地环境构建的版本：

```bash
$ rustc +stage1 -vV
rustc 1.25.0-dev
binary: rustc
commit-hash: unknown
commit-date: unknown
host: x86_64-unknown-linux-gnu
release: 1.25.0-dev
LLVM version: 4.0
```
## 其他 `x.py` 命令

这是其他一些有用的`x.py`命令。我们将在其他章节中详细介绍其中一些：

- 构建:
  - `./x.py clean` – 清理构建目录 (`rm -rf build` 也能达到这个效果，但你必须重新构建LLVM)
  - `./x.py build --stage 1` – 使用stage 1 编译器构建所有东西，不止是 `libstd`
  - `./x.py build` – 构建 stage2 编译器
- 运行测试 （见 [运行测试](../tests/running.html) 章节）:
  - `./x.py test --stage 1 src/libstd` – 为`libstd`运行 `#[test]` 测试
  
  - `./x.py test --stage 1 src/test/ui` – 运行 `ui` 测试组
  
  - `./x.py test --stage 1 src/test/ui/const-generics` - 运行`ui` 测试组下的 `const-generics/` 子文件夹中的测试
  
  - `./x.py test --stage 1 src/test/ui/const-generics/const-types.rs` 
  
    - 运行`ui`测试组下的 `const-types.rs` 中的测试

### 清理构建文件夹

有时您需要重新开始，但是通常情况并非如此。
如果您感到需要这么做，那么其实有可能是`rustbuild`无法正确执行，此时你应该提出一个bug来告知我们什么出错了。
如果确实需要清理所有内容，则只需运行一个命令！

```bash
./x.py clean
```
