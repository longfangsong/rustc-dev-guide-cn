# MIR 访问者

The MIR visitor is a convenient tool for traversing the MIR and either
looking for things or making changes to it. The visitor traits are
defined in [the `rustc_middle::mir::visit` module][m-v] – there are two of
them, generated via a single macro: `Visitor` (which operates on a
`&Mir` and gives back shared references) and `MutVisitor` (which
operates on a `&mut Mir` and gives back mutable references).

[m-v]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/visit/index.html

要实现访问者，您必须创建一个代表这个访问者的类型。 通常，此类型希望在处理MIR时“挂起”到您需要的任何状态：

```rust,ignore
struct MyVisitor<...> {
    tcx: TyCtxt<'tcx>,
    ...
}
```

然后您可以为那个类型实现 `Visitor` 或 `MutVisitor` trait：

```rust,ignore
impl<'tcx> MutVisitor<'tcx> for NoLandingPads {
    fn visit_foo(&mut self, ...) {
        ...
        self.super_foo(...);
    }
}
```

如上所示，在impl中，您可以覆盖任何`visit_foo`方法（例如，`visit_terminator`），以便编写一些在看到`foo`时执行的代码。
如果您想递归遍历`foo`的内容，则可以调用`super_foo`方法。
（注意。您永远都不应该覆盖`super_foo`）

可以在[`NoLandingPads`]中找到一个非常简单的访问者示例。该访问者甚至不需要任何状态：它仅访问所有终止符并删除其`unwind`后继。

[`NoLandingPads`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_mir/transform/no_landing_pads/struct.NoLandingPads.html

## 遍历

In addition the visitor, [the `rustc_middle::mir::traversal` module][t]
contains useful functions for walking the MIR CFG in
[different standard orders][traversal] (e.g. pre-order, reverse
post-order, and so forth).

除了访问者之外， [`rustc::mir::traversal`模块][t]也包含了用于按[不同的标准顺序][traversal]（例如，先根序、后根序等等）便利MIR CFG的实用函数。

[t]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/mir/traversal/index.html
[traversal]: https://en.wikipedia.org/wiki/Tree_traversal

