# Building a Programming Language from 0.1 - int i; and int i=1;

Although the syntax of bstyle mimics C, most programming languages do not share some of C's characteristics. For example, most languages can properly handle statements like:

```text
int i
int j = 1

Dim i As Int32
Dim j As Int32 = 1

```

without requiring a semicolon at the end of each statement. However, these are clearly two different types of statements: the former is usually called a declaration, while the latter is usually called a definition. Therefore, just like the implementation of nlexer, syntaxer also follows two rules: longest/greedy matching and priority matching.

Thus, for these two rules:

```text
value-definition name name assignment value
value-declaration name name 

```

regardless of their order, they will always correctly match the corresponding rule.
