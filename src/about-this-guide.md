# 关于本指南

本指南旨在帮助记录rustc——Rust编译器的工作方式，并帮助新的参与者参与rustc开发。

本指南分为六个部分：

[构建和调试和`rustc`][p1]：无论您准备如何作出贡献，这部分信息都应该都很有用，例如如何构建、调试、分析编译器等。
[向`rustc`作出贡献][p1-5]：无论您准备如何作出贡献，这部分信息都应该都很有用，例如作出贡献的一般过程，功能的稳定性等等。
2. 编译器的高层架构：讨论编译器的高级架构和编译过程的各个阶段。
3. 源码表示：描述了从用户那里获取源代码并将其转换为编译器可以轻松使用的各种形式的过程。
4. 分析：讨论编译器用来检查代码的各种属性并通知编译过程的后期阶段（例如，类型检查）的分析过程。
5. 从MIR到Binaries：如何生成链接好的可执行机器代码。
6. 附录：其中有大量实用的参考信息，包括词汇表。

[p1]: ./getting-started.md
[p1-5]: ./compiler-team.md
[p2]: ./part-2-intro.md
[p3]: ./part-3-intro.md
[p4]: ./part-4-intro.md
[p5]: ./part-5-intro.md
[app]: ./appendix/background.md

该指南本身当然也是开源的，可以在[GitHub repository]中找到本书的源码。如果您在本书中发现任何错误，请开一个issue，或更好的，开一个带有更正内容的PR！

## 其他信息

以下站点也可能对您来说有用：

- [rustc API docs] -- 编译器的rustdoc文档
- [Forge] -- 包含rust基础设施、团队工作流程、以及更多
- [compiler-team] -- rust编译器团队的基地，其中包含对团队工作流程，活动中的工作组和团队日历的描述。

[GitHub repository]: https://github.com/rust-lang/rustc-dev-guide/
[rustc API docs]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_middle/
[Forge]: https://forge.rust-lang.org/
[compiler-team]: https://github.com/rust-lang/compiler-team/
