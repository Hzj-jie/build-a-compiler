# Building a Programming Language from 0.1 - Function Definitions and return Statements

Recall that when implementing functions and `return` statements in the `logic` language, we took a bit of a shortcut: `return` statements always required providing the function name:

```text
callee copy-one-byte1 type0 ( source type* pos uint len uint ) {
    define char type*
    cut_slice char source pos len
    append target1 char
    return copy-one-byte1 *
}

```

That was a somewhat arbitrary decision, though since it is already implemented that way, there is no compelling reason to change it. Instead, in the `bstyle` language, we will implement a `return` statement that does not require specifying the function name.

Just like `read_scoped` mentioned earlier, we first implement `write_scoped`. In contrast to `read_scoped`, `write_scoped` pops from the stack when writing is complete. Specifically, it returns an `IDisposable` when pushing a value, leveraging .NET's `Using` statement to pop the stack safely. [service/compiler/logic/logic_gen/scoped/write_scoped.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/logic_gen/scoped/write_scoped.vb)

```vb.net
Public NotInheritable Class ref
    Implements IDisposable

    Private ReadOnly r As write_scoped(Of T)
    Private ReadOnly v As T

    Public Sub New(ByVal r As write_scoped(Of T))
        assert(Not r Is Nothing)
        assert(Not r.s.empty())
        Me.r = r
        Me.v = r.s.back()
    End Sub

    Public Sub Dispose() Implements IDisposable.Dispose
        r.s.pop()
    End Sub

    Public Shared Operator +(ByVal this As ref) As T
        assert(Not this Is Nothing)
        Return this.v
    End Operator
End Class

Public Function push(ByVal v As T) As ref
    s.emplace(v)
    Return New ref(Me)
End Function

Public Function current() As T
    assert(Not s.empty())
    Return s.back()
End Function

```

The implementation of `function` uses `write_scoped` to record the current function definition. Although `bstyle` does not yet support behaviors like nested functions at compile time—meaning only a single function definition needs to be tracked at any given moment—`write_scoped` simplifies many minor details and makes it easier to extend the `bstyle` language in the future, so we simply use `write_scoped`.

[service/compiler/bstyle/logic/nodes/function.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function.vb)

`function` implements a `ref` class to record the function currently being compiled:

```vb.net
Public NotInheritable Class ref
    Private ReadOnly ta As type_alias
    Private ReadOnly params As vector(Of pair(Of String, String))
    Private ReadOnly n As typed_node

    Public Sub New(ByVal ta As type_alias,
                   ByVal params As vector(Of pair(Of String, String)),
                   ByVal n As typed_node)
        assert(Not ta Is Nothing)
        assert(Not params Is Nothing)
        assert(Not n Is Nothing)
        assert(n.child_count() = 5 OrElse n.child_count() = 6)
        Me.ta = ta
        Me.params = params
        Me.n = n
    End Sub

    Public Function allow_return_value() As Boolean
        Return Not strsame(n.child(0).word().str(), types.void)
    End Function

    Public Function name() As String
        Return function_name.of_function(ta, n.child(1).word().str(), params)
    End Function
End Class

```

It also involves some implementation details of `type_alias` and `function_name`, which we will discuss later. Beyond that, `ref` records the name of the current function and provides helper methods such as checking whether a return value is allowed.

When compiling the body of a function, this definition is pushed onto the stack so that the function's details can be retrieved while compiling its body.

```vb.net
function_name.of_callee(ta,
                        n.child(1).word().str(),
                        n.child(0).word().str(),
                        +params,
                        Function() As Boolean
                            Using s.push(New ref(ta, +params, n))
                                Dim gi As UInt32 = 0
                                gi = CUInt(If(has_paramlist, 5, 4))
                                Return l.of(n.child(gi)).build(o)
                            End Using
                        End Function).to(o)

```

Usage in the `return` statement: [service/compiler/bstyle/logic/nodes/return_clause.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/return_clause.vb)

```vb.net
c = code_gen_of(Of [function])().target()
If c.allow_return_value() Then
    If n.child_count() <> 2 Then
        raise_error(error_type.user, "Expect return value ", n.child().trace_back_str())
        Return False
    End If
Else
    If n.child_count() <> 1 Then
        raise_error(error_type.user, "Unexpected return value ", n.child(1).trace_back_str())
        Return False
    End If
End If
If n.child_count() = 2 Then
    If Not l.of(n.child(1)).build(o) Then
        Return False
    End If
    Using r As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
        builders.of_return(c.name(), +r).to(o)
    End Using
Else
    builders.of_return(c.name()).to(o)
End If

```

Here, `params` is implemented by `paramlist`, and both `paramlist` and `param` also use `read_scoped` to record the current parameter list and parameter. Similarly, although neither `paramlist` nor `param` supports recursion, using `read_scoped` simplifies the implementation. [service/compiler/bstyle/logic/nodes/paramlist.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/paramlist.vb)
