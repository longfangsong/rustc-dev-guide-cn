# MIR （中层IR）

MIR 是 Rust's _中层中间表示_. 
MIR是在[RFC 1211]中引入的。 它是Rust的一种非常简化的形式，用于某些对控制流敏感的安全检查——尤其是是借用检查器！ ——以及优化和代码生成。
如果您想阅读对MIR非常层次的介绍，以及它所依赖的一些编译器概念（例如控制流图和简化），则可以欣赏[介绍MIR的rust-lang博客文章 ][blog]。

[blog]: https://blog.rust-lang.org/2016/04/19/MIR.html

## 介绍 MIR

MIR 在 [`src/librustc_middle/mir/`][mir] 模块中定义，但许多操纵它的代码都在 [`src/librustc_mir`][mirmanip].

[RFC 1211]: https://rust-lang.github.io/rfcs/1211-mir.html

MIR的一些核心特征有：

- 它基于 [控制流图][cfg]。
- 他没有嵌套的表达式。
- MIR中的所有类型都是完全显式的。

[cfg]: ../appendix/background.html#cfg

## MIR核心词汇

本节介绍了MIR的关键概念，总结如下：

- **基本块**： 控制流图的单元，包含了：
  - **语句：** 有一个后继的动作
  - **终结句：** 可能有多个后继的动作，永远在块的末尾
  - （如果你对术语*基本块*不熟悉，见 [背景知识][cfg])
- **本地变量：** 在堆栈上分配的内存位置（至少在概念上是这样），例如函数参数，局部变量和临时变量。
这些由索引标识，并带有前导下划线，例如`_1`。 还有一个特殊的“本地变量”（`_0`）分配来存储返回值。
- **位置：** 用来表达内存中一个位置的表达式，像`_1` 或者`_1.f`.
- **右值：** 生成一个值的表达式，“右”意味着这些表达式一般只会出现在赋值语句的右侧。
  - **操作数：** 右值表达式的参数，可以是一个常数（如`22`）或者一个位置（如`_1`）。

通过将简单的程序转换为MIR并读取pretty print的输出，您可以了解MIR的结构。
实际上，playgroud使得此操作变得容易，因为它提供了一个MIR按钮，该按钮将向您显示程序的MIR。
尝试运行此程序（或[单击此链接][sample-play]），然后单击顶部的“ MIR”按钮：

[sample-play]: https://play.rust-lang.org/?gist=30074856e62e74e91f06abd19bd72ece&version=stable

```rust
fn main() {
    let mut vec = Vec::new();
    vec.push(1);
    vec.push(2);
}
```

你会看见：

```mir
// WARNING: This output format is intended for human consumers only
// and is subject to change without notice. Knock yourself out.
fn main() -> () {
    ...
}
```

这是 `main` 函数的MIR格式。

**变量定义** 如果我们深入一些，我们可以看到函数以一些变量定义开始，他们看起来像这样：

```mir
let mut _0: ();                      // return place
let mut _1: std::vec::Vec<i32>;      // in scope 0 at src/main.rs:2:9: 2:16
let mut _2: ();
let mut _3: &mut std::vec::Vec<i32>;
let mut _4: ();
let mut _5: &mut std::vec::Vec<i32>;
```

您会看到MIR中的变量没有名称，而是具有索引，例如`_0`或`_1`。
我们还将用户变量（例如`_1`）与临时值（例如`_2`或`_3`）混为一谈。
但您还是可以区分出哪些是用户定义的变量，因为它们具有与之相关联的调试信息（请参见下文）。

**用户变量的调试信息** 在变量定义下面，我们能发现唯一能提醒我们 `_1` 代表的是一个用户变量的提示：
```mir
scope 1 {
    debug vec => _1;                 // in scope 1 at src/main.rs:2:9: 2:16
}
```
每个 `debug <Name> => <Place>;` 注解都描述了一个用户定义变量与调试器在哪里（即位置）能找到这个变量对应的数据。
这里这个映射非常简单，但优化可能会使得这个位置的使用情况复杂化，也可能会让多个用户变量共享同一个位置。
另外，闭包的捕获也是用同一套系统描述的，这种情况下，即使不进行优化，也已经很复杂了。如：`debug x => (*((*_1).0: &T));`。

“scope”块（例如，`scope 1 {..}`）描述了源程序的词法结构（某个名称在哪个作用域中），
因此，用`// in scope 0`中注释的程序的任何部分都看不到`vec`，在调试器中单步执行代码时就能发现这一点。

**基本块**：进一步阅读代码，我们能看到我们的第一个“基本块”（自然，当您查看它时，它看起来可能略有不同，我也省略了一些注释）：

```mir
bb0: {
    StorageLive(_1);
    _1 = const <std::vec::Vec<T>>::new() -> bb2;
}
```

基本块由一系列**语句**和最终**终结句**定义。 在这个例子，有一个语句：

```mir
StorageLive(_1);
```

该语句表明变量` _1`是“活动的”，这意味着它可以在以后使用 —— 它将持续存在，直到遇到` StorageDead(_1)`语句为止，该语句表明变量`_1`已完成使用。
LLVM使用这些“存储语句”来分配栈空间。

`bb0`块的 **终结句** 是对 `Vec::new`的调用：

```mir
_1 = const <std::vec::Vec<T>>::new() -> bb2;
```

终结句和一般语句不同，它们能有多个后继 —— 控制流可能会流向不同的地方。
像 `Vec::new` 这样的函数调用永远是终结句，因为这可能可以导致堆栈解退，尽管在`Vec::new`的情况下显然堆栈解退是不可能的，因此我们只列出了唯一的后继块`bb2`。

如果我们继续向前看到 `bb2`，我们可以看见像这样的代码：

```mir
bb2: {
    StorageLive(_3);
    _3 = &mut _1;
    _2 = const <std::vec::Vec<T>>::push(move _3, const 1i32) -> [return: bb3, unwind: bb4];
}
```

这里有两个语句：另一个 `StorageLive`，引入了` _3`临时变量，然后是一个赋值：

```mir
_3 = &mut _1;
```

赋值一般有形式：

```text
<Place> = <Rvalue>
```

位置是类似于`_3`，`_ 3.f`或`* _3`的表达式——它表示内存中的位置。
**右值**是一个创建值的表达式：在这种情况下，rvalue是一个可变借用表达式，看起来像`&mut <Place>`。
因此，我们可以为右值定义语法，如下所示：

```text
<Rvalue>  = & (mut)? <Place>
          | <Operand> + <Operand>
          | <Operand> - <Operand>
          | ...

<Operand> = Constant
          | copy Place
          | move Place
```

从该语法可以看出，右值不能嵌套——它们只能引用位置和常量。
此外，当您使用某个位置时，我们会指明是要**复制**该位置（要求该位置的类型为 `T: Copy`）还是**移动**它（适用于 任何类型的位置）。
因此，例如，如果我们在Rust中写了表达式`x = a + b + c`，它将被编译为两个语句和一个临时变量：

```mir
TMP1 = a + b
x = TMP1 + c
```

（[试试看][play-abc]，你可能想要使用release模式来编译来跳过overflow检查）

[play-abc]: https://play.rust-lang.org/?gist=1751196d63b2a71f8208119e59d8a5b6&version=stable

## MIR 中的数据类型

MIR中的数据类型的定义在 [`src/librustc_middle/mir/`][mir]模块中。
前面章节提到的关键概念都有一个直接对应的Rust类型。

MIR的主要数据类型为`Mir`。 它包含单个函数的数据（以及Mir的“提升过的常量”的子实例，[您可以在下面阅读其中的内容](#promoted)）。

- **基本块**: 基本块被保存在 `basic_blocks`成员中；这是一个`BasicBlockData`向量。
我们不会直接引用一个基本块，代替地，我们会传递`BasicBlock`值，其实际上是[newtype过的]这个向量中的索引。
- **语句** 由 `Statement`类型表示。
- **终结句** 由  `Terminator`类型表示。
- **本地变量**  由类型 `Local` （[newtype过的]索引）表示。
本地变量的实际数据保存在`Mir`中的`local_decls`。
也有一个特殊的常量`RETURN_PLACE`来标记一个特殊的表示返回值的本地变量。
- **位置** 由枚举 `Place`表示。有如下变种：
  - 本地变量如 `_1`
  - 静态变量如 `FOO`
  - **投影**，这一般是结构的成员或者从某个基位置“投影”出来的位置。
例如`_1.f`就是从`)1`上投影出来的。
`*_1`也是一个投影，这类投影由 `ProjectionElem::Deref` 代表。
- **Rvalues** 由 `Rvalue`枚举表示。
- **Operands** 由 `Operand` 枚举表示。

## 表示常量

*to be written*

<a name="promoted"></a>

### 提升过的常量

*to be written*


[mir]: https://github.com/rust-lang/rust/tree/master/src/librustc_middle/mir
[mirmanip]: https://github.com/rust-lang/rust/tree/master/src/librustc_mir
[mir]: https://github.com/rust-lang/rust/tree/master/src/librustc_middle/mir
[newtype过的]: ../appendix/glossary.html#newtype
