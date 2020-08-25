# THIR 和 MIR 的构建

从 [HIR] lower到 [MIR] 的过程会在下面（可能不完整）这些item上进行：

* 函数体和闭包体
* `static`和`const`的初始化
* 枚举判别的初始化
* 任何类型的胶水和补充代码
    * Tuple结构体的初始化函数
    * Drop 代码 （ `Drop::drop` 函数不会直接被调用）
    * 没有显式 `Drop` 实现的对象的`drop`

The lowering is triggered by calling the [`mir_built`] query.
There is an intermediate representation
between [HIR] and [MIR] called the [THIR] that is only used during the lowering.
[THIR] means "Typed HIR" and used to be called "HAIR (High-level Abstract IR)".
The [THIR]'s most important feature is that the various adjustments (which happen
without explicit syntax) like coercions, autoderef, autoref and overloaded method
calls have become explicit casts, deref operations, reference expressions or
concrete function calls.

The [THIR] has datatypes that mirror the [HIR] datatypes, but instead of e.g. `-x`
being a `thir::ExprKind::Neg(thir::Expr)` it is a `thir::ExprKind::Neg(hir::Expr)`.
This shallowness enables the `THIR` to represent all datatypes that [HIR] has, but
without having to create an in-memory copy of the entire [HIR].
[MIR] lowering will first convert the topmost expression from
[HIR] to [THIR] (in [`rustc_mir_build::thir::cx::expr`]) and then process
the [THIR] expressions recursively.

Lowering会为函数签名中指定的每个参数创建局部变量。
接下来，它为指定的每个绑定创建局部变量（例如， `(a, b): (i32, String)`）产生3个绑定，一个用于参数，两个用于绑定。
接下来，它生成字段访问，该访问从参数读取字段并将其值写入绑定变量。

在解决了初始化的情况下，lowering为函数体递归生成MIR（ `Block` 表达式）并将结果写入`RETURN_PLACE`。

## `unpack!` 所有东西

生成MIR的函数有两种模式。
第一种情况，如果该函数仅生成语句，则它将以基本块作为参数，这些语句应放入该基本块。
然后可以正常返回结果：

```rust,ignore
fn generate_some_mir(&mut self, block: BasicBlock) -> ResultType {
   ...
}
```

但是还有其他一些函数会生成新的基本块。
例如，lowering像`if foo { 22 } else { 44 }`这样的表达式需要生成一个小的“菱形图”。
在这种情况下，函数将在其代码开始处使用一个基本块，并在代码生成结束时返回一个（可能）新的基本块。
`BlockAnd`类型用于表示此类情况：

```rust,ignore
fn generate_more_mir(&mut self, block: BasicBlock) -> BlockAnd<ResultType> {
    ...
}
```

当您调用这些函数时，通常有一个局部变量`block`，它实际上是一个“光标”。 它代表了我们要添加新的MIR的位置。
当调用`generate_more_mir`时，您会想更新该光标。
您可以手动执行此操作，但这很繁琐：

```rust,ignore
let mut block;
let v = match self.generate_more_mir(..) {
    BlockAnd { block: new_block, value: v } => {
        block = new_block;
        v
    }
};
```

For this reason, we offer a macro that lets you write
`let v = unpack!(block = self.generate_more_mir(...))`.
It simply extracts the new block and overwrites the
variable `block` that you named in the `unpack!`.

因此，我们提供了一个宏，可让您编写
`let v = unpack!(block = self.generate_more_mir(...))`。
它简单地提取新的块并覆盖在`unpack!`中指明的变量`block`。

## 将表达式 Lowering 到 MIR

本质上一个表达式可以有四种表示形式：

* `Place` 指一个（或一部分）已经存在的内存地址（本地，静态，或者提升过的）
* `Rvalue` 是可以给一个`Place`赋值的东西
* `Operand` 是一个给像 `+` 这样的运算符或者一个函数调用的参数
* 一个存放了一个值的拷贝的临时变量

下图描绘了表示之间的交互的一般概述：

<img src="mir_overview.svg">

[点此看大图](mir_detailed.svg)

我们首先将函数体lowering到一个 `Rvalue`，这样我们就可以为 `RETURN_PLACE` 创建一个赋值，
这个`Rvalue`的lowering反过来会触发其参数的`Operand` lowering（如果有的话）
lowering `Operaper`会产生一个`const`操作数，或者移动/复制出`Place`，从而触发`Place` lowering。
如果降低的表达式包含操作，则lowering到`Place`的表达式可以触发创建一个临时变量。
这是蛇咬自己的尾巴的地方，我们需要触发`Rvalue` lowering，以将表达式的值写入本地变量。

## Operator lowering

内置类型的运算符不会lower为函数调用（这将导致无限递归调用，因为trait包含了操作本身）。相反，存在用于二元和一元运算符和索引运算的`Rvalue`。
这些`Rvalue`稍后将生成为llvm基本操作或llvm内部函数。

所有其他类型的运算符都被lower为对运算符对应特征的`impl`的函数调用。

无论采用哪种lower方式，运算符的参数都会lower为`Operand`。
这意味着所有参数都是常量，或者引用局部或静态位置中已经存在的值。

## 方法调用的 lowering

方法调用被降低到与一般函数调用相同的`TerminatorKind`。
在[MIR]中，方法调用和一般函数调用之间不再存在差异。

## 条件

不带字段变量的`enum`的`if`条件判断和`match`语句都会被lower为`TerminatorKind::SwitchInt`。
每个可能的值（如果为`if`条件判断，则对应的值为`0`和`1`）都有一个对应的`BasicBlock`。
分支的参数是表示if条件值的`Operand`。

### 模式匹配

具有字段`enum`的`match`语句也被lower为`TerminatorKind::SwitchInt`，但是操作数是一个`Place`，可以在其中找到该值的判别式。
这通常涉及将判别式读取为新的临时变量。

## 聚合构造

任何类型的聚合值（例如结构或元组）都是通过`Rvalue::Aggregate`建立的。
所有字段都lower为`Operator`。 
从本质上讲，这等效于每个聚合字段都会有一个赋值语句，再加上一个对`enum`的判别式的赋值。

[MIR]: ./index.html
[HIR]: ../hir.html
[THIR]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_build/thir/index.html

[`rustc_mir_build::thir::cx::expr`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_build/thir/cx/expr/index.html
[`mir_built`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir_build/build/fn.mir_built.html
