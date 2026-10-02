# Building a Programming Language from 0.1 - Syntax Analysis of Templates in b3style

The theoretical principles behind template implementation were covered previously in [Building a Programming Language from 0.1 - Simple Pre-expanded Templates](https://zhuanlan.zhihu.com/p/451620752), making template support in `b3style` primarily an engineering challenge. Considering that the primary implementation of templates resides in `scope` and `b2style`, and that `scope` is decoupled from language-specific backends ([Building a Programming Language from 0.1 - b3style/scope](https://zhuanlan.zhihu.com/p/578969284)), the central problem becomes maximizing code reuse from `b2style`'s template implementation.

When enabling `bstyle` compatibility in `b3style` ([Building a Programming Language from 0.1 - Supporting bstyle in b3style](https://zhuanlan.zhihu.com/p/639883836)), a simple `sed` script was heavily employed to generate `b3style` code from `bstyle` source. The same strategy can be used to generate `b3style` code from `b2style`. However, one crucial difference is that `b3style` emits code via `logic_writer`, whereas `b2style` uses `typed_node_writer`—a distinction that prevents direct sharing of code generation routines.

The first step was to decouple shareable logic from backend-specific code using VB.NET / C# partial classes. For example, `b2style.function` was split into [function.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/function.vb) and [function.template.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/function.template.vb): the former handles code generation, while the latter houses shared template handling. Consequently, `b3style.function` only needs its own implementation of `function.vb`.

`b2style/function.vb`:

```vb.net
Partial Public NotInheritable Class b2style
    Partial Private NotInheritable Class _function
        Implements code_gen(Of typed_node_writer)

        Private Function build(ByVal n As typed_node,
                               ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            Dim function_name As String = _namespace.bstyle_format.of(n.child(1))
            Using scope.current().start_scope().current_function().define(function_name)
                Dim fo As New typed_node_writer()
                Return code_gens().of_all_children(n).build(fo) AndAlso
                       o.append(scope.current().call_hierarchy().filter(function_name, AddressOf fo.str))
            End Using
        End Function
```

Corresponding `b3style/function.vb`:

```vb.net
Partial Public NotInheritable Class b3style
    Private NotInheritable Class _function
        Implements code_gen(Of logic_writer)

        Private Shared remove_unused_functions As argument(Of Boolean)

        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            Using new_scope As scope = scope.current().start_scope()
                Dim fo As New logic_writer()
                Dim has_paramlist As Boolean = strsame(n.child(3).type_name, "paramlist")
                If has_paramlist Then
                    If Not code_gen_of(n.child(3)).build(fo) Then
                        Return False
                    End If
                End If
                Dim function_name As String = scope.fully_qualified_function_name.of(n.child(1))
                Dim params As vector(Of builders.parameter) = new_scope.params().unpack()
                Return logic_name.of_callee(function_name,
                                            scope.type_name.of(n.child(0)),
                                            params,
                                            Function() As Boolean
                                                Dim gi As UInt32 = CUInt(If(has_paramlist, 5, 4))
                                                Return code_gen_of(n.child(gi)).build(fo)
                                            End Function,
                                            fo) AndAlso
                       o.append(new_scope.call_hierarchy().filter(
                                    logic_name.of_function(function_name, +params),
                                    AddressOf fo.str))
            End Using
        End Function
```

Shared components are translated across dialects using [b3style/from-b2style.cmd](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/from-b2style.cmd):

```bat
  setlocal

  path %PATH%;..\..\..\..\..\root\codegen\sed;

  call :run class
+ call :run function.template
+ call :run function_call_with_template
+ call :run function_name_with_template
+ call :run delegate_with_semi_colon
+ call :run template_type_name.template

  exit /b 0

  :RUN
  sed " Class b2style" " Class b3style" ..\..\..\b2style\rewriter\nodes\%~1.vb | sed " typed_node_writer" " logic_writer" > %~1.vb
  exit /b 0
```

Likewise, [b3style.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/b3style.vb) registers these newly shared code generators alongside handlers supplied directly by `scope`:

```diff
diff --git a/service/compiler/b3style/b3style.vb b/service/compiler/b3style/b3style.vb
index d78ebc72..79465627 100644
--- a/service/compiler/b3style/b3style.
+++ b/service/compiler/b3style/b3style.vb
@@ -71,7 +71,6 @@ Partial Public NotInheritable Class b3style
         Protected Overrides Function at() As vector(Of Action(Of code_gens(Of logic_writer)))
             Return New code_gens_registrar(Of logic_writer)().
                 with_of_only_childs(
-                    "root-type",
                     "base-root-type",
                     "include",
                     "typedef-type",
@@ -83,12 +82,15 @@ Partial Public NotInheritable Class b3style
                     "variable-name",
                     "for-increase",
                     "base-for-increase",
-                    "unary-operation-value"
+                    "unary-operation-value",
+                    "ignore-result-function-call-with-template",
+                    "type-name"
                 ).
                 with_of_all_childrens(
                     "paramtypelist",
                     "raw-value",
                     "paramlist",
+                    "paramtype",
                     "value-with-operation"
                 ).
                 with(code_gen.of_ignore_last_child(Of logic_writer)("root-type-with-semi-colon")).
@@ -98,8 +100,8 @@ Partial Public NotInheritable Class b3style
                 with(code_gen.of_first_child(Of logic_writer)("value-with-comma")).
                 with(code_gen.of_first_child(Of logic_writer)("paramtype-with-comma")).
                 with(code_gen.of_children(Of logic_writer)("else-condition", 1)).
-                with(code_gen.of_children(Of logic_writer)("value-with-bracket", 1)).
-                with(code_gen.of_input_without_ignored(Of logic_writer)("paramtype")).
+                with(code_gen.of_children(Of logic_writer)("value-with-bracket", 1)).
+                with(code_gen.of_only_descendant_str(Of logic_writer)("reference")).
  _
                 with(Of bstyle.logic)().
                 with(Of bstyle.typedef_type_name)().
@@ -121,7 +123,7 @@ Partial Public NotInheritable Class b3style
                 with_delegate("ignore-result-function-call",
                               Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
                                   assert(Not n Is Nothing)
-                                  Return function_call.without_return(n.child(), o)
+                                  Return function_call.with_parameters.without_return(n.child(), o)
                               End Function).
                 with(Of param)().
                 with(Of return_clause)().
@@ -210,7 +212,31 @@ Partial Public NotInheritable Class b3style
                 with("post-operation-value", New unary_operation_value(1, "_post")).
                 with_delegate("self-value-clause", AddressOf binary_operation_value.without_return).
                 with(Of _class)().
-                with(Of class_initializer)()
+                with(Of class_initializer)().
+ _
+                with_delegate("template",
+                              Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
+                                  Dim name As String = Nothing
+                                  Dim t As scope.template_template = Nothing
+                                  Return scope.template_t.of(n, name, t) AndAlso
+                                            scope.current().template().define(name, t)
+                              End Function).
+                with(code_gen.of_first_child(Of logic_writer)("type-param-with-comma")).
+                with(code_gen.of_only_descendant_str(Of logic_writer)("type-param")).
+                with(Of function_call_with_template)().
+                with(Of function_name_with_template)().
+                with_delegate("root-type",
+                              Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
+                                  scope.current().root_type_injector()._new(o)
+                                  Return code_gen_of(n.child()).build(o)
+                              End Function).
+                with(Of delegate_with_semi_colon)().
+                with_delegate("raw-type-name",
+                              Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
+                                  Return o.append(scope.normalized_type.of(n))
+                              End Function).
+                with(code_gen.of_first_child(Of logic_writer)("type-name-with-comma")).
+                with(Of template_type_name)()
         End Function
     End Class
 End Class
```

With these registrations in place, `b3style` can parse template syntax, although execution of extensive template features remains to be completed.

The pull request with the full set of changes is available at [pull/98](https://github.com/Hzj-jie/osi.net/pull/98).
