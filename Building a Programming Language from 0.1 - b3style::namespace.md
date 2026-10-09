# Building a Programming Language from 0.1 - b3style::namespace

As a more complete implementation of the `b2style` language, `b3style` supports all of `b2style`'s syntax and should therefore be able to use `b2style`'s libraries. Naturally, namespaces are the first `b2style` syntax feature we need to implement beyond `bstyle` syntax.

The implementation itself is very simple and follows the same basic idea as `b2style.scope._namespace`: [service/compiler/b3style/logic/nodes/namespace.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/namespace.vb)

```vb.net
Partial Public NotInheritable Class b3style
    Private NotInheritable Class _namespace
        Implements code_gen(Of logic_writer)

        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(n.child_count() >= 4)
            Using scope.current().current_namespace().define(scope.current_namespace_t.of(n.child(1).word().str()))
                For i As UInt32 = 3 To n.child_count() - uint32_2
                    If Not code_gen_of(n.child(i)).build(o) Then
                        Return False
                    End If
                Next
            End Using
            Return True
        End Function
    End Class
End Class
```

Simply put, all statements inside a `namespace` block are evaluated within the corresponding `scope`, so whenever needed, `scope.current_namespace_t.of` can prepend the active namespace to a symbol's name.

However, unlike `b2style` which uses `_namespace.bstyle_format`, `b3style` translates statements directly into `logic` code. Because the namespace separator `::` is valid in `logic` syntax, `b3style` can use the full name directly—similar to the concept of a fully qualified name in other languages. For example, here is a snippet compiled by `b3style`:

```text
callee ::to_long:BigUnsignedInteger long ( x BigUnsignedInteger ) { start_scope { define y long
 cut_len y x @@prefixes@constants@int_0 @@prefixes@constants@size_of_long
 return ::to_long:BigUnsignedInteger y } }
```

With this in place, `b3style` can begin compiling small portions of `b2style` code, paving the way for implementing the remaining `b2style` statements.

The main changes are in [commit/d6a07421064dd373ad3584f87b77ff10f78d03b1](https://github.com/Hzj-jie/osi.net/commit/d6a07421064dd373ad3584f87b77ff10f78d03b1).
