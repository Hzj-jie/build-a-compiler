# Building a Programming Language from 0.1 - More Accurate Function Call Detection

In [Zijie He: Building a Programming Language from 0.1 - An Optimization: Removing Unused Functions from Compilation Output](https://zhuanlan.zhihu.com/p/443777931), we mentioned that both the bstyle and b2style compilers can remove unreferenced functions from the generated code to reduce code size and compilation time.

The implementation at the time was overly simplistic and failed to account for certain edge cases, resulting in slow execution and even stack overflows under specific circumstances. Take this example, [service/tests/compiler/b2style/self-test/call-func-in-param.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/call-func-in-param.txt) (admittedly not a great filename):

```cpp
#include <b2style.h>
#include <testing.h>

void f(int x) {}

void g() {
  f(1);
}

void f() {
  g();
}

void main() {
  f();
  b2style::testing::finished();
}
```

It looks like a very simple piece of code, but since b2style does not resolve function overloading, processing the function `f` creates an infinite cycle `f -> g -> f`, which leads to a stack overflow. The previous implementation used a recursive depth-first search:

```cpp
assert(Not f.null_or_whitespace())
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
```

A straightforward improvement is to convert it to a non-recursive implementation. Whether we use depth-first or breadth-first search should have little impact on speed, though breadth-first search is a bit more intuitive:

```vb.net
+ Private Function _can_reach_root(ByVal f As String) As Boolean
+     Dim q As New queue(Of String)()
+     Dim v As New unordered_set(Of String)()
+     q.emplace(f)
+     assert(v.emplace(f).second())
+     While q.pop(f)
+         Dim it As unordered_map(Of String, vector(Of String)).iterator = m.find(f)
+         If it = m.end() Then
+             Continue While
+         End If
+         Dim fs As vector(Of String) = (+it).second
+         assert(Not fs.null_or_empty())
+         For i As UInt32 = 0 To fs.size() - uint32_1
+             Dim r As Boolean = True
+             If fs(i).Equals(main_name) OrElse
+                (tm.find(fs(i), r) AndAlso r) Then
+                 assert(tm.emplace(f, True).second())
+                 Return True
+             End If
+             If Not r Then
+                 Continue For
+             End If
+             q.emplace(fs(i))
+         Next
+     End While
+     Return False
+ End Function

  Default Public ReadOnly Property can_reach_root(ByVal f As String) As Boolean
      Get
          assert(Not f.null_or_whitespace())
          If f.Equals(main_name) Then
              Return True
          End If
          Dim it As unordered_map(Of String, Boolean).iterator = tm.find(f)
          If it <> tm.end() Then
              Return (+it).second
          End If
-         Dim r As Boolean = False
-         Dim it2 As unordered_map(Of String, vector(Of String)).iterator = m.find(f)
-         If it2 <> m.end() Then
-             Dim v As vector(Of String) = (+it2).second
-             assert(Not v.null_or_empty())
-             For i As UInt32 = 0 To v.size() - uint32_1
-                 If can_reach_root(v(i)) Then
-                     r = True
-                     Exit For
-                 End If
-             Next
-         End If
-         tm(f) = r
-         Return r
+         Return _can_reach_root(f)
      End Get
  End Property
```

The caching optimization is still kept to store search results, while an `unordered_set` is used to track whether a function name (or node) has already been visited. Because this is a breadth-first search, unfortunately it cannot mark all nodes along the path at once.

The main changes are in [Implement a better scope.call_hierarchy.can_reach_root to avoid faili… · Hzj-jie/osi.net@cb4f186](https://github.com/Hzj-jie/osi.net/commit/cb4f186756fc66f9ba93a9e30281b04e5dd2d76c).

## Reverse Search

Of course, the optimal approach is to search in reverse—starting from `main` rather than searching toward `main`. However, doing so requires adding a dedicated step during compilation to inspect the list of functions directly or indirectly referenced by `main` after code generation. This does not completely align with how the compiler is currently structured, though we can perform this calculation by adding a suffix. The purpose of a suffix is to append content after the compiled code; since this operation runs between compilation and emitting the final compilation output, it fits our needs nicely. Of course, that is just an implementation detail.

The implementation is still a breadth-first search, only starting from the `main` function:

```vb.net
        Private Sub calculate()
            assert(tm Is Nothing)
            tm = New unordered_set(Of String)()
            Dim q As New queue(Of String)()
            q.emplace(main_name)
            assert(tm.emplace(main_name).second())
            Dim f As String = Nothing
            While q.pop(f)
                Dim fs As vector(Of String) = Nothing
                If Not m.find(f, fs) Then
                    Continue While
                End If
                assert(Not fs Is Nothing)
                fs.stream().
                   filter(Function(ByVal other As String) As Boolean
                              ' filter requires stateless operations.
                              Return tm.find(other) = tm.end()
                          End Function).
                   foreach(Sub(ByVal other As String)
                               q.emplace(other)
                               assert(tm.emplace(other).second())
                           End Sub)
            End While
        End Sub 
```

This function is invoked by a suffix named `call_hierarchy.calculator`:

```vb.net
        Public NotInheritable Class calculator
            Implements statement(Of WRITER)

            Private Shared ReadOnly instance As New calculator()

            Private Sub New()
            End Sub

            Public Shared Sub register(ByVal p As statements(Of WRITER))
                assert(Not p Is Nothing)
                p.register(instance)
            End Sub

            Public Sub export(ByVal o As WRITER) Implements statement(Of WRITER).export
                current().call_hierarchy().calculate()
            End Sub
        End Class 
```

Once the calculation is complete, `tm` records whether each function is reachable at runtime.

The main code can be found in [commit/3cccee2818c139e3044073a51cfc89b8899488f2](https://github.com/Hzj-jie/osi.net/commit/3cccee2818c139e3044073a51cfc89b8899488f2).
