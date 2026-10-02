# Building a Programming Language from 0.1 - heap_struct_name in b3style

Previously, [b3style_b2style_self_test](https://github.com/Hzj-jie/osi.net/blob/be159b71db42c81064b234faf3b0a2d1959c8226/service/tests/compiler/b3style/b3style_b2style_self_test.vb#L27-L30) skipped tests related to `static_cast` because `b3style` did not yet support `heap_struct_name`:

```vb.net
' Missing heap-struct-name - assertion failure
r = r OrElse unordered_set.of("static_cast_ptr_type3.txt",
                              "static_cast_ptr_type_bool.txt",
                              "static_cast_ptr_type.txt").find(name).is_not_end()
```

Supporting this syntax was the natural next step. `static_cast_ptr_type2.txt` was skipped for the same underlying reason, though the test comment was somewhat imprecise.

`heap_struct_name` appears in two distinct contexts:
1. As variable expressions (reading and writing members of heap-allocated structs).
2. Invoking methods on heap-allocated structs.

As a variable expression, consider [static_cast_ptr_type.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type.txt), reading and writing fields of a heap-allocated struct:

```cpp
#include <b2style.h>
#include <testing.h>
#include <b2style/to_str.h>

struct S {
  int x;
  string y;
};

void f(S s) {
  for (int i = 0; i < 10; i++) {
    b2style::testing::assert_equal<int>(s[i].x, i);
    b2style::testing::assert_equal<string>(s[i].y, b2style::int_to_str(i));
  }
}

void f(type_ptr p) {
  static_cast(p, S);
  f(p);
}

void main() {
  S s[10];
  for (int i = 0; i < 10; i++) {
    s[i].x = i;
    s[i].y = b2style::to_str(i);
  }
  static_cast(s, type_ptr);
  f(s);
  b2style::testing::finished();
}
```

As method calls on heap-allocated structs, consider [class_on_heap.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/class_on_heap.txt), invoking `cs[i].print()`:

```cpp
#include <b2style.h>
#include <b2style/stdio.h>

class C {
  int x;
  string y;

  void print() {
    b2style::std_out(this.x);
    b2style::std_out(" ");
    b2style::std_out(this.y);
  }
};

class C2 {
  C x;
  int y;

  void print() {
    this.x.print();
    b2style::std_out(" ");
    b2style::std_out(this.y);
    b2style::std_out("\n");
  }
};

void main() {
  C2 cs[100];
  for (int i = 0; i < 100; i++) {
    cs[i].x.x = i;
    cs[i].x.y = b2style::int_to_str(i);
    cs[i].y = i;
  }

  for (int i = 0; i < 100; i++) {
    cs[i].print();
  }
}
```

These correspond to three grammar constructs registered in [b3style.vb#L127](https://github.com/Hzj-jie/osi.net/blob/177bfd85be44f1d0ed4909c9e26fc1ae1a4bd68b/service/compiler/b3style/b3style.vb#L127):

```vb.net
+        with(Of heap_struct_function_call)().
+        with_delegate("ignore-result-heap-struct-function-call",
+                      Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
+                          assert(Not n Is Nothing)
+                          Return heap_struct_function_call.with_parameters.without_return(n.child(), o)
+                      End Function).
         with(Of param)().
...
         with(code_gen.of_first_child(Of logic_writer)("type-name-with-comma")).
-        with(Of template_type_name)()
+        with(Of template_type_name)().
+        with(Of heap_struct_name)()
```

`heap_struct_function_call` and `ignore-result-heap-struct-function-call` represent method calls, whereas `heap_struct_name` represents variable accesses. As with standard functions, whether a return value is received does not alter the underlying call convention, so both forms are implemented inside `heap_struct_function_call`.

First, consider [heap_struct_name](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/heap_struct_name.vb). This is a specialized `heap_name`: following `b2style`, `a[i].b` translates to `a.b[i]` on the heap:

```vb.net
 Partial Public NotInheritable Class b3style
    Private NotInheritable Class heap_struct_name
        Implements code_gen(Of logic_writer)

        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            Dim t As tuple(Of String, typed_node) = b2style.heap_struct_name.logic_order(n)
            Return heap_name.as_raw_variable_name(t.second(), t.first(), o)
        End Function
    End Class
End Class
```

The process comprises two steps: `b2style.heap_struct_name.logic_order` and `heap_name.as_raw_variable_name`.

[b2style.heap_struct_name.logic_order](https://github.com/Hzj-jie/osi.net/blob/89eec7be6196626ff3858c74ca50c77b8a7cb296/service/compiler/b2style/rewriter/nodes/heap_struct_name.vb#L16) decomposes `a[i].b` into `a.b` and index `i`, returning them in a tuple:

```vb.net
' Return name + index_node
Public Shared Function logic_order(ByVal n As typed_node) As tuple(Of String, typed_node)
    assert(n.child_count() = 3)
    assert(n.child(0).child_count() = 4)
    Return tuple.emplace_of(streams.of(n.child(0).child(0),
                                       n.child(1),
                                       n.child(2)).
                                    map(Function(ByVal x As typed_node) As String
                                            assert(Not x Is Nothing)
                                            Return x.input_without_ignored()
                                        End Function).
                                    collect_by(stream(Of String).collectors.to_str("")).
                                    ToString(),
                            n.child(0).child(2))
End Function
```

Next, [heap_name.as_raw_variable_name](https://github.com/Hzj-jie/osi.net/blob/89eec7be6196626ff3858c74ca50c77b8a7cb296/service/compiler/b3style/logic/nodes/heap_name.vb#L46):

```vb.net
Public Shared Function as_raw_variable_name(ByVal index As typed_node,
                                            ByVal name As String,
                                            ByVal o As logic_writer) As Boolean
    assert(Not index Is Nothing)
    assert(Not name.null_or_whitespace())
    Return build(index,
                 o,
                 Function(ByVal indexstr As String) As Boolean
                     Return raw_variable_name.build(
                                name,
                                Function(ByVal type As String,
                                         ByVal ps As stream(Of builders.parameter)) As Boolean
                                    scope.
                                        current().
                                        value_target().
                                        with_value(
                                            type,
                                            ps.map(Function(ByVal d As builders.parameter) As builders.parameter
                                                       assert(Not d Is Nothing)
                                                       Return d.map_name(
                                                               Function(ByVal n As String) As String
                                                                   Return variable.name_of(n, indexstr)
                                                               End Function)
                                                   End Function))
                                    Return True
                                End Function)
                 End Function)
End Function
```

Notice that `name` is now a `String` rather than a single `typed_node`, because `a` and `.b` do not form a contiguous AST node. This refactoring propagated into [raw_variable_name](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/raw_variable_name.vb):

```vb.net
-        Private Shared Function build(ByVal name As String,
-                                      ByVal handle As Func(Of String, stream(Of builders.parameter), Boolean)) _
-                                     As Boolean
+        Public Shared Function build(ByVal name As String,
+                                     ByVal handle As Func(Of String, stream(Of builders.parameter), Boolean)) _
+                                    As Boolean
             assert(Not name.null_or_whitespace())
             assert(Not handle Is Nothing)

@@ -38,12 +38,6 @@ Partial Public NotInheritable Class b3style
             Return handle(p.full_type(), ps.primitives())
         End Function

-        Public Shared Function build(ByVal n As typed_node,
-                                     ByVal handle As Func(Of String, stream(Of builders.parameter), Boolean)) As Boolean
-            assert(Not n Is Nothing)
-            Return build(scope.variable_name.of(n), handle)
-        End Function
```

Next, [value_clause](https://github.com/Hzj-jie/osi.net/blob/89eec7be6196626ff3858c74ca50c77b8a7cb296/service/compiler/b3style/logic/nodes/value_clause.build.vb#L25) must handle `heap-struct-name`:

```vb.net
If name.type_name.Equals("heap-name") Then
    Return heap_name_build(name, value, o)
End If
If name.type_name.Equals("heap-struct-name") Then
    Dim t As tuple(Of String, typed_node) = b2style.heap_struct_name.logic_order(name)
    Return heap_name_build(t.second(), t.first(), value, o)
End If
```

At this stage, `static_cast_ptr_type.txt` ran successfully. Supporting method calls on heap structs required generalizing `function_call`:

```diff
diff --git a/service/compiler/b3style/logic/nodes/function_call.vb b/service/compiler/b3style/logic/nodes/function_call.vb
index 9354fddd..14b080e0 100644
--- a/service/compiler/b3style/logic/nodes/function_call.vb
+++ b/service/compiler/b3style/logic/nodes/function_call.vb
@@ -11,40 +11,59 @@ Imports osi.service.automata
 Imports builders = osi.service.compiler.logic.builders

 Partial Public NotInheritable Class b3style
-    Private NotInheritable Class function_call
+    Private Class function_call(Of _name_builder As func_t(Of String, typed_node, logic_writer, Boolean))
         Implements code_gen(Of logic_writer)

+        Private Shared ReadOnly name_builder As _name_builder = alloc(Of _name_builder)()
+
+        Private Shared Function build_function_caller(
+                                    ByVal function_name As String,
+                                    ByVal parameters As vector(Of String),
+                                    ByVal build_caller As Func(Of String, vector(Of String), Boolean)) As Boolean
+            assert(Not function_name.null_or_whitespace())
+            assert(Not parameters Is Nothing)
+            assert(Not build_caller Is Nothing)
+            Dim name As String = Nothing
+            If Not logic_name.of_function_call(function_name, parameters, name) Then
+                Return False
+            End If
+            scope.current().call_hierarchy().to(name)
+            Return build_caller(name, parameters)
+        End Function
+
         Private Shared Function build(ByVal raw_function_name As String,
+                                      ByVal name_node As typed_node,
                                       ByVal build_caller As Func(Of String, vector(Of String), Boolean),
                                       ByVal build_caller_ref As Func(Of String, vector(Of String), Boolean),
                                       ByVal o As logic_writer) As Boolean
             assert(Not build_caller Is Nothing)
             assert(Not build_caller_ref Is Nothing)
+            assert(Not raw_function_name.null_or_whitespace() OrElse Not name_node Is Nothing)
+            If raw_function_name Is Nothing Then
+                raw_function_name = scope.function_name.of(name_node)
+            End If
+            assert(Not raw_function_name.null_or_whitespace())
             Using targets As read_scoped(Of vector(Of String)).ref = value_list.current_targets()
                 Dim parameters As vector(Of String) = +targets
-                If scope.current().variables().defined(raw_function_name) Then
-                    Return build_caller_ref(raw_function_name, parameters)
-                End If
-
                 Dim struct_func As tuple(Of String, String) = Nothing
-                Dim function_name As String = Nothing
+                ' The split_struct_function needs to be executed first, otherwise the
+                ' scope.current().variables().defined() will check is_heap_name which expects the [] to be at the end
+                ' or triggers an assertion failure.
                 If b2style.function_call.split_struct_function(raw_function_name, struct_func) Then
-                    If Not raw_variable_name.build(struct_func.first()) Then
+                    If Not name_builder.run(struct_func.first(), name_node, o) Then
                         raise_error(error_type.user, "Cannot find class instance ", struct_func.first())
                         Return False
                     End If
-                    parameters = (+scope.current().value_target().value()).names + parameters
-                    function_name = scope.namespace_t.fully_qualified_name(struct_func.second())
-                Else
-                    function_name = scope.fully_qualified_function_name.of(raw_function_name)
+                    Return build_function_caller(scope.namespace_t.fully_qualified_name(struct_func.second()),
+                                                 (+scope.current().value_target().value()).names + parameters,
+                                                 build_caller)
                 End If
-                assert(Not function_name Is Nothing)
-                Dim name As String = Nothing
-                If Not logic_name.of_function_call(function_name, parameters, name) Then
-                    Return False
+                If scope.current().variables().defined(raw_function_name) Then
+                    Return build_caller_ref(raw_function_name, parameters)
                 End If
-                scope.current().call_hierarchy().to(name)
-                Return build_caller(name, parameters)
+                Return build_function_caller(scope.fully_qualified_function_name.of(raw_function_name),
+                                             parameters,
+                                             build_caller)
             End Using
         End Function

@@ -124,14 +143,19 @@ Partial Public NotInheritable Class b3style

         Public NotInheritable Class ignore_parameters
             Public Shared Function without_return(ByVal function_name As String, ByVal o As logic_writer) As Boolean
-                Return function_call.build(function_name,
+                Return function_call(Of _name_builder).build(function_name,
+                                                             Nothing,
                                                              without_return_caller_builder(o),
                                                              without_return_caller_ref_builder(o),
                                                              o)
             End Function

             Public Shared Function build(ByVal function_name As String, ByVal o As logic_writer) As Boolean
-                Return function_call.build(function_name, caller_builder(o), caller_ref_builder(o), o)
+                Return function_call(Of _name_builder).build(function_name,
+                                                             Nothing,
+                                                             caller_builder(o),
+                                                             caller_ref_builder(o),
+                                                             o)
             End Function

             Private Sub New()
@@ -146,16 +170,13 @@ Partial Public NotInheritable Class b3style
                                           ByVal o As logic_writer) As Boolean
                 assert(Not n Is Nothing)
                 assert(Not o Is Nothing)
-                If name Is Nothing Then
-                    name = scope.function_name.of(n.child(0))
-                End If
                 assert(n.child_count() >= 3)
                 If n.child_count() = 3 Then
                     value_list.with_empty()
                 ElseIf Not code_gen_of(n.child(2)).build(o) Then
                     Return False
                 End If
-                Return function_call.build(name, build_caller, build_caller_ref, o)
+                Return function_call(Of _name_builder).build(name, n.child(0), build_caller, build_caller_ref, o)
             End Function

             Public Shared Function without_return(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
@@ -201,5 +222,31 @@ Partial Public NotInheritable Class b3style
             Return with_parameters.build(function_name, n, o)
         End Function
     End Class
+
+    Private NotInheritable Class function_call
+        Inherits function_call(Of raw_variable_name_of)
+
+        Public Structure raw_variable_name_of
+            Implements func_t(Of String, typed_node, logic_writer, Boolean)
+
+            Public Function run(ByVal i As String, ByVal j As typed_node, ByVal k As logic_writer) As Boolean _
+                               Implements func_t(Of String, typed_node, logic_writer, Boolean).run
+                Return raw_variable_name.build(i)
+            End Function
+        End Structure
+    End Class
+
+    Private NotInheritable Class heap_struct_function_call
+        Inherits function_call(Of heap_struct_name_of)
+
+        Public Structure heap_struct_name_of
+            Implements func_t(Of String, typed_node, logic_writer, Boolean)
+
+            Public Function run(ByVal i As String, ByVal j As typed_node, ByVal k As logic_writer) As Boolean _
+                               Implements func_t(Of String, typed_node, logic_writer, Boolean).run
+                Return heap_name.build(j.child(0), k)
+            End Function
+        End Structure
+    End Class
 End Class
```

This addresses two fundamental requirements:
1. Stack variables and heap variables are expanded differently (`raw_variable_name` vs `heap_name`).
2. Checking whether `a[i].b` is a delegate is deferred, because `is_heap_name` asserts that `[i]` appears at the end. Note that this implies `a[i].b` cannot currently be a delegate, which can be addressed if needed in the future.

Beyond these points, the core principle is straightforward: translate `a[i].b(c)` into `::b(a[i], c)`, while ensuring `a.b(c)` continues to map to `::b(a, c)`. The key line enabling this is:

```diff
-    If Not raw_variable_name.build(struct_func.first()) Then
+    If Not name_builder.run(struct_func.first(), name_node, o) Then
```

With these tests integrated, all target scenarios pass in `b3style`. The full change is available at [commit/ff051a4a6f92bc4b3a0dbf5202ba956c53f335f0](https://github.com/Hzj-jie/osi.net/commit/ff051a4a6f92bc4b3a0dbf5202ba956c53f335f0).
