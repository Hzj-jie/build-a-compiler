# Building a Programming Language from 0.1 - Supporting namespace

As mentioned earlier, `namespace` support in `b2style` is implemented through the `kw-namespace` keyword and modified `name` rules. The role of `kw-namespace` is to define the current `namespace`, while the `name` rule always rewrites function and variable names using the current `namespace`.

Very much like `function`, `kw-namespace` uses `write_scoped` to store and manage the current `namespace`, which also enables nested `namespace` support through `write_scoped`. Beyond that, what `kw-namespace` does during rewriting is very simple: it just forwards all of its child statements to their corresponding `rewriter`/`code_gen`. [service/compiler/b2style/rewriter/nodes/kw_namespace.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/kw_namespace.vb)

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    assert(n.child_count() >= 4)
    Using ws.push(format(n.child(1).word().str()))
        For i As UInt32 = 2 To n.child_count() - uint32_2
            If Not l.of(n.child(i)).build(o) Then
                Return False
            End If
        Next
    End Using
    Return True
End Function

```

`kw-namespace` provides two helper functions. The first is `format`, which determines the current `namespace`. For example:

```cpp
namespace a {
  namespace b {
    namespace c {
      int ia;
    }
namespace ::d {
  int ib;
}
    int ic;
  }
}

```

Here, the full `namespace` of `c` is `a::b::c`, while the full `namespace` of `d` is `d`. The same applies when referencing variables and functions: the full variable name of `ia` is `a::b::c::ia`, `ib` is `d::ib`, and `ic` is `a::b::ic`. Of course, this rule is not completely identical to `namespace` resolution in C++, but it is sufficient for `b2style`'s needs.

The second helper is `bstyle_format`, which converts a `namespace`-qualified variable or function name in `b2style` into a `bstyle` name. For example:

```text
::b2style::std_out -> bstyle__std_out
::b2style::true_str -> bstyle__true_str

```

The conversion rule is straightforward: replace colons with underscores. Because the characters allowed in `bstyle` variable or function names (the `name` rule) are a subset of those in `b2style`, `b2style` cannot use characters disallowed by the source language to avoid collisions the way `bstyle` does when compiling to the `logic` language. As a result, if you manually write a function named `namespace__function`, it can also be called as `namespace::function`. While not entirely ideal, it does not cause any serious problems.

In the `name` rule, `bstyle_format` from `kw-namespace` is used to convert a `namespace`-qualified variable or function name into a form supported by the `bstyle` language. [service/compiler/b2style/rewriter/nodes/name.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/name.vb)

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    assert(n.leaf())
    o.append(code_gen_of(Of kw_namespace)().bstyle_format(n.word().str()))
    Return True
End Function

```
