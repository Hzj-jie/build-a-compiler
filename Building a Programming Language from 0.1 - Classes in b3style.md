# Building a Programming Language from 0.1 - Classes in b3style

Just as `b2style` implemented classes ([Building a Programming Language from 0.1 - Implementing Classes with Structs](https://zhuanlan.zhihu.com/p/437868234)), `b3style` can also implement classes using structs and functions. In `b2style`, all functions reside in the root namespace, using a type marker to distinguish different structs: [service/compiler/scope/scope.struct.vb#L53](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope/scope.struct.vb#L53):

```vb.net
        Public Shared Function create_type_id(ByVal name As String) As builders.parameter
            assert(Not name.null_or_whitespace())
            Return builders.parameter.non_ref(name + "__struct__type__id__type", name + "__struct__type__id")
        End Function

        Public Function define(ByVal name As String,
                               ByVal members As vector(Of builders.parameter),
                               ByVal define_type As Action(Of String, UInt32)) As Boolean
            assert(Not name.null_or_whitespace())
            assert(Not members Is Nothing)  ' Allow empty struct
            assert(Not define_type Is Nothing)
            Dim full_type As String = builders.parameter_type.of(name).
                                               map_type(normalized_type.of).
                                               full_type()
            assert(Not builders.parameter_type.is_ref_type(full_type))
            Dim type_id As builders.parameter = create_type_id(name)
            define_type(type_id.map_type(normalized_type.of).non_ref_type(), uint32_1)
            members = members.CloneT().emplace_with(type_id)
```

Strictly speaking, this is not mandatory—especially in `b3style`, which can track types natively. However, for the initial implementation, following `b2style`'s approach has the advantage of allowing reuse of `b2style` code, just as `bstyle` code was reused earlier.

Recalling the implementation in `b2style`, it consists of four main parts:
1. Converting classes into structs and functions
2. Defining class instances
3. Calling methods on class instances
4. Invoking constructors and destructors

## Converting Classes into Structs and Functions

This is actually the simplest part, as we can reuse `b2style` code with `sed`: [service/compiler/b3style/logic/nodes/class.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/class.vb).

The details are omitted here as they match the earlier design.

## Defining Class Instances

In the `b2style` implementation, there was a bug: when encountering a `value-declaration`, `b2style` did not check whether the type was a class or a struct, so such class objects were never destroyed. The fix is simple and discussed later, but `b3style` initially inherited this bug.

Consequently, in this case, class objects are not instantiated via dedicated class code, but handled directly by struct code: [service/compiler/b3style/logic/nodes/value_declaration.vb#L34](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/value_declaration.vb#L34):

```vb.net
        Public Shared Function declare_struct_type(ByVal type As typed_node,
                                                   ByVal name As typed_node,
                                                   ByVal o As logic_writer) As Boolean
            assert(Not type Is Nothing)
            assert(Not name Is Nothing)
            Return struct.define_in_stack(type.input_without_ignored(), value_definition.name_of(name), o)
        End Function
```

## Calling Methods on Class Instances

Here the implementation begins to diverge, even though the high-level logic remains similar. While `b2style` translates method calls into `bstyle` function calls, `b3style` translates them directly into `logic` language code via `b3style`'s `function_call`: [service/compiler/b3style/logic/nodes/function_call.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/function_call.vb):

```vb.net
                Dim struct_func As tuple(Of String, String) = Nothing
                If b2style.function_call.split_struct_function(raw_function_name, struct_func) Then
                    If Not raw_variable_name.build(struct_func.first(), o) Then
                        raise_error(error_type.user, "Cannot find class instance ", struct_func.first())
                        Return False
                    End If
                    parameters = (+scope.current().value_target().value()).names + parameters
                    function_name = scope.current_namespace_t.fully_qualified_name(struct_func.second())
                Else
                    function_name = _function.name_of(raw_function_name)
                End If
```

`split_struct_function` from `b2style.function_call` is used to determine whether a call targets a method of a class object. If so, it adjusts the parameter list and function name before generating `logic` language code using standard call mechanisms:

```vb.net
                Dim name As String = Nothing
                If Not logic_name.of_function_call(function_name, parameters, name) Then
                    Return False
                End If
```

Conceptually, although there is no intermediate code output, it transforms:

```cpp
object.func(param1, param2);
```

into:

```cpp
func(object, param1, param2);
```

where `object` itself acts as `this` for the class method:

```cpp
func(C& this, P1 param1, P2 param2) { }
```

## Invoking Constructors and Destructors

Like `b2style`, `b3style` defines a `class-initializer` grammar rule, since `b3style` shares `b2style`'s grammar definitions. `b3style/class_initializer` mirrors `b2style/class_initializer` with three phases: defining the type, invoking the constructor, and invoking the destructor: [service/compiler/b3style/logic/nodes/class_initializer.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/class_initializer.vb):

```vb.net
            ' Unlike b2style, b3style uses struct declarations directly to define nested variables in the current scope.
            If Not value_declaration.declare_struct_type(n.child(0), n.child(1), o) Then
                Return False
            End If
            ' Next, call the constructor itself via function_call.
            If n.child_count() = 4 Then
                value_list.with_empty()
            ElseIf Not code_gen_of(n.child(3)).build(o) Then
                Return False
            End If
            If Not function_call.without_return(b2style.function_call.build_struct_function(
                                                    n.child(1).input_without_ignored(),
                                                    scope.class_def.construct),
                                                o) Then
                Return False
            End If
            ' Finally, invoke the destructor when the current scope exits.
            ' Functions are always in global namespace.
            ' Use of this function should only be in the !_disable_namespace.
            scope.current().when_end_scope(
                Sub()
                    value_list.with_empty()
                    assert(function_call.without_return(b2style.function_call.build_struct_function(
                                                            n.child(1).input_without_ignored(),
                                                            scope.class_def.destruct),
                                                        o))
                End Sub)
```

Two subsequent statements mark both the constructor and destructor as invoked so that dead-code elimination does not strip them: [Building a Programming Language from 0.1 - Optimizing by Removing Unused Functions from Compilation Output](https://zhuanlan.zhihu.com/p/443777931).

The remaining work involved running class-related tests from `b2style` under `b3style`, along with minor supporting modifications.

One edge case: `bstyle` allowed `f.f(1)` to call a standalone function named `f.f`, whereas `b2style` and `b3style` interpret this as `f(f, 1)`. That test was accordingly removed from `b3style`: [service/tests/compiler/bstyle/bstyle_test.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/bstyle_test.vb):

```vb.net
+    ' b3style treat f.f() as f(f) / class function.
+    <test>
+    Private Shared Sub func_name_with_dot()
+        Dim io As New console_io.test_wrapper()
+        Dim e As executor = Nothing
+        assertion.is_true(bstyle.with_functions(New interrupts(+io)).
+                                 parse(_bstyle_test_data.func_name_with_dot.as_text(), e))
+        assertion.is_not_null(e)
+        e.assert_execute_without_errors()
+        assertion.equal(io.output().Length(), 2)
+        assertion.equal(Convert.ToInt32(io.output()(0)), 1)
+        assertion.equal(Convert.ToInt32(io.output()(1)), 2)
+    End Sub
```

The main changes can be found in [commit/487797dc5a50d95f6bc38fb2b67fe5b0be14d9e4](https://github.com/Hzj-jie/osi.net/commit/487797dc5a50d95f6bc38fb2b67fe5b0be14d9e4).
