# MIR passes

如果您想获取某个函数（或常量，等等）的MIR，则可以使用`optimized_mir(def_id)`查询。
这将返回给您最终的，优化的MIR。 对于外部def-id，我们只需从其他crate的元数据中读取MIR。
但是对于本地def-id，查询将构造MIR，然后通过应用一系列pass来迭代优化它。 本节描述了这些pass的工作方式以及如何扩展它们。

为了为给定的def-id`D`生成`optimized_mir(D)`，MIR会通过几组优化，每组均由一个查询表示。
每个套件都包含多个优化和转换。
这些套件代表有用的中间点，我们可以在这些中间点访问MIR以进行类型检查或其他目的：

- `mir_build(D)` —— 这不是一个查询，而是构建初始MIR
- `mir_const(D)` —— 应用一些简单的转换以使MIR准备进行常量求值；
- `mir_validated(D)` —— 应用更多的转换，使MIR准备好进行借用检查；
- `optimized_mir(D)` —— 完成所有优化后的最终状态。

### 实现并注册一个pass

`MirPass`是处理MIR的一些代码，通常（但不总是）以某种方式对其进行转换。 例如，它可能会执行优化。
`MirPass` trait本身可在[`rustc_mir::transform`模块][mirtransform]中找到，它基本上由一个`run_pass`方法组成，
该方法仅要求一个`&mut Mir`（以及tcx及有关其来源的一些信息）。
因此，其是对MIR进行原地修改而非重新构造（这有助于保持效率）。

基本的MIR pass的一个很好的例子是[`NoLandingPads`]，它遍历MIR并删除由于unwinding而产生的所有边——当配置为`panic=abort`时，它unwind永远不会发生。
从源代码可以看到，MIR pass是通过首先定义虚拟类型（无字段的结构）来定义的，例如：

```rust
struct MyPass;
```

接下来您可以为它实现`MirPass`特征。
然后，您可以将此pass插入到在诸如`optimized_mir`，`mir_validated`等查询的pass的适当列表中。
（如果这是一种优化，则应进入`optimized_mir`列表中。）

如果您要写pass，则很有可能要使用[MIR访问者]。
MIR访问者是一种方便的方法，可以遍历MIR的所有部分，以进行搜索或进行少量编辑。

### 窃取

中间查询`mir_const()`和`mir_validated()`产生一个使用`tcx.alloc_steal_mir()`分配的`&tcx Steal<Mir<'tcx >>`。
这表明结果可能会被下一组优化**窃取** —— 这是一种避免克隆MIR的优化。
尝试使用被窃取的结果会导致编译器panic。
因此，重要的是，除了作为MIR处理管道的一部分之外，不要直接从这些中间查询中读取信息。

由于存在这种窃取机制，因此还必须注意确保在处理管道中特定阶段的MIR被窃取之前，任何想要读取的信息都已经被读取完了。
具体来说，这意味着如果您有一些查询`foo(D)`要访问`mir_const(D)`或`mir_validated(D)`的结果，
则需要让后继使用`ty::queries::foo::force(...)`强制传递`foo(D)`。
即使您不直接要求查询结果，这也会强制执行查询。

例如，考虑MIR const限定。
它想读取由`mir_const()`套件产生的结果。
但是，该结果将被`mir_validated()`套件**窃取**。
如果什么都不做，那么如果`mir_const_qualif(D)`在`mir_validated(D)`之前执行，它将成功执行，否则就会失败。
因此，`mir_validated(D)`会在实际窃取之前对`mir_const_qualif`进行强制执行，
从而确保读取已发生（请记住[查询已被记忆](../query.html)，因此执行第二次查询时只是从缓存加载）：

```text
mir_const(D) --read-by--> mir_const_qualif(D)
     |                       ^
  stolen-by                  |
     |                    (forces)
     v                       |
mir_validated(D) ------------+
```

这种机制有点偷奸耍滑的感觉。 [rust-lang/rust#41710] 中讨论了更优雅的替代方法。

[rust-lang/rust#41710]: https://github.com/rust-lang/rust/issues/41710
[mirtransform]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/
[`NoLandingPads`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/no_landing_pads/struct.NoLandingPads.html
[MIR访问者]: ./visitor.html
