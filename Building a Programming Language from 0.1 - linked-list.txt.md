# Building a Programming Language from 0.1 - linked-list.txt

The previous article [Building a Programming Language from 0.1 - b3style_b2style_self_test](https://zhuanlan.zhihu.com/p/1941474295638394338) mentioned that the failure of `linked-list.txt` to execute was due to a minor bug. The cause was quite simple: the value of `type0` varies in `b3style`, which in turn affected how the compiler determined whether a function had a return value.

```diff
--- a/service/compiler/scope/scope.current_function.vb
+++ b/service/compiler/scope/scope.current_function.vb
@@ -15,15 +15,6 @@ Partial Public Class scope(Of WRITER As {lazy_list_writer, New},
     Protected NotInheritable Class current_function_t
         Inherits function_signature(Of builders.parameter)
 
-        Private Shared ReadOnly type0 As String =
-            Function() As String
-                Dim t As builders.parameter_type = builders.parameter_type.of(logic.scope.type_t.zero_type)
-                If current().features().with_namespace() Then
-                    t = t.map_type(AddressOf namespace_t.fully_qualified_name)
-                End If
-                Return t.full_type()
-            End Function()
-
         Public Sub New(ByVal name As String,
                        ByVal return_type As String,
                        ByVal params As vector(Of builders.parameter))
@@ -31,8 +22,20 @@ Partial Public Class scope(Of WRITER As {lazy_list_writer, New},
         End Sub
 
         Public Function allow_return_value() As Boolean
-            ' This constant needs to match the implementation in logic/scope.type_t.zero_type.
-            Return Not return_type.Equals(type0)
+            ' Note, b3style can run with or without namespace, the type0 isn't a constant for b3style. This can be done
+            ' by creating a b3style.scope_1 with disabled namespace, but the effort isn't very meaningful to test only
+            ' the behavior of bstyle compatibility.
+            ' So instead of caching the constant, calculating its value each time calling this function is a little bit
+            ' wasteful but much simpler.
+            ' A proper solution is indeed to fix the b3style's bstyle compatibility with the namespace enabled.
+            Return Not return_type.Equals(Function() As String
+                                              Dim t As builders.parameter_type =
+                                                  builders.parameter_type.of(logic.scope.type_t.zero_type)
+                                              If current().features().with_namespace() Then
+                                                  t = t.map_type(AddressOf namespace_t.fully_qualified_name)
+                                              End If
+                                              Return t.full_type()
+                                          End Function())
         End Function
```

Originally, `type0` was treated as a constant. However, `b3style` supports a `disable_namespace` mode, under which `type0` is no longer `"::type0"`, leading to incorrect evaluations of whether a function permits a return value.

While `type0`'s definition could alternatively be placed directly in `scope`, the exact placement is not critical.

With the above change, `linked-list.txt` executes normally: [commit/12baade42529e8523a801d4babf9b90562719144](https://github.com/Hzj-jie/osi.net/commit/12baade42529e8523a801d4babf9b90562719144).
