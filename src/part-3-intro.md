# Part 3: 源代码的不同表示

这部分描述了从用户那里获取原始源代码并将其转换为编译器可以轻松使用的各种形式的过程。 这些称为中间表示。

This process starts with compiler understanding what the user has asked for:
parsing the command line arguments given and determining what it is to compile.
After that, the compiler transforms the user input into a series of IRs that
look progressively less like what the user wrote.
