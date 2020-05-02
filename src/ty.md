# `ty` 模块：类型的表示

`ty`模块定义了Rust编译器如何在内部表示类型。 它还定义了*类型上下文*（`tcx`或`TyCtxt`），这是编译器中的中央数据结构。

## `ty::Ty`

当我们谈论rustc如何表示类型时，我们通常指的是称为`Ty`的类型。 编译器中有很多`Ty`的模块和类型（[Ty 文档][ty]）。

[ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/index.html

我们指的`Ty`是[`rustc::ty:: Ty`][ty_ty]（而不是[`rustc_hir::Ty`][hir_ty]）。它们之间的区别很重要，因此我们将在讨论`ty::Ty`之前先进行讨论。

[ty_ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/type.Ty.html
[hir_ty]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Ty.html

## `rustc_hir::Ty` vs `ty::Ty`

rustc中的HIR可以看作是高级中间表示。
它或多或少是一种AST（请参阅[本章](hir.md)），因为它代表用户编写的语法，并且是在语法分析和一些*desugaring*之后获得的。
它具有类型的表示形式，但实际上它反映了用户编写的内容，即他们为表示该类型而编写的内容。

相反，`ty::Ty`表示类型的语义，即用户编写内容的*含义*。
例如，`rustc_hir::Ty`会记录用户在程序中使用了两次`u32`这个名字，但是`ty::Ty`会记录两种用法都指向同一类型。

**例如： `fn foo(x: u32) → u32 { }`** 在这个函数中，我们看到`u32`出现了两次。
我们知道这是同一类型，即该函数接受一个参数并返回相同类型的参数，但是从HIR的角度来看，将存在两个不同的类型实例，因为它们分别在程序中的两个不同位置出现。
也就是说，它们有两个不同的[`Span`][span]（位置）。

[span]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_span/struct.Span.html

**例如： `fn foo(x: &u32) -> &u32)`** 另外，HIR可能遗漏了信息。
`&u32`类型是不完整的，因为在完整的Rust类型中实际上这里应该存在一个生命周期，但是我们不需要编写这些生命周期。
还有一些省略规则可以插入信息。
结果可能看起来像`fn foo<'a>(x: &'a u32) -> &'a u32)`.

在HIR级别上，这些内容并未阐明。
但是，在`ty::Ty`级别，添加了这些详细信息。
此外，对于给定类型，我们将只有一个`ty::Ty`，例如`u32`，并且该`ty::Ty`用于整个程序中的所有u32，而不是只在特定场景中使用，这与 `rustc_hir::Ty`不同。

这里有一个总结：

| [`rustc_hir::Ty`][hir_ty] | [`ty::Ty`][ty_ty] |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 描述类型的*语法*：用户写的内容（去除了一些语法糖）。 | 描述一种类型的“语义”：用户写的内容的含义。 |
| 每个`rustc_hir::Ty`都有自己的span，对应于程序中的适当位置。 | 与用户程序中的单个位置不对应。 |
| `rustc_hir::Ty`具有泛型和生命周期； 但是，其中一些生命周期是特殊标记，例如[`LifetimeName::Implicit`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/enum.LifetimeName.html#variant.Implicit)。 | `ty::Ty`具有完整的类型，包括泛型和生命周期，即使用户忽略了它们 |
| `fn foo(x: u32) → u32 { }` —— 两个`rustc_hir::Ty`代表了`u32`的两次不同的使用。 每个都有自己的`Span`等。——`rustc_hir::Ty`不能告诉我们两者是同一类型 | 整个程序中所有`u32`是同一个`ty::Ty`。——`ty::Ty`告诉我们，`u32`的两次使用表示相同的类型。 |
| `fn foo(x: &u32) -> &u32)` —— 仍然有两个`rustc_hir::Ty`。 —— 在`rustc_hir::Ty`中这两个引用的生命期使用特殊标记[`LifetimeName::Implicit](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/enum.LifetimeName.html#variant.Implicit)表示。 | `fn foo(x: &u32) -> &u32)` —— 单个`ty::Ty`。—— `ty::Ty`具有隐藏的生命周期参数 |


**次序**  HIR是直接从AST构建的，因此会在生成任何`ty::Ty`之前发生。
构建HIR之后，将完成一些基本的类型推断和类型检查。
在类型推断过程中，我们找出所有事物的`ty::Ty`是什么，并且还要检查某事物的类型是否不明确。
然后，`ty::Ty`将用于类型检查，来确保所有内容都具有预期的类型。
[`astconv`模块][astconv]是负责将`rustc_hir::Ty`转换为`ty::Ty`的代码所在的位置。 
这发生在类型检查阶段，但也发生在编译器的其他部分，例如“该函数需要什么样的参数类型”之类的问题。

[astconv]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_typeck/astconv/index.html

**语义如何驱动两个`Ty`实例** 您可以将HIR视为对类型信息假设最少的视角。
我们假设两件事是截然不同的，直到证明它们是同一件事为止。
换句话说，我们对它们的了解较少，因此我们应该对它们的假设较少。


从文法上讲,第N行第20列的`"u32"`和第N行第35列的`"u32"`是两个字符串。我们尚不知道它们是否相同。 因此，在HIR中，我们将它们视为不同的。
后来，我们确定它们在语义上是相同的类型，这就是我们使用`ty::Ty`的地方。

考虑另一个例子： `fn foo<T>(x: T) -> u32` 假设有人调用了 `foo::<u32>(0)`。

这意味着`T`和`u32`（在本次调用中）实际上是相同的类型，因此最终我们最终将得到相同的`ty::Ty`，但是我们有截然不同的`rustc_hir::Ty`。

（不过，这有点过于简化了，因为在类型检查过程中，我们将对函数范型检查，并且仍然具有不同于`u32`的`T`。
之后，在进行代码生成时，我们将始终处理每个函数的“单态化"（完全替换的）版本，因此我们将知道`T`代表什么（特别是它是`u32`）。

这里还有一个例子：

```rust
mod a {
    type X = u32;
    pub fn foo(x: X) -> i32 { 22 }
}
mod b {
    type X = i32;
    pub fn foo(x: X) -> i32 { x }
}
```

显然，这里的`X`类型将根据上下文而变化。 如果查看`rustc_hir::Ty`，您会发现`X`在两种情况下都是别名（尽管它将通过名称解析映射到不同的别名）。
但是，如果您查看`ty::Ty`中的函数签名，它将是 `fn(u32) -> u32`或`fn(i32) -> i32`（类型别名已完全展开）。

## `ty::Ty` 的实现

[`rustc::ty::Ty`][ty_ty]实际上是[`&TyS`][tys]的类型别名（稍后会详细介绍）。
`TyS`（Type Structure）是主要功能所在的位置。
您通常可以忽略`TyS`结构；您基本上永远不会显式访问它。我们总是使用`Ty`别名通过引用传递它。
唯一的例外是在类型上定义固有方法。
特别地，`TyS`具有类型为[`TyKind`][tykind]的[`kind`][kind]字段，其表示关键类型信息。
`TyKind`是一个很大的枚举，代表了不同类型的类型（例如原生类型，引用，抽象数据类型，泛型，生命周期等）。
`TyS`还有另外2个字段：`flags`和`outer_exclusive_binder`。
它们是提高效率的便捷工具，可以汇总有关我们可能想知道的类型的信息，但本文并不多涉及这部分内容。
最后，`ty::TyS`是[interned](./memory.md)的，以便使`ty::TyS`可以是类似于指针的瘦类型。这使我们能够进行低成本的相等比较，以及其他的interning的好处。

[tys]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyS.html
[kind]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyS.html#structfield.kind
[tykind]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html

## 分配和使用类型

要分配新类型，可以使用在`tcx`上定义的各种`mk_`方法。 它们的名称主要对应于各种类型。 例如：

```rust,ignore
let array_ty = tcx.mk_array(elem_ty, len * 2);
```

这些方法都返回`Ty<'tcx>` —— 注意，返回的生命周期是该`tcx`可以访问的生命周期。 类型总是被规范化和interned（因此我们永远不会两次分配完全相同的类型）。

> 注意 由于类型是interned的，因此可以使用`==`高效地比较它们是否相等 —— 但是，除非您碰巧正在散列并寻找重复项，否则您应该不会希望这么做。 
这是因为在Rust中通常有多种方法来表示同一类型，特别是一旦涉及到类型推断。
如果要测试类型相等性，则可能需要开始研究类型推倒的代码才能正确完成。

您还可以通过访问`tcx.types.bool`，`tcx.types.char`等来在`tcx`中找到各种常见类型（有关更多信息，请参见 [`CommonTypes`]。）。

[`CommonTypes`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.CommonTypes.html

## `ty::TyKind` 的变体

注意：`TyKind` **并非** *Kind*的函数式编程概念。

每当在编译器中使用`Ty`时，通常会在类型上进行匹配：

```rust,ignore
fn foo(x: Ty<'tcx>) {
  match x.kind {
    ...
  }
}
```

`kind`字段的类型为`TyKind<'tcx>`，它是一个枚举，用于定义编译器中所有不同种类的类型。

> N.B. 在类型推断过程中检查类型的`kind`字段可能会很冒险，因为可能会有推断变量和其他要考虑的因素，或者有时类型未知，并且稍后将变得已知。

相关类型的很多，我们会及时介绍（例如，区域/生命周期，“替代”等）。

`TyKind`枚举上有很多变体，您可以通过查看rustdocs来看到。 这是一个样本：


[**代数数据类型（ADT）**]() [*代数数据类型*][wikiadt]是`struct`，`enum`或`union`。 
实际上，`struct`，`enum`和`union`是用相同的方式实现的：它们都是[`ty::TyKind::Adt`][kindadt]类型。
这基本上是用户定义的类型。稍后我们将详细讨论。

[**Foreign**][kindforeign] 对应 `extern type T`.

[**Str**][kindstr] 是str类型。当用户编写`&str`时，`Str`是我们表示该类型的`str`部分的方式。

[**Slice**][kindslice] 对应 `[T]`.

[**Array**][kindarray] 对应 `[T; n]`.

[**RawPtr**][kindrawptr] 对应 `*mut T` 或者 `*const T`

[**Ref**][kindref] `Ref`代表安全的引用，`&'a mut T`或`&'a T`。
`Ref`具有一些相关类型，例如，`Ty<tcx>`是引用所引用的类型，`Region<tcx>`是引用的生命周期或区域，`Mutability`则是引用的可变性。 

[**Param**][kindparam] 代表类型参数，如`Vec<T>`中的`T`。

[**Error**][kinderr] 在某处表示类型错误，以便我们可以打印出更好的诊断信息。 我们将在后面讨论它。

[**以及更多**...][kindvars]

[wikiadt]: https://en.wikipedia.org/wiki/Algebraic_data_type
[kindadt]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Adt
[kindforeign]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Foreign
[kindstr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Str
[kindslice]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Slice
[kindarray]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Array
[kindrawptr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.RawPtr
[kindref]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Ref
[kindparam]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Param
[kinderr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variant.Error
[kindvars]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/enum.TyKind.html#variants

## Import 约定

尽管没有硬性规定，但是`ty`模块的用法通常如下：

```rust,ignore
use ty::{self, Ty, TyCtxt};
```

由于`Ty`和`TyCtxt`类型使用得非常普遍，因此可以直接导入。
其他类型通常使用显式的`ty::`前缀来引用（例如`ty::TraitRef<'tcx>`）。
但是某些模块选择显式导入更大或更小的名称集。

## ADT的表示

让我们考虑像`MyStruct<u32>`这样的类型的例子，其中MyStruct的定义如下：

```rust,ignore
struct MyStruct<T> { x: u32, y: T }
```

类型`MyStruct<u32>`将是`TyKind::Adt`的实例：

```rust,ignore
Adt(&'tcx AdtDef, SubstsRef<'tcx>)
//  ------------  ---------------
//  (1)            (2)
//
// (1) 表示 `MyStruct` 部分
// (2) 表示 `<u32>`, 或者 "substitutions" / 范型参数
```

有两个部分：

- [`AdtDef`][adtdef]引用struct/enum/union，但没有类型参数的值。
在我们的示例中，这是MyStruct部分，*没有*参数u32。
  - 请注意，在HIR中，结构体，枚举和union的表示方式是不同的，但是在`ty::Ty`中，它们均使用`TyKind::Adt`表示。
- [`SubstsRef`][substsref]是要替换的范型参数值的内部列表。
在我们的`MyStruct<u32>`的示例中，我们会得到一个类似`[u32]`的列表。
     稍后，我们将进一步探讨泛型和替换。

[adtdef]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.AdtDef.html
[substsref]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/subst/type.SubstsRef.html

**`AdtDef` 和 `DefId`**

对于源代码中定义的每种类型，都有一个唯一的`DefId`（请参阅[本章](hir.md#identifiers-in-the-hir)）。
这包括ADT和泛型。 在上面给出的`MyStruct<T>`定义中，有两个`DefId`：一个用于`MyStruct`，一个用于`T`。
注意，上面的代码不会为`u32`生成新的`DefId`，因为该代码并不定义`u32`（而仅是引用它）。

`AdtDef`或多或少是`DefId`的包装，其中包含许多有用的辅助方法。
`AdtDef`和`DefId`之间本质上是一对一的关系。
您可以通过[`tcx.adt_def(def_id)`查询][adtdefq]`DefId`对应的`AdtDef`。 所有`AdtDef`都被缓存了（您可以看到其上的`'tcx`生命周期）。

[adtdefq]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.TyCtxt.html#method.adt_def


## 类型错误

用户制造了类型错误时会生成`TyKind::Error`。
我们的想法是，我们将传播这种类型并抑制由于该类型而引起的其他错误，以免级联的编译器错误消息使用户不知所措。

`TyKind::Error`的使用有一个**重要的原则**。
除非您知道已经向用户报告了错误，否则您**绝不要**返回“错误类型”。
通常是因为（a）您刚刚在此报告了该错误，或者（b）您正在传播现有的Error类型（在这种情况下，应该在生成该错误类型时报告该错误）。

此原则非常重要，因为`Error`类型的全部目的就是抑制其他错误——即，我们不报告它们。
如果我们在不向用户实际制造了错误的情况下生成`Error`类型，则这可能导致以后的错误被抑制，并且编译可能会无意中成功！

有时还有第三种情况。
您认为已报告了一个错误，但是您认为该错误将在编译的更早阶段而不是现在得到报告。
在这种情况下，您可以调用[`delay_span_bug`]，这表示编译应该会产生错误——如果编译意外地成功了，则将触发编译器错误报告。

[`delay_span_bug`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_session/struct.Session.html#method.delay_span_bug

## 问题：为什么在`AdtDef`“内部”做替换？

回想一下，我们用`(AdtDef，substs)`表示一个范型结构体。 那么，为什么要使用这种麻烦的模式？

我们可以选择表示这种类型的另一种方法是始终创建一个新的，完全不同的`AdtDef`形式，其中所有类型都已被替换。
这样做好像比较方便。 但是，`(AdtDef，substs)`方案对此有一些优势。

首先，`(AdtDef，substs)`方案可以提高效率：

```rust,ignore
struct MyStruct<T> {
  ... 100s of fields ...
}

// Want to do: MyStruct<A> ==> MyStruct<B>
```

在像这样的示例中，只需将对`A`的一个引用替换为`B`，就可以低成本地地将`MyStruct<A>`替换为`MyStruct<B>`（依此类推）。 
但是，如果我们替换所有字段，则可能需要多做很多工作，我们可能必须遍历`AdtDef`中的所有字段并更新所有类型。

更深入一点来说，Rust中的结构体是[*nominal* 类型][nominal]——这意味着它们是由其名称定义的（然后它们的内容将从该名称的定义中进行索引，而不是携带在类型本身“内”）。

[nominal]: https://en.wikipedia.org/wiki/Nominal_type_system
