# 查询: 需求驱动的编译

如[编译器高级概述][hl]中所述，Rust编译器当前正在从传统的“基于pass”的设置过渡到“需求驱动”的系统。
**编译器查询系统是我们新的需求驱动型组织的关键。**
背后的想法很简单。 您有各种查询来计算有关输入的内容
– 例如，有一个名为`type_of(def_id)`的查询，给定某项的[def-id]，它将计算该项的类型并将其返回给您。

[def-id]: appendix/glossary.md#def-id
[hl]: compiler-src.html

查询执行是“**记忆式**”的 —— 因此，第一次调用查询时，它将执行计算，但是下一次，结果将从哈希表中返回。
此外，查询执行非常适合“**增量计算**”； 大致的想法是，当您执行查询时，**可能**会通过从磁盘加载存储的数据来将结果返回给您（但这是一个单独的主题，我们将不在此处进一步讨论）。

总体愿景是，最终，整个编译器控制流将由查询驱动。
实际上，将有一个顶级查询（“编译”）在一个crate上运行编译。
这反过来会要求从最底层开始的有关该crate的信息。 例如：

- 此“编译”查询可能需要获取代码生成单元列表（即需要由LLVM编译的模块）。
- 但是计算代码生成单元列表将调用一些子查询，该子查询返回Rust源代码中定义的所有模块的列表。
- 该查询会调用一些要求HIR的内容。
- 这会越来越远，直到我们完成实际的parsing。

但是，这一愿景尚未完全实现。 尽管如此，编译器的大量代码（例如，生成MIR）仍然完全像这样工作。

### 增量编译的详细说明

[增量编译的详细说明][查询模型]一章提供了关于什么是查询及其工作方式的深入描述。
如果您打算编写自己的查询，那么可以读一读这一章节。

### 调用查询

调用查询很简单。 tcx（“类型上下文”）为每个定义的查询提供了一种方法。 因此，例如，要调用`type_of`查询，只需执行以下操作：

```rust,ignore
let ty = tcx.type_of(some_def_id);
```

### 编译器如何执行查询

您可能想知道调用查询方法时会发生什么。
答案是，对于每个查询，编译器都会维护一个缓存——如果您的查询已经执行过，那么我们将简单地从缓存中复制上一次的返回值并将其返回
（因此，您应尝试确保查询的返回类型可以低成本的克隆；如有必要，请插入`Rc`）。

#### Providers

但是，如果查询不在缓存中，则编译器将尝试找到合适的**provider**。
provider是已定义并链接到编译器的某个函数，其包含用于计算查询结果的代码。

**Provider是按crate定义的。**
编译器至少在概念上在内部维护每个crate的provider表。
目前，实际上有两组表：用于查询“**本地crate**”的provider（即正在编译的crate）和用于查询“**外部crate**”（即正在编译的crate的依赖） 的provider。
请注意，确定查询所在的crate的类型不是查询的*类型*，而是*键*。
例如，当您调用`tcx.type_of(def_id)`时，它可以是本地查询或外部查询，
具体取决于`def_id`所指的crate（请参阅[`self::keys::Key`][Key] trait 以获取有关其工作原理的更多信息）。

Provider 始终具有相同的签名：

```rust,ignore
fn provider<'tcx>(
    tcx: TyCtxt<'tcx>,
    key: QUERY_KEY,
) -> QUERY_RESULT {
    ...
}
```

提供者采用两个参数：`tcx`和查询键。 并返回查询结果。

####如何初始化provider

创建tcx时，它的创建者会使用[`Providers`][providers_struct]结构为它提供provider。
此结构是由此处的宏生成的，但基本上就是一大堆函数指针：

[providers_struct]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/query/struct.Providers.html

```rust,ignore
struct Providers {
    type_of: for<'tcx> fn(TyCtxt<'tcx>, DefId) -> Ty<'tcx>,
    ...
}
```

目前，我们为本地crate提供一份该结构的副本，为所有外部crate提供一份该结构的副本，尽管计划是最终可能为每个crate提供一份。

这些`Provider`结构最终是由`librustc_driver`创建并填充的，但是它是通过将工作分配给其他`rustc_*`crate来完成的。
这是通过调用各种[`provide`][provide_fn]函数来完成的。 这些函数看起来像这样：

[provide_fn]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/hir/fn.provide.html

```rust,ignore
pub fn provide(providers: &mut Providers) {
    *providers = Providers {
        type_of,
        ..*providers
    };
}
```

也就是说，他们使用一个 `&mut Providers` 并对其进行in place的修改。
通常我们使用上面的写法只是因为它看起来比较漂亮，但是您也可以`providers.type_of = type_of`，这是等效的。
（在这里，`type_of` 将是一个顶层函数，如我们之前看到的那样定义。）
因此，如果我们想为其他查询添加provider，我们可以在上面的crate中将其称为“ fubar”，我们可以修改 `provide()`函数如下：

```rust,ignore
pub fn provide(providers: &mut Providers) {
    *providers = Providers {
        type_of,
        fubar,
        ..*providers
    };
}

fn fubar<'tcx>(tcx: TyCtxt<'tcx>, key: DefId) -> Fubar<'tcx> { ... }
```

注意大多数`rustc_*` crate仅提供**local provider**。
几乎所有的**extern provider**都会通过[`rustc_metadata` crate][rustc_metadata] 进行处理，后者会从crate元数据中加载信息。
但是在某些情况下，某些crate可以既提供本地也提供外部crate查询，
在这种情况下，它们会定义`rustc_driver`可以调用的[`provide`][ext_provide]和[`provide_extern`][ext_provide_extern]函数。

[rustc_metadata]: https://github.com/rust-lang/rust/tree/master/src/librustc_metadata
[ext_provide]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_llvm/attributes/fn.provide.html
[ext_provide_extern]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_codegen_llvm/attributes/fn.provide_extern.html

### 添加一种新的查询

假设您想添加一种新的查询，您该怎么做？
定义查询分为两个步骤：

1. 首先，必须指定查询名称和参数； 然后，
2. 您必须在需要的地方提供查询提供程序。

要指定查询名称和参数，您只需将条目添加到
[`src/librustc_middle/query/mod.rs`][query-mod]中的大型宏调用之中，类似于：

[query-mod]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/query/index.html

```rust,ignore
rustc_queries! {
    Other {
        /// Records the type of every item.
        query type_of(key: DefId) -> Ty<'tcx> {
            cache { key.is_local() }
        }
    }

    ...
}
```

查询分为几类（`Other`，`Codegn`，`TypeChecking`等）。
每组包含一个或多个查询。 每个查询的定义都是这样分解的：

```rust,ignore
query type_of(key: DefId) -> Ty<'tcx> { ... }
^^    ^^^^^^^      ^^^^^     ^^^^^^^^   ^^^
|     |            |         |          |
|     |            |         |          查询修饰符
|     |            |         查询的结果类型
|     |            查询的 key 的类型
|     查询名称
query关键字
```

让我们一一介绍它们：

- **query关键字：** 表示查询定义的开始。
- **查询名称：**查询方法的名称（`tcx.type_of(..)`）。也用作将生成以表示此查询的结构的名称（`ty::queries::type_of`）。
- **查询的 key 的类型：**此查询的参数类型。此类型必须实现[`ty::query::keys::Key`][Key] trait，该trait定义了（例如）如何将其映射到crate，等等。
- **查询的结果类型：** 此查询产生的类型。
这种类型应该（a）不使用`RefCell`或其他内部可变性模式，并且
（b）可以廉价地克隆。对于非平凡的数据类型，建议使用Interning方法或使用`Rc`或`Arc`。
  - 一个例外是`ty::steal::Steal`类型，该类型用于廉价地修改MIR。
有关更多详细信息，请参见`Steal`的定义。不应该在不警告`@rust-lang/compiler`的情况下添加对`Steal`的新的使用。
- **查询修饰符：** various flags and options that customize how the
  query is processed (mostly with respect to [incremental compilation][incrcomp]).

[Key]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/query/keys/trait.Key.html
[incrcomp]: queries/incremental-compilation-in-detail.html#query-modifiers

因此，要添加查询：

- 使用上述格式在`rustc_queries!`中添加一个条目。
- 通过修改适当的`provide`方法链接provider； 或根据需要添加一个新文件，并确保`rustc_driver`会调用它。

#### 查询结构体和查询描述

对于每种类型，`rustc_queries`宏都会生成一个以查询命名的“查询结构体”。
此结构体是描述查询的一种占位符。 每个这样的结构都要实现[`self::config::QueryConfig`][QueryConfig] trait，
该trait具有与该特定查询的键/值相关的类型。
基本上，生成的代码如下所示：

```rust,ignore
// Dummy struct representing a particular kind of query:
pub struct type_of<'tcx> { data: PhantomData<&'tcx ()> }

impl<'tcx> QueryConfig for type_of<'tcx> {
  type Key = DefId;
  type Value = Ty<'tcx>;

  const NAME: QueryName = QueryName::type_of;
  const CATEGORY: ProfileCategory = ProfileCategory::Other;
}
```

您可能希望实现一个额外的trait，称为[`self::config::QueryDescription`][QueryDescription]。
这个trait是用于在发生cycle错误时使用，为查询提供一个“人类可读”的名称，以便我们可以探明在cycle发生的情况。
如果查询键是`DefId`，则实现此特征是可选的，但是如果*不*实现它，则会得到一个相当普通的错误（“processing `foo` ...”）。
您可以将新的impl放入`config`模块中。 他们看起来像这样：

[QueryConfig]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/ty/query/trait.QueryConfig.html
[QueryDescription]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_query_system/query/config/trait.QueryDescription.html

```rust,ignore
impl<'tcx> QueryDescription for queries::type_of<'tcx> {
    fn describe(tcx: TyCtxt, key: DefId) -> String {
        format!("computing the type of `{}`", tcx.def_path_str(key))
    }
}
```

另一个选择是添加`desc`修饰符：

```rust,ignore
rustc_queries! {
    Other {
        /// Records the type of every item.
        query type_of(key: DefId) -> Ty<'tcx> {
            desc { |tcx| "computing the type of `{}`", tcx.def_path_str(key) }
        }
    }
}
```

`rustc_queries` 宏会自动生成合适的 `impl`。

[查询模型]: queries/incremental-compilation-in-detail.md
