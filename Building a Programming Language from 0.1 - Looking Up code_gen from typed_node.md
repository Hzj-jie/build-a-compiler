# Building a Programming Language from 0.1 - Looking Up code_gen from typed_node

A very important property of `typed_node` is the integer and string representations of its type. For example, the `type_name` of a parsed `function` node is `function`. Therefore, the simplest approach is to select the corresponding code generator using `type_name`. First, we need to define the interface for code generation:

```text
Public Interface code_gen(Of WRITER)
    Function build(ByVal n As typed_node, ByVal o As WRITER) As Boolean
End Interface

```

Here, `WRITER` is implementation-dependent. Although it is generally just string concatenation, the allowed parameter types vary depending on the use case.

`code_gens` is a collection of `code_gen` instances that uses a node's `type_name` to select the corresponding `code_gen` to generate code.

```text
Public Class code_gens(Of WRITER)
    Private ReadOnly m As map(Of String, code_gen(Of WRITER))

    Public Sub New()
        m = New map(Of String, code_gen(Of WRITER))()
    End Sub

    Public Shared Function code_gen_name(Of T As code_gen(Of WRITER))() As String
        Return GetType(T).Name().Replace("_"c, "-"c)
    End Function

    Public Sub register(ByVal s As String, ByVal b As code_gen(Of WRITER))
        assert(Not s.null_or_whitespace())
        assert(Not b Is Nothing)
        m.emplace(s, b)
    End Sub
...
End Sub

```

[service/compiler/code_gen/code_gen.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen.vb)

However, code generation is not simply a direct line-by-line translation. For example, the `logic` language has no concept of a `main` function; `logic` merely translates code into a sequence of `primitive` instructions, so its entry point can be understood as the first statement that is not jumped over (`jmp`). In contrast, the entry point of the bstyle language is a specific function, because its root node only contains variable or function definitions. To handle this, the concept of `statements` is introduced. `statements` represents a fixed block of code that can be prepended or appended to the generated code, consisting essentially of constants. Thus, we can implement the entry point of the bstyle language simply by appending:

```text
caller main ( )

```

to the end of the generated code.

[service/compiler/code_gen/statements.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/statements.vb)

By combining `code_gens`, `statements`, and the rule files for nlexer and syntaxer, we can unify source code analysis and target code generation. This combination is called `code_gen_rule_wrapper`.

[service/compiler/code_gen/code_gen_rule_wrapper.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb)

As mentioned above, `statements` has two instances: `prefixes` and `suffixes`, one prepended to the code and the other appended to the end. Therefore, the core logic resides in the `build` function of `code_gen_rule_wrapper`.

```text
Public Function build(ByVal root As typed_node, ByVal o As WRITER) As Boolean
    If root.leaf() Then
        Return False
    End If
    p.export(o)
    assert(root.child_count() > 0)
    Dim i As UInt32 = 0
    While i < root.child_count()
        If Not l.of(root.child(i)).build(o) Then
            Return False
        End If
        i += uint32_1
    End While
    s.export(o)
    Return True
End Function

```

A common `code_gen` implementation forwards child nodes to another `code_gen`. For instance, `multi-sentence-paragraph` forwards all of its child nodes except `start-paragraph` and `end-paragraph` to the `code_gen` of `sentence`.

There are two ways to forward nodes. One is type-agnostic, where the generator does not care about the specific type of the target generator it is forwarding to. Forwarding from `multi-sentence-paragraph` to `sentence` falls into this category, because the syntaxer rules guarantee type correctness—its second through second-to-last child nodes can only be `sentence`.

The other is type-aware, where the processing logic of the current node needs to invoke specific functions on the target generator. This is used on nodes such as `value` and `paramlist`, and we will discuss its role in detail when we cover their concrete implementations.
