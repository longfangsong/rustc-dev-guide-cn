# 推荐的工作流程

完整的自举过程需要比较长的时间。 这里有五个建议能使您的生活更轻松。

## Configuring `rust-analyzer` for `rustc`

`rust-analyzer` can help you check and format your code whenever you save
a file. By default, `rust-analyzer` runs the `cargo check` and `rustfmt`
commands, but you can override these commands to use more adapted versions
of these tools when hacking on `rustc`. For example, for Visual Studio Code,
you can write:

```JSON
{
    "rust-analyzer.checkOnSave.overrideCommand": [
        "./x.py",
        "check",
        "--json-output"
    ],
    "rust-analyzer.rustfmt.overrideCommand": [
      "./build/TARGET_TRIPLE/stage0/bin/rustfmt"
    ],
    "editor.formatOnSave": true
}
```

in your `.vscode/settings.json` file. This will ask `rust-analyzer` to use
`x.py check` to check the sources, and the stage 0 rustfmt to format them.

If running `x.py check` on save is inconvenient, in VS Code you can use a [Build
Task] instead:

```JSON
// .vscode/tasks.json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "./x.py check",
            "command": "./x.py check",
            "type": "shell",
            "problemMatcher": "$rustc",
            "presentation": { "clear": true },
            "group": { "kind": "build", "isDefault": true }
        }
    ]
}
```

[Build Task]: https://code.visualstudio.com/docs/editor/tasks


## Check, check，再 check

When doing simple refactorings, it can be useful to run `./x.py check`
continuously. If you set up `rust-analyzer` as described above, this will
be done for you every time you save a file. Here you are just checking that
the compiler can **build**, but often that is all you need (e.g., when renaming a
method). You can then run `./x.py build` when you actually need to
run tests.

实际上，有时即使您不完全确定该代码将起作用，也可以暂时把测试放在一边。 
然后，您可以继续构建重构，提交代码，并稍后再运行测试。 
然后，您可以使用`git bisect`来**精确**地跟踪导致问题的提交。
这种风格的一个很好的副作用是最后能得到一组相当细粒度的提交，所有这些提交都能构建并通过测试。 这通常有助于审核代码。

## 使用 `--keep-stage` 持续构建

有时仅检查编译器是否能够编译是不够的。
一个常见的例子是您需要添加一条`debug!`语句来检查某些状态的值或更好地理解问题。
在这种情况下，您确实需要完整的构建。
但是，通过利用增量，您通常可以使这些构建非常快速地完成（例如，大约30秒）。
唯一的问题是，这需要一些伪造，并且可能会导致编译器无法正常工作（但是很容易检测和修复）。

所需的命令序列如下：

- 初始构建： `./x.py build -i library/std`
  - 如 [上文所述](#command)，这将在运行所有stage0命令，包括括构建一个stage1编译器与与其相兼容的`libstd`，
  并运行"stage 1 actions"的前几步，到“stage1 (sysroot stage1) builds libstd”为止。
- 接下来的构建： `./x.py build -i library/std --keep-stage 1`
  - 注意我们在此添加了 `--keep-stage 1` flag

如前所述，`-keep-stage 1`的作用是我们*假设*可以复用旧的标准库。 
如果你修改的是编译器的话，几乎总是这样：毕竟，您还没有更改标准库。 
但是有时候，这是不行的：例如，如果您正在编辑编译器的“元数据”部分，
该部分控制着编译器如何将类型和其他状态编码到`rlib`文件中，
或者您正在编辑的部分会体现在元数据上（例如MIR的定义）。

**TL;DR，使用`--keep-stage 1` 时您得到的结果可能会有奇怪的行为**，
例如，奇怪的[ICE](../appendix/glossary.html#ice)或其他panic。 
在这种情况下，您只需从命令中删除`--keep-stage 1`，然后重新构建。 
这样应该就能解决问题了。

## 使用系统自带的 LLVM 构建

- Initial test run: `./x.py test -i src/test/ui`
- Subsequent test run: `./x.py test -i src/test/ui --keep-stage 1`
默认情况下，LLVM是从源代码构建的，这可能会花费大量时间。 一种替代方法是使用计算机上已经安装的LLVM。

## Fine-tuning optimizations

Setting `optimize = false` makes the compiler too slow for tests. However, to
improve the test cycle, you can disable optimizations selectively only for the
crates you'll have to rebuild
([source](https://rust-lang.zulipchat.com/#narrow/stream/131828-t-compiler/topic/incremental.20compilation.20question/near/202712165)).
For example, when working on `rustc_mir_build`, the `rustc_mir_build` and
`rustc_driver` crates take the most time to incrementally rebuild. You could
therefore set the following in the root `Cargo.toml`:
```toml
[profile.release.package.rustc_mir_build]
opt-level = 0
[profile.release.package.rustc_driver]
opt-level = 0
```

## Working on multiple branches at the same time

Working on multiple branches in parallel can be a little annoying, since
building the compiler on one branch will cause the old build and the
incremental compilation cache to be overwritten. One solution would be
to have multiple clones of the repository, but that would mean storing the
Git metadata multiple times, and having to update each clone individually.

Fortunately, Git has a better solution called [worktrees]. This lets you
create multiple "working trees", which all share the same Git database.
Moreover, because all of the worktrees share the same object database,
if you update a branch (e.g. master) in any of them, you can use the new
commits from any of the worktrees. One caveat, though, is that submodules
do not get shared. They will still be cloned multiple times.

[worktrees]: https://git-scm.com/docs/git-worktree

Given you are inside the root directory for your rust repository, you can
create a "linked working tree" in a new "rust2" directory by running
the following command:

```bash
git worktree add ../rust2
```

Creating a new worktree for a new branch based on `master` looks like:

```bash
git worktree add -b my-feature ../rust2 master
```

You can then use that rust2 folder as a separate workspace for modifying
and building `rustc`!

## Building with system LLVM

默认情况下，LLVM是从源代码构建的，这可能会花费大量时间。 一种替代方法是使用计算机上已经安装的LLVM。

在 `config.toml`中的 `target`一节进行配置:

```toml
[target.x86_64-unknown-linux-gnu]
llvm-config = "/path/to/llvm/llvm-7.0.1/bin/llvm-config"
```

我们之前已经观察到以下路径，这些路径可能与您的系统上的不同：

- `/usr/bin/llvm-config-8`
- `/usr/lib/llvm-8/bin/llvm-config`

请注意，您需要安装LLVM`FileCheck`工具，该工具用于代码生成测试。 
这个工具通常是LLVM内建的，但是如果您使用自己的预装LLVM，则需要以其他方式提供`FileCheck`。
在基于Debian的系统上，您可以安装`llvm-N-tools`软件包（其中`N`是LLVM版本号，例如`llvm-8-tools`）。
或者，您可以使用`config.toml`中的`llvm-filecheck`配置项指定`FileCheck`的路径，
也可以使用`config.toml`中的`codegen-tests`项禁用代码生成测试。