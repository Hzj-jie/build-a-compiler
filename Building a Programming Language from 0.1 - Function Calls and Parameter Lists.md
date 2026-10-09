# Building a Programming Language from 0.1 - Function Calls and Parameter Lists

Just like `value` expressions, function calls can also be nested, for example:

```cpp
func(func2(func3(value), func4(value2)));

```

The definition of `function-call` is:

```text
function-call name start-bracket [value-list]? end-bracket
value-list [value-with-comma]* value
value-with-comma value comma

```

Therefore, `read_scoped` can be used here as well.

When `value-list` processes each `value`, it places the resulting temporary variable name into a list, and finally pushes the entire list onto the stack. [service/compiler/bstyle/logic/nodes/value_list.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_list.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() > 0)
    Dim v As vector(Of String) = Nothing
    v = New vector(Of String)()
    Dim i As UInt32 = 0
    While i < n.child_count()
        If Not l.of(n.child(i)).build(o) Then
            Return False
        End If
        Using r As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
            v.emplace_back(+r)
        End Using
        i += uint32_1
    End While
    rs.push(v)
    Return True
End Function

```

After processing `value-list`, `function-call` reads the entire list from the stack to generate the `callee` instruction in the `logic` language. [service/compiler/bstyle/logic/nodes/function_call.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function_call.vb)

```vb.net
Private Function build(ByVal n As typed_node,
                       ByVal o As writer,
                       ByVal build_caller As Action(Of String, vector(Of String))) As Boolean
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(Not build_caller Is Nothing)
    assert(n.child_count() >= 3)
    Dim callee_name As String = Nothing
    callee_name = n.child(0).word().str()
    If n.child_count() = 3 Then
        build_caller(callee_name, vector.of(Of String)())
        Return True
    End If
    If Not l.of(n.child(2)).build(o) Then
        Return False
    End If
    Using targets As read_scoped(Of vector(Of String)).ref = code_gen_of(Of value_list)().current_targets()
        build_caller(callee_name, +targets)
        Return True
    End Using
End Function

```

There are two types of function calls: `ignore-result-function-call`, which ignores the return value, and a standard `function-call`, which uses the return value as the result of a `value`.

The former generates a `callee *` statement in the `logic` language:

```vb.net
Public Function without_return(ByVal n As typed_node, ByVal o As writer) As Boolean
    Return build(n,
                 o,
                 Sub(ByVal callee_name As String, ByVal parameters As vector(Of String))
                     function_name.of_caller(callee_name, parameters).to(o)
                 End Sub)
End Function

```

The latter generates a `callee return_value` statement in the `logic` language:

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() >= 3)
    Return build(n,
                 o,
                 Sub(ByVal callee_name As String, ByVal parameters As vector(Of String))
                     Dim name As String = Nothing
                     name = function_name.of_function_call(callee_name, parameters)
                     Dim value_name As String = Nothing
                     value_name = code_gen_of(Of value)().with_temp_target(macros.return_type_of(name), n, o)
                     builders.of_caller(name, value_name, parameters).to(o)
                 End Sub)
End Function

```
