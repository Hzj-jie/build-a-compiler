# Building a Programming Language from 0.1 - #include <stdio.h>

In [The Basic Implementation Structure of the bstyle Language](https://zhuanlan.zhihu.com/p/104344721), it was mentioned that `bstyle` adds several prefixes and a suffix to the generated `logic` code. Prefixes are used to define types, constants, temporary variables, etc., while the suffix is used to generate the call to the `main` function. In [The Purpose and Basic Implementation Structure of the b2style Language](https://zhuanlan.zhihu.com/p/105230021), it was also mentioned that operators in `b2style` are implemented by converting operations into function calls, and these functions are provided by several prefixes of the `b2style` language, which are themselves implemented in the `bstyle` language.

Let's review the workflow of [codegen_rule_wrapper](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb) again:

- Use `nlexer` for lexical analysis to turn the input into a sequence of `typed_word`s
- Use `syntaxer` for syntactic analysis to turn the sequence of `typed_word`s into a tree of `typed_node`s
- Sequentially find the `code_gen` corresponding to the type of each direct child node of the root node, and convert it into `WRITER`
- Run all prefixes and suffixes and append them to `WRITER`

The first two steps are completed by [nlp](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/nlp.vb), while prefixes are actually executed before running `nlexer` and `syntaxer`. This is purely a performance consideration and has no special significance. The key point is that prefixes and suffixes do not undergo lexical or syntactic analysis, nor are they converted through `code_gen`, so they are all implemented in the lower-level language. For example, [service/compiler/bstyle/logic/suffixes/main.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/suffixes/main.vb).

This is inherently inconvenient; the purpose of a higher-level language is to provide more convenient syntax and features. Moreover, it is impossible to endlessly add everything into prefixes and suffixes—after all, compilers at any level currently cannot strip out unused functions, so adding prefixes and suffixes increases the length of the final compiled `primitive` code. Providing an `include` feature has become a necessity. This `include` needs to:

1. Allow referencing code implemented in the language itself—for example, [stdio.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/lib/stdio.h) is implemented in the `bstyle` language rather than `logic` syntax.
2. Provide a way to prevent a piece of code from being included repeatedly—for example, if both `a.h` and `b.h` require `c.h`, [http://d.cc](http://d.cc) can include both `a.h` and `b.h` without causing `c.h` to be included twice.

Because the `bstyle` language itself imitates C, `ifndef` and `define` are also provided to prevent a piece of code from being included repeatedly.

In conventional implementations, `#ifndef`, `#define`, and `#include` belong to the preprocessing stage, where the compiler specially handles these symbols before compilation. However, doing so would require running `nlp` twice, and the rules used would differ. Of course, we could also implement a `b3style` language to specifically handle this issue, but that would be overly complex and unable to upgrade both `bstyle` and `b2style` at the same time. Therefore, the requirements are:

1. Do not run `nlp` twice
2. `include` and its rules reside in the same `nlexer` and `syntaxer` rules as other syntax rules
3. Usable in both `bstyle` and `b2style`

Under this design, the handling of `include` itself is also a kind of `code_gen`, except that this `code_gen` must invoke the compiler itself to compile the included code. Here, the compiler only includes the `nlp` and `code_gen` parts, excluding prefixes and suffixes. Prefixes and suffixes are only ever processed once.

The builder in `code_gen_rule_wrapper` is split into two parts: [builder](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb#L93) and [code_builder](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb#L37). The former executes prefixes and suffixes and calls the latter to compile, while the latter calls `nlp` and `code_gen` for code transformation.

```vb.net
' class builder
Public Function build(ByVal input As String, ByVal o As WRITER) As Boolean
    assert(Not o Is Nothing)
    Dim scope As SCOPE_T = alloc(Of SCOPE_T)()
    p.export(o)
    If Not cb.build(input, o) Then
        Return False
    End If
    s.export(o)
    scope.end_scope()
    Return True
End Function

```

When `include` executes, aside from performing necessary file operations, the rest is simply [calling the current `code_builder` to compile the included content](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/includes.vb#L36).

```text
Public Overrides Function at(ByRef i As String, ByRef j As writer) As Boolean
    Return code_builder.current().build(i, j)
End Function

```

What remains are engineering issues: how to call the current `code_builder` and how to reduce duplicate code between `bstyle` and `b2style`.

The current `code_builder` is [maintained by a thread-static variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb#L75). There are certainly better ways, but they would require too many changes, and compilation performance is not a particularly critical concern.

```text
Public Shared Function current() As code_builder
    Return thread_static_implementation_of(Of builder).resolve().cb
End Function

```

Similar to `code_gen_rule_wrapper`, a shared [includes](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/includes.vb) implementation provides the basic functionality, controlled by a few function template parameters.

For instance, `bstyle`'s own [includes](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/includes.vb) uses [zipgen](https://github.com/Hzj-jie/osi.net/tree/master/service/compiler/bstyle/lib) to store `.h` files inside the code, [extracting them to a temporary folder at runtime](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/includes.vb#L23), while also allowing [command-line arguments to control whether additional folders should be searched](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/includes.vb#L17).

```text
Private Shared include_folders As argument(Of vector(Of String))
Private Shared ignore_default_include As argument(Of Boolean)

Public NotInheritable Class default_includes
    Private Shared ReadOnly folder As String = Path.Combine(temp_folder, "bstyle-inc")

    Shared Sub New()
        assert(Not Directory.CreateDirectory(folder) Is Nothing)
        assert(bstyle_lib.stdio_h.sync_export(Path.Combine(folder, "stdio.h"), True))
        assert(bstyle_lib.cstdio.sync_export(Path.Combine(folder, "cstdio"), True))
        assert(bstyle_lib.bstyle_h.sync_export(Path.Combine(folder, "bstyle.h"), True))
        assert(bstyle_lib.bstyle_types_h.sync_export(Path.Combine(folder, "bstyle_types.h"), True))
        assert(bstyle_lib.bstyle_constants_h.sync_export(Path.Combine(folder, "bstyle_constants.h"), True))
    End Sub

```

However, going to the extreme of moving all built-in functions into `.h` files also creates some problems. For example, `b2style::equal` mentioned earlier is required by the `b2style` language; otherwise, `a == b` cannot be compiled. Previously, this was in `b2style`'s prefixes, and now it has been moved to [b2style_operators.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style_operators.h). If this file is not included, `a == b` will fail to compile. There are other similar issues later on, which we will discuss when the time comes. This is somewhat similar to `std::initializer_list` in C++11: how tightly coupled should a language be to the standard library it provides, and can the language itself still work without its standard library? Of course, the situation with `a == b` and `b2style::equal` is slightly better: users can provide their own implementation of `b2style::equal`, which can be considered a feature.

Another point is that some types themselves are mandatory, such as `string`, because the `bstyle` compiler converts string literals it reads into temporary variables of type `string` in `logic`. Therefore, these types remain in `bstyle`'s prefix [code_types](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/prefixes/code_types.vb).

Finally, there is a small implementation issue. When `bstyle` compiles into `logic` code, it needs to generate temporary variables. To simplify the implementation, these temporary variables [are named after their position in the code](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/logic_name.vb).

```text
Partial Public NotInheritable Class bstyle
    Public NotInheritable Class logic_name
        Public Shared Function temp_variable(ByVal n As typed_node) As String
            assert(Not n Is Nothing)
            Return strcat("temp_value_@",
                          code_builder.current().nested_build_level(),
                          "@",
                          n.word_start(),
                          "-",
                          n.word_end())
        End Function

```

This way, when processing `value` nodes, we can [avoid redefining a temporary variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value.target.vb#L51).

```text
Private Sub define_single_data_slot_temp_target(ByVal type As String,
                                                ByVal name As String,
                                                ByVal o As writer)
    assert(Not o Is Nothing)
    type = scope.current().type_alias()(type)
    assert(Not scope.current().structs().defined(type))
    Dim existing_type As String = Nothing
    If scope.current().variables().try_resolve(name, existing_type) Then
        assert(type.Equals(existing_type))
    Else
        assert(scope.current().variables().define(type, name))
        assert(builders.of_define(name, type).to(o))
    End If
End Sub

```

However, whenever `code_builder` compiles a file, there is no guarantee that these temporary variable names won't collide across files. So a simple solution is to include the current compilation depth—that is, [the recursion depth of the current `code_builder` call](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb#L33)—in the temporary variable name.

```text
' class code_builder
Public Function nested_build_level() As UInt32
    Return nested
End Function

Public Function build(ByVal input As String, ByVal o As WRITER) As Boolean
    nested += uint32_1
    Using defer.to(Sub()
                       nested -= uint32_1
                   End Sub)

```

In this way, any temporary variable name is guaranteed to be unique within a compilation unit.

As for `#ifndef` and `#define`, this simply uses an `unordered_set<string>` to record whether each string has already been defined. This `unordered_set<string>` is defined in [scope.define](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.define.vb), and the specific implementation of `scope` will be explained in detail in [Building a Programming Language from 0.1 - typedef](https://zhuanlan.zhihu.com/p/429373821).
