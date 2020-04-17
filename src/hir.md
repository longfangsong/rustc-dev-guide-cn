# HIR

HIR ——“高级中间表示” ——是大多数rustc组件中使用的主要IR。
它是抽象语法树（AST）的编译器友好表示形式，该结构在语法分析，宏扩展和名称解析之后生成（有关如何创建HIR，请参见[Lowering](./lowering.html)）。
HIR的许多部分都非常类似于Rust表面语法，但是Rust的某些表达形式已被“去糖”。
例如，`for`循环将转换为了`loop`，因此不会出现在HIR中不会出现`for`。 这使HIR比普通AST更易于分析。

本章介绍了HIR的主要概念。

您可以通过将`-Zunpretty=hir-tree`标志传递给rustc来查看代码的HIR表示形式：

```bash
cargo rustc -- -Zunpretty=hir-tree
```

### 带外存储和`Crate`类型

HIR中的顶层数据结构是[`Crate`]，它存储当前正在编译的crate的内容（我们只为当前crate构造HIR）。
在AST中，crate数据结构基本上只包含根模块，而HIR`Crate`结构则包含许多map和其他用于组织crate内容以便于访问的数据。

[`Crate`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Crate.html

例如，HIR中单个项目的内容（例如模块，功能，特征，隐含等）不能在父级中立即访问。
因此，例如，如果有一个包含函数`bar()`的模块项目`foo`：

```rust
mod foo {
    fn bar() { }
}
```

那么在模块`foo`的HIR中表示（[`Mod`]结构）中将只有`bar()`的**`ItemId`** `I`。
要获取函数`bar()`的详细信息，我们将在`items`映射中查找`I`。

[`Mod`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Mod.html

这种表示的一个很好的结果是，可以通过遍历这些映射中的键值对来遍历crate中的所有项目（而无需遍历整个HIR）。
对于trait项和impl项以及“实体”（如下所述）也有类似的map。

使用这种表示形式的另一个原因是为了更好地与增量编译集成。
这样，如果您想要访问[`&rustc_hir::Item`]（例如mod`foo`），则不能立即访问函数`bar()`的内容。
相反，您只能访问`bar()`的**id**，并且必须调用要求id作为参数的某些函数来查找`bar`的内容。 这使编译器有机会观察到您访问了`bar()`的数据，然后记录依赖。

[`&rustc_hir::Item`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Item.html

<a name="hir-id"></a>

### HIR中的标识符

大多数必须处理HIR中事物的代码都倾向于不携带对HIR的引用，而是携带*标识符号*（或“ids”）。现在，您会发现四种正在使用的标识符：

- [`DefId`]，主要标识“定义”或顶层项目。
  - 您可以认为[`DefId`]是非常明确和完整路径的简写，例如`std::collections::HashMap`。
但是，这些路径能够命名在普通Rust中无法命名的东西（例如impls），并且它们还包含有关crate的其他信息（例如其版本号，因为同一crate的两个版本可以共存）。
  - 一个[`DefId`]实际上由两部分组成，一个`CrateNum`（标识一个crate）和一个`DefIndex`（索引到每个箱子中维护的item列表）。
- [`HirId`]，它将特定item的索引与该item内的偏移量结合在一起。
  - [`HirId`]的关键点是它是相对于某些其他项目（通过[`DefId`]标识）的偏移量。
- [`BodyId`]引用了板条箱中的特定项目的实际内容（函数或常量的定义）。
当前，它实际上是“ newtype'd”的 [`HirId`]。
- [`NodeId`]，它是一个绝对ID，用于标识HIR树中的单个节点。
  - 尽管这种标识符仍很常用，但**它们正在被逐步淘汰**。
  - 由于它们在crate中是绝对的，因此在树中的任何位置添加新节点都会导致包装箱中所有后续代码的[`NodeId`]发生更改。
你可能已经看出来了，这对增量编译极为不利。

[`DefId`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/def_id/struct.DefId.html
[`HirId`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/hir_id/struct.HirId.html
[`BodyId`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.BodyId.html
[`NodeId`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/node_id/struct.NodeId.html

我们还有一个内部map，是从`DefId`到所谓的 “Def path”的映射。 
“ Def path”就像一个模块路径，但是内容更为丰富。
例如，可能是`crate::foo::MyStruct`唯一标识此定义。
它与模块路径有些不同，因为它可能包含类型参数`T`，例如`crate::foo::MyStruct::T`，在普通的Rust中，就不能这么写。
这些用于增量编译。

### HIR Map

在大多数情况下，当您使用HIR时，您将通过 **HIR Map**进行操作，该map可通过[`tcx.hir_map`]在tcx中访问（并在[`hir::map`]模块中定义）。
[HIR map]包含[多种方法]，用于在各种ID之间进行转换并查找与HIR节点关联的数据。

[`tcx.hir_map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/context/struct.GlobalCtxt.html#structfield.hir_map
[`hir::map`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/index.html
[HIR map]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html
[多种方法]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#methods

例如，如果您有[`DefId`]，并且想将其转换为[`NodeId`]，则可以使用[`tcx.hir.as_local_node_id(def_id)`][as_local_node_id]。
这将返回一个`Option<NodeId>` —— 如果def-id引用了当前crate之外的内容（因为这种内容没有HIR节点），则将为`None`；
否则返回`Some(n)`，其中`n`是定义的节点ID。

[as_local_node_id]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.as_local_node_id

同样，您可以使用[`tcx.hir.find(n)`][find]在节点上查找[`NodeId`]。
这将返回一个`Option<Node<'tcx>>`，其中[`Node`]是在map中定义的枚举。

通过对此进行匹配，您可以找出node-id所指的节点类型，并获得指向数据本身的指针。
通常，您知道节点`n`是哪种类型——例如 如果您知道`n`必须是某些HIR表达式，
则可以执行[`tcx.hir.expect_expr(n)`][expect_expr]，它将提取并返回[`&hir::Expr`][Expr]，此时如果`n`实际上不是一个表达式，那么会panic。

[find]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.find
[`Node`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/enum.Node.html
[expect_expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.expect_expr
[Expr]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Expr.html

最后，您可以通过[`tcx.hir.get_parent_node(n)`][get_parent_node]之类的调用，使用HIR map来查找节点的父节点。

[get_parent_node]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.get_parent_node

### HIR Bodies

[`rustc_hir::Body`]代表某种可执行代码，例如函数/闭包的主体或常量的定义。
body与一个**所有者**相关联，“所有者”通常是某种Item（例如，`fn()`或`const`），但也可以是闭包表达式（例如， `|x, y| x + y`）。
您可以使用HIR映射来查找与给定def-id（[`maybe_body_owned_by`]）关联的body，或找到body的所有者（[`body_owner_def_id`]）。

[`rustc_hir::Body`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_hir/struct.Body.html
[`maybe_body_owned_by`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.maybe_body_owned_by
[`body_owner_def_id`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/map/struct.Map.html#method.body_owner_def_id
