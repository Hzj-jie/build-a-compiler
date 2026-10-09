# Building a Programming Language - Basic Architecture of b3style

In the previous article, [Zijie He: Building a Programming Language from 0.1 - If Linked Lists Are Needed](https://zhuanlan.zhihu.com/p/524962667), we mentioned that b2style cannot support member template functions in classes because b2style lacks parameter type information for function calls when expanding templates:

```cpp
// This should match template <T> int f(int a, string b);
// However, b2style does not know that `1 + 1` is an integer, or that `"abc"` is a string.
int x = f<int>(1 + 1, "abc");
```

In b2style's implementation, all of this relies entirely on bstyle. Of course, we could port bstyle's type-tracking functionality into b2style as well—for instance, [scope.call_hierarchy](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope.call_hierarchy.vb) already exists in both bstyle and b2style with slight variations. However, continuing down that path would make b2style a superset of bstyle not just in syntax, but in implementation as well. Compiling b2style into bstyle statements and then compiling those into the logic language would become redundant and inefficient. Instead, it is better to implement b3style:

- In terms of lexis and syntax, it will be identical to b2style for now. Additional features may be added later, though that is not an immediate requirement.
- In terms of implementation, it takes C++-like source code as input and directly outputs logic language code.
- Certain parts of the implementation may be optimized—for example, allowing `#include` to search multiple directories, or performing `pack` and `unpack` when passing structs as parameters to reduce complexity—though these are not immediate requirements either.
- It will be fully compatible with both b2style and bstyle, capable of compiling code written in either language directly.

Thus, b3style can be described as a bstyle compiler augmented with b2style-specific lexical rules, syntax, and features. Since it "is a" bstyle compiler, at first glance it might seem that b3style should inherit from bstyle, though in practice we will not implement it that way.

Let's review the existing implementations of bstyle and b2style, which are essentially divided into three parts:

- Lexis and syntax: these two parts are simple and only require rule files to drive `nlexer` and `syntaxer`.
- Stateless code generators: roughly one code generator per `syntaxer` rule, producing the corresponding logic language or bstyle language code for that rule.
- `scope`: used to manage compiler state, such as storing types, variables, templates, and so on.

In fact, even the logic language implementation follows the same structure, except that the logic language does not use the `nlexer` and `syntaxer` automata to match its input.

### Lexis and Syntax

Nothing needs to be done here; we can use b2style's rules directly.

### Code Generators

Conceptually, this can be thought of as bstyle's code generators combined with b2style's code generators chained into bstyle's code generators. For example, `function` uses bstyle's code generator, while `class` could chain the output of b2style's code generator into bstyle's `struct` and `function` code generators. However, chaining them like that is somewhat awkward—for instance, `variable` code is not recognized by the grammar as a `ROOT_TYPE` and therefore cannot be compiled directly. Moreover, naive chaining is inefficient, so the vast majority of b2style's code generators will still need to be reimplemented to emit logic language code directly.

### scope

This is roughly equal to `bstyle.scope` plus `b2style.scope`; the functionality of both is needed, aside from a small amount of overlap.

### Test-Driven Development

This is a key departure from bstyle and b2style: whereas both of those were feature-driven, the implementation of b3style will be test-driven. Simply put, we will first implement an NLP (lexer and parser) compatible with b2style's syntax, then a compiler compatible with bstyle's syntax, then a compiler compatible with b2style's syntax, and only after that consider adding new features. All three stages will be driven by tests, with the baseline requirement that b3style can run the existing test suites and produce the exact same sequence of primitive instructions—or at least produce the same runtime results.

Producing an identical instruction sequence is a strong requirement, demanding that the implementation match completely (such as dead-code elimination logic); producing identical runtime results is a weaker requirement that allows room for certain optimizations or trade-offs.
