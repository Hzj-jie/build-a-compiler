# Building a Programming Language from 0.1 - Conditionals and Loops

While conditionals and loops in the `bstyle` language are not quite as simple as the basic `code_gen` implementations mentioned earlier, they are not particularly complex either, so let's summarize them here.

`condition`:

```text
condition kw-if start-bracket value end-bracket paragraph [else-condition]?
else-condition kw-else paragraph 

```

While using `read_target` of `value`, if the `condition` node has 5 child nodes, there is no `else-condition`. Otherwise, we simply invoke the `code_gen` of `else-condition` as well. [service/compiler/bstyle/logic/nodes/condition.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/condition.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() >= 5)
    If Not l.of(n.child(2)).build(o) Then
        Return False
    End If
    Using read_target As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
        Dim satisfied_paragraph As Func(Of Boolean) = Nothing
        satisfied_paragraph = Function() As Boolean
                                  Return l.[of](n.child(4)).build(o)
                              End Function
        If n.child_count() = 5 Then
            Return builders.of_if(+read_target, satisfied_paragraph).to(o)
        End If
        Dim unsatisfied_paragraph As Func(Of Boolean) = Nothing
        unsatisfied_paragraph = Function() As Boolean
                                    Return l.[of](n.child(5)).build(o)
                                End Function
        Return builders.of_if(+read_target, satisfied_paragraph, unsatisfied_paragraph).to(o)
    End Using
End Function

```

`for-loop`:

```text
for-loop kw-for start-bracket [value-declaration, value-definition]? semi-colon [value]? semi-colon [value-clause]? end-bracket paragraph 

```

Because there are many possible combinations here, we use `child_named_map` of `typed_node` to generate a map keyed by each child node's type name. Beyond that, it uses `while_then` from the `logic` language. [service/compiler/bstyle/logic/nodes/for_loop.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/for_loop.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Dim ref As ref = Nothing
    ref = New ref(n)
    assert(Not o Is Nothing)
    If Not ref.declaration Is Nothing AndAlso Not l.of(ref.declaration).build(o) Then
        Return False
    End If
    If Not ref.definition Is Nothing AndAlso Not l.of(ref.definition).build(o) Then
        Return False
    End If
    If Not condition_value(ref, o) Then
        Return False
    End If
    Using read_target As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
        Return builders.of_while_then(+read_target,
                                      Function() As Boolean
                                          Return l.of(ref.paragraph).build(o) AndAlso
                                                 (ref.clause Is Nothing OrElse
                                                  l.of(ref.clause).build(o)) AndAlso
                                                 condition_value(ref, o)
                                      End Function).to(o)
    End Using
End Function

```

`while`:

```text
while kw-while start-bracket value end-bracket paragraph 

```

This is essentially a combination of `while_then` from the `logic` language and `value`. [service/compiler/bstyle/logic/nodes/while.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/while.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() = 5)
    If Not while_value(n, o) Then
        Return False
    End If
    Using value_target As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
        Return builders.of_while_then(+value_target,
                                      Function() As Boolean
                                          Return l.of(n.child(4)).build(o) AndAlso while_value(n, o)
                                      End Function).to(o)
    End Using
End Function

```
