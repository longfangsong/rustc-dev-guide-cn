# 词法分析与语法分析

> 词法分析和语法分析器当前正在进行大量重构，因此本章的某些部分可能已过时。

编译器要做的第一件事就是将程序（一堆Unicode字符）转换为比字符串更方便编译器使用的表示形式。 这发生在两个阶段：词法分析和语法分析。

词法分析接收字符串并将其转换为token流。
例如，`a.b + c`将被转换为token `a`，`.`，`b`，`+`和`c`。 该词法分析器位于[`librustc_lexer`][lexer]中。

[lexer]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_lexer/index.html

然后，文法分析将获取到的token流并将其转换为一种通常称为[*抽象语法树*][ast]（AST）的结构化形式，便于编译器使用。
AST使用`Span`将特定的AST节点链接回其源文本，从而镜像内存中Rust程序的结构。

AST是在[`librustc_ast`][librustc_ast]中定义的，
其中包括token和token流的一些定义、用于修改AST的数据结构/trait、以及编译器其他与AST相关的部分（如词法分析器和宏展开）。

文法分析器在[`librustc_parse`][librustc_parse]中定义，这个crate中也包含了词法分析器的高级接口以及一些在宏扩展后运行的验证例程。
特别的，[`rustc_parse::parser`][parser]包含文法分析器实现。

文法分析器的主要入口点是通过 [parser][parser] 中的各种`parse_*`函数。
它们使您可以执行以下操作，例如将[`SourceFile`][sourcefile]（例如，单个文件中的源）转换为token流，
从token流创建文法分析器，然后执行文法分析器以获取`Crate`（ AST根节点）。

为了最大程度地减少复制的数量，`StringReader`和`Parser`都具有将其绑定到父`ParseSess`的生命周期。它包含文法分析时所需的所有信息以及`SourceMap`本身。

## 更多关于词法分析的信息

词法分析代码分为两个部分：

- `rustc_lexer` crate负责将`&str`分成组成token。
尽管普遍的做法是使用程序生成基于有限状态机的词法分析器，但`rustc_lexer`中的词法分析器是手写的。
- 来自[`librustc_ast`][librustc_ast] 的[`StringReader`]将`rustc_lexer`与`rustc`特定的数据结构集成在一起。
具体来说，它将`Span`信息添加到`rustc_lexer`返回的token和内部标识符中。

[librustc_ast]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/index.html
[rustc_errors]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_errors/index.html
[ast]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
[`SourceMap`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_span/source_map/struct.SourceMap.html
[ast module]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/ast/index.html
[librustc_parse]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/index.html
[parser]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/parser/index.html
[`Parser`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/parse/parser/struct.Parser.html
[`StringReader`]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_parse/lexer/struct.StringReader.html
[visit module]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_ast/visit/index.html
[sourcefile]: https://doc.rust-lang.org/nightly/nightly-rustc/rustc_span/struct.SourceFile.html
