# Building a Programming Language from 0.1 - Optimizing by Removing Unused Functions from Compilation Output

Up to this point, `b2style` compiles code into `bstyle` code, `bstyle` compiles code into `logic` code, and `logic` compiles code into a sequence of `primitive` instructions. Because the compiled instruction sequence always starts execution from the `main` function, some code may never be used, yet the compilation process has not taken this into account.

For example:

```cpp
#include <b2style.h>
void main() {
  b2style::std_out("abc");
}

```

This is a very simple program where `main` only uses the `b2style::std_out(string)` function from `b2style.h`. However, [b2style.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style.h) contains a large amount of code. Compiling all of that code into the final `primitive` instruction sequence does not cause a dramatic runtime performance issue, but the time wasted during compilation itself is considerable—after all, compiling any piece of code involves complex string matching and parsing.

Thus, a simple optimization is to strip out unused code. However, what to remove and at which compiler layer to remove it are interesting questions. Since we just mentioned unused functions, a natural starting point is removing functions that are never called. Although other things like unreachable dead code could also be removed and optimized, we will leave those aside for now.

```cpp
#include <b2style.h>
void main() {
  if (false) {
    b2style::std_out("abc");
  }
}
// This code could be optimized into an empty main function.

```

Next, let's consider which layer to remove them from. If we want the simplest implementation, the natural choice would be inside the `logic` compiler, which has no function overloading at all. However, doing it there means both the `b2style` and `bstyle` compilers would still generate large amounts of useless code, and both the `bstyle` and `logic` compilers would spend significant time compiling that unused code. On the other hand, `b2style` implements classes via `bstyle`'s function overloading, and the `b2style` compiler itself does not understand function overloading. Given this, implementing unused function removal in the `bstyle` compiler is a good middle ground.

Simply put, starting from any function, the code that needs to be executed forms a tree structure:

- Because `logic`, `bstyle`, and `b2style` have no linking stage, if `func_a` calls `func_b`, `func_b` must be defined before `func_a`, which makes it impossible for `func_b` to call `func_a`. Consequently, the call relationship is a DAG/tree rather than a general cyclic graph.

![image](https://pic1.zhimg.com/v2-2ab3c7a6e124fb3cef94f0caa36af624_r.jpg?source=d16d100b)
Credit: https://stackoverflow.com/questions/12450608/tool-to-create-call-hierarchy-of-library

To determine whether a function is needed in the final `primitive` instruction sequence, we only need to check whether it is reachable in the call tree rooted at `main`. Therefore, we first implement a [scope.call_hierarchy](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope.call_hierarchy.vb) class to record the function call relationships encountered during compilation, which is essentially a dictionary mapping a string (callee) to a list of strings (callers).

```vb.net
Partial Public Class scope(Of T As scope(Of T))
    Public MustInherit Class call_hierarchy
        Private ReadOnly main_name As String
        ' To -> From
        Private ReadOnly m As New unordered_map(Of String, vector(Of String))()
        Private ReadOnly tm As New unordered_map(Of String, Boolean)()
' ...
        Public Sub [to](ByVal name As String)
            assert(Not name.null_or_whitespace())
            Dim from As String = current_function_name().or_else(main_name)
            assert(Not from.null_or_whitespace())
            name = name.Trim()
            from = from.Trim()
            If Not name.Equals(from) Then
                m(name).emplace_back(from)
            End If
        End Sub

```

When `bstyle` generates code for a `function`, instead of writing it directly to the output, it writes it into an intermediate wrapper type. If at the end of compilation this function cannot be reached in `call_hierarchy`, it is simply omitted.

```vb.net
 Private NotInheritable Class filtered_writer
    Private ReadOnly ch As call_hierarchy
    Private ReadOnly f As String
    Private ReadOnly o As Func(Of String)

    Public Sub New(ByVal ch As call_hierarchy, ByVal f As String, ByVal o As Func(Of String))
        assert(Not ch Is Nothing)
        assert(Not f.null_or_whitespace())
        assert(Not o Is Nothing)
        Me.ch = ch
        Me.f = f.Trim()
        Me.o = o
    End Sub

    Public Overrides Function ToString() As String
        If (scope_arguments.remove_unused_functions Or True) AndAlso Not ch(f) Then
            Return empty_string
        End If
        Return o()
    End Function
End Class

```

Note that there is a flag here, `--remove-unused-functions`, which can disable this feature. This is useful—for example, when generating `.o` or `.so` files, this feature needs to be turned off, which we will discuss later.

`call_hierarchy` provides a method to check whether a function can be reached from the `main` function. The `string -> bool` dictionary above caches the query results.

```vb.net
 Default Public ReadOnly Property can_reach_root(ByVal f As String) As Boolean
    Get
        assert(Not f.null_or_whitespace())
        f = f.Trim()
        If f.Equals(main_name) Then
            Return True
        End If
        Dim it As unordered_map(Of String, Boolean).iterator = tm.find(f)
        If it <> tm.end() Then
            Return (+it).second
        End If
        Dim r As Boolean = False
        Dim it2 As unordered_map(Of String, vector(Of String)).iterator = m.find(f)
        If it2 <> m.end() Then
            Dim v As vector(Of String) = (+it2).second
            assert(Not v.null_or_empty())
            For i As UInt32 = 0 To v.size() - uint32_1
                If can_reach_root(v(i)) Then
                    r = True
                    Exit For
                End If
            Next
        End If
        tm(f) = r
        Return r
    End Get
End Property

```

In `bstyle`, the function name here is always the `logic` language function name—that is, the mangled function name augmented with parameter types to support overloading.

```vb.net
Function(ByVal parameters As vector(Of String)) As Boolean
    Dim name As String = Nothing
    If Not logic_name.of_function_call(n.child(0).word().str(), parameters, name) Then
        Return False
    End If
    scope.current().call_hierarchy().to(name)
    Return build_caller(name, parameters)
End Function

```

[service/compiler/bstyle/logic/nodes/function_call.vb#L40](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function_call.vb#L40)

```vb.net
o.append(scope.current().call_hierarchy().filter(
            logic_name.of_function(n.child(1).word().str(), +params),
            AddressOf fo.dump))

```

[service/compiler/bstyle/logic/nodes/function.vb#L54](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function.vb#L54)

The changes above are mainly in [commit/7d2d662e797f0719389370f283b338701f22f4e1](https://github.com/Hzj-jie/osi.net/commit/7d2d662e797f0719389370f283b338701f22f4e1).

At the same time, a relatively simple but less precise version of the same feature can also be implemented in the `b2style` compiler. It is "relatively simple but less precise" because `b2style` cannot resolve overloads, so all overloaded functions sharing a called name are retained; it is "similar" because the implementation is essentially identical to `bstyle`'s.

[Implement remove-unused-functions in b2style · Hzj-jie/osi.net@b74aa04](https://github.com/Hzj-jie/osi.net/commit/b74aa045aeab3a3dae49ef554ed5c8eb7369de50)

Testing this feature is a bit tricky and can only be done by inspecting the generated intermediate code:

```vb.net
<test>
Private Shared Sub unused_functions_should_be_removed()
    Dim bstyle_str As String = Nothing
    assertion.is_true(b2style.parse(_b2style_test_data.case1.as_text(), bstyle_str))
    ' b2style does not handle overload itself, so b2style::std_out(ufloat) and b2style::ufloat_to_str are included
    ' in the output.
    assertions.of(bstyle_str).not_contains("b2style__ufloat__from")
    Dim logic_str As String = Nothing
    assertion.is_true(bstyle.parse(bstyle_str, logic_str))
    assertions.of(logic_str).not_contains("b2style__ufloat")
End Sub

```

[service/tests/compiler/b2style/b2style_test.vb#L564](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/b2style_test.vb#L564)

Now let's look at the results:

```text
C:\Users\Hzj_jie\Documents\Visual Studio 2015\Projects\gemini\osi\root\utt\bin\Release>osi.root.utt b2stylec_main --input=c:\temp\c.cc --output=c:\temp\c.primitive > nul & dir c:\temp\c.primitive
 Volume in drive C has no label.
 Volume Serial Number is F611-2C42

 Directory of c:\temp

12/13/2021  01:26 AM             1,917 c.primitive
               1 File(s)          1,917 bytes
               0 Dir(s)  33,429,184,512 bytes free

C:\Users\Hzj_jie\Documents\Visual Studio 2015\Projects\gemini\osi\root\utt\bin\Release>osi.root.utt b2stylec_main --input=c:\temp\c.cc --output=c:\temp\c.primitive --remove-unused-functions=false > nul & dir c:\temp\c.primitive
 Volume in drive C has no label.
 Volume Serial Number is F611-2C42

 Directory of c:\temp

12/13/2021  01:27 AM            23,973 c.primitive
               1 File(s)         23,973 bytes
               0 Dir(s)  33,427,841,024 bytes free

```

The effect is quite significant.
