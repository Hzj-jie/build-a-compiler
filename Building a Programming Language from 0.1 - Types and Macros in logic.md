# Building a Programming Language from 0.1 - Types and Macros in logic

The `logic` language supports width-based types, allowing a high-level language to implement a variety of types—such as the common `int`, `bool`, `byte`, and `string`, as well as the less common `big_uint`. At the same time, the high-level language no longer needs to track the type of every variable or function return value at compile time. For example:

```text
int f() { ... }
int i = f();

```

will be compiled into:

```text
callee int { ... }
define i int
define raw_value_unique_name {return-type-of-f}
caller raw_value_unique_name f ( )
move i raw_value_unique_name

```

In the fourth step, the `logic` language automatically verifies whether the types of `raw_value_{unique_name}` and `i` match. To avoid duplicating type bookkeeping in both the high-level language layer and the `logic` language layer, `logic` implements a set of macros.

As a compile-time structure in the `logic` language, `types` is responsible for recording all types defined via the `type` command. Simply put, it is a dictionary mapping type name strings to their byte widths.

Beyond fixed-width types, the `logic` language supports two special built-in types called `type0` and `type*`. `type0` is a zero-length type, which serves a role similar to `void` in C. `type*` is a variable-length type, which can be used to implement strings, arrays, or `big_uint`. These two types follow specific assignment rules: `type0` cannot be copied to any type, nor can it receive a value of any type. Conversely, `type*` can receive a value of any type, but can only be copied to another `type*`. Defining custom types with a length of 0 or infinite length is neither allowed nor necessary.

Code for `types`: [service/compiler/logic/types.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/types.vb)

Meanwhile, `macros` implements macros in the `logic` language. Put simply, it rewrites `logic` source code at compile time by querying `scope`, `anchors`, and `types`—for instance, rewriting `{return-type-of-f}` in the example above into `int`. Unlike traditional preprocessor macros that run before compilation, these macros must be evaluated during compilation: before `int f() { ... }` is compiled, the `logic` compiler has no way of knowing that `{return-type-of-f}` resolves to `int`.

`macros` also provides a simple mechanism for implementing function overloading in high-level languages. Depending on how the high-level language is defined, overloaded functions can be disambiguated by encoding parameter type names or parameter widths into the mangled function name. We will discuss this in more detail later.

Code for `macros`: [service/compiler/logic/macros.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/macros.vb)
