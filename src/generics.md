# 范型和替换

给定泛型类型`MyType<A, B, ...>`，我们可能希望将泛型参数`A, B, ...`替换为其他一些类型（可能是其他泛型参数或具体类型）。
我们在进行类型推断，类型检查和Trait求解时会做很多这类事情。
从概念上讲，在这些过程中，我们可能会发现一种类型等于另一种类型，并希望将一种类型换成另一种类型，依此类推，直到最终得到一些具体的类型（或错误） 。

在rustc中，这是使用我们上面提到的`SubstsRef`完成的（“substs” = “substitutions”）。
从概念上讲，您可以认为`SubstsRef` 是一个替换ADT泛型类型参数的类型列表。

`SubstsRef`是`List<GenericArg<'tcx>>`的类型别名（请参阅[rust文档中的`List`][list]）。
[`GenericArg`]本质上是[`GenericArgKind`]周围的节省空间的包装器，这是一个枚举，指示类型参数是哪种泛型（类型，生存期或const）。
因此，`SubstsRef`在概念上类似于`&tcx [GenericArgKind <'tcx>]`切片（但它实际上是一个`List`）。

[list]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.List.html
[`GenericArg`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/subst/struct.GenericArg.html
[`GenericArgKind`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/subst/enum.GenericArgKind.html

那么为什么我们使用这种`List`类型而不是真正的slice呢？
它的长度是“内连”的，因此`&List`仅为32位。
结果，它不能被切片（仅在长度超出范围时才起作用）。

这也意味着您可以通过`==`来检查两个`List`的相等性（对于普通切片是不可能的）。
正是因为它们从不代表“子列表”，而仅代表完整的“列表”，该列表已被散列和interned。

综上所述，让我们回到上面的示例：

```rust,ignore
struct MyStruct<T>
```

- `MyStruct`会有一个`AdtDef`（和相应的`DefId`）。
- `T`会有一个`TyKind::Param`（以及相应的`DefId`）（稍后再介绍）。
- 将有一个包含列表`[GenericArgKind::Type(Ty(T))]`的`SubstsRef`。
    - 这里的`Ty(T)`是对`ty::Ty`的简写，其中有`TyKind::Param`，我们在之前提到过这一点。
- 这是一个`TyKind::Adt`，其中包含`MyStruct`的`AdtDef`和上面的`SubstsRef`。

最后，我们将快速提到[`Generics`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/struct.Generics.html)类型。 它用于提供某个类型的类型参数的信息。

### 替换前的范型

因此，回想一下，在我们的示例中，`MyStruct`结构具有范型`T`。
例如，当我们对使用`MyStruct`的函数进行类型检查时，我们将需要能够在不真正知道`T`是什么的情况下引用该类型`T`。
总的来说，在所有泛型定义中都是如此：我们需要能够处理未知类型。 这是通过`TyKind::Param`（我们在上面的示例中提到的）完成的。

每个`TyKind::Param`都包含两个字段：名称和索引。
通常，索引完全定义了参数，并且大多数代码都使用该索引。
名称则包含在调试打印输出中。

这么做有两个原因。
首先，索引很方便，它使您可以在替换时将其包含在通用参数列表中。
其次，索引鲁棒性更强。 例如，原则上可以有两个使用相同名称的不同类型参数，例如 `impl<A> Foo<A> {
fn bar<A>() { .. } }`，尽管禁止阴影的规则使此操作变得困难（但是将来这些语言规则可能会更改）。

类型参数的索引是一个整数，指示其在类型参数列表中的顺序。 此外，我们认为该列表包括来自外部作用域的所有类型参数。
考虑以下示例：

```rust,ignore
struct Foo<A, B> {
  // A would have index 0
  // B would have index 1

  .. // some fields
}
impl<X, Y> Foo<X, Y> {
  fn method<Z>() {
    // inside here, X, Y and Z are all in scope
    // X has index 0
    // Y has index 1
    // Z has index 2
  }
}
```

当我们在泛型定义中工作时，我们将像其他`TyKind`一样使用`TyKind::Param`。
毕竟这只是一种类型。
但是，如果我们想在某个地方使用范型，那么我们将需要进行替换。

例如，假设前面示例中的`Foo <A, B>`类型的字段为`Vec<A>`。
请注意，`Vec`也是通用类型。
我们要告诉编译器，应将`Vec`的类型参数替换为`Foo<A,B>`的`A`类型参数。我们通过替换来做到这一点：

```rust,ignore
struct Foo<A, B> { // Adt(Foo, &[Param(0), Param(1)])
  x: Vec<A>, // Adt(Vec, &[Param(0)])
  ..
}

fn bar(foo: Foo<u32, f32>) { // Adt(Foo, &[u32, f32])
  let y = foo.x; // Vec<Param(0)> => Vec<u32>
}
```

这个例子有一些不同的替代：

- 在`Foo`的定义中，在字段`x`的类型中，将`Vec`的类型参数替换为`Param(0)`，即`Foo<A, B>`的第一个参数，因此`x`的类型是`Vec <A>`。
- 在函数`bar`上，我们指定要使用`Foo<u32, f32>`。这意味着我们将用`u32`和`f32`替换`Param(0)`和 `Param(1)`。
- 在`bar`的函数体中，我们访问`foo.x`，其类型为`Vec<Param(0)>`，但`Param(0)`已经被替换为`u32`，因此，`foo.x `的类型为`Vec<u32>`。

让我们更仔细地看看最后的替换方法，以了解为什么使用索引。如果要查找`foo.x`的类型，则可以获取x的范型，即`Vec<Param(0)>`。
现在我们可以使用索引`0`，并使用它来查找正确的类型替换：查看`Foo`的`SubstsRef`，我们有列表`[u32, f32]`，
因为我们要替换索引`0 `，我们采用此列表的第0个索引，即`u32`。然后就好了！

您可能有几个后续问题……

 **`type_of`** 我们如何获得`x`的范型？您可以通过 `tcx.type_of(def_id)` 查询获得几乎所有类型的东西，在这种情况下，我们将传递字段`x`的`DefId`。
`type_of`查询总是返回带有定义范围内的泛型的定义。
例如，`tcx.type_of(def_id_of_my_struct)`将返回`MyStruct`的“自视图”：`Adt(Foo, &[Param(0), Param(1)])`。

**`subst`** 我们如何实际地进行替换？也有一个用来这么做的函数！您可以使用[`subst`](https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/subst/trait.Subst.html)将`SubstRef`替换为其他类型的列表。

[这里是在编译器中实际使用`subst`的示例][substex]。
确切的细节并不是太重要，但是在这段代码中，我们碰巧将其从`rustc_hir::Ty`转换为真实的`ty::Ty`。
您可以看到我们首先得到了一些替换（`substs`）。然后我们调用`type_of`来获取类型，并调用`ty.subst(substs)`来获得新的`ty`类型，并进行替换。

[substex]: https://github.com/rust-lang/rust/blob/597f432489f12a3f33419daa039ccef11a12c4fd/src/librustc_typeck/astconv.rs#L942-L953

**关于索引的注释：**`Param`中的索引可能与我们期望的不匹配。
例如，索引可能超出范围，或者可能是我们期望类型时却得到了一个生命周期的索引。
从`rustc_hir::Ty`转换为`ty::Ty`时或者更早，编译器会捕获这些错误。
如果它们在那以后发生，那就是编译器错误。


