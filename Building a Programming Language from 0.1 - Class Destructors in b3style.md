# Building a Programming Language from 0.1 - Class Destructors in b3style

[Building a Programming Language from 0.1 - Classes in b3style](https://zhuanlan.zhihu.com/p/711767428) mentioned that `b2style` only generated destructor calls upon encountering a `class-initializer`, a limitation that `b3style` initially inherited. Naturally, this behavior was incorrect, though there was an underlying reason for the flaw in `b2style`: `b2style` did not implement dedicated handlers for `value-declaration` and `value-definition`, but rather delegated them directly to `bstyle`: [service/compiler/b2style/b2style.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/b2style.vb):

```vb.net
with(code_gen.of_all_children_with_precondition(Of typed_node_writer)(
         "value-definition",
         scope.variable_proxy.define(),
         scope.call_hierarchy_t.from_value_clause())).
with(code_gen.of_all_children_with_precondition(Of typed_node_writer)(
         scope.variable_proxy.define(), "value-declaration")).
```

To support destructors properly, `b3style` must handle these two grammar rules on its own, allowing destructor calls to be generated. Saying "generating destructor calls" is precise: the destructor call is queued to execute when the enclosing lexical scope terminates.

## value-declaration, value-definition, and class-initializer

This is an interesting topic often highlighted in C++ tutorials: there are multiple ways an object of a class type can be instantiated:

```cpp
C a;
C b(p1, p2);
C c{p1, p2};
C d = f(p1, p2);
C e = C(p1, p2);
```

Here, `b` and `c` are equivalent (since C++11 recommends `{}` for constructor invocation to prevent the most vexing parse). `e` has no unique semantics, but can lead to ambiguity over whether copy constructors, move constructors, or assignment operators are invoked. Consider the following C++ example:

```cpp
#include <iostream>
using namespace std;

class C {
public:
  C(int x) {
    cout << "C(" << x << ")" << endl;
  }
  
  C(C&&) {
    cout << "C(C&&)" << endl;
  }
  
  C& operator=(const C&) {
    cout << "C& operator=(const C&)" << endl;
    return *this;
  }
  
  C& operator=(C&&) {
    cout << "C& operator=(C&&)" << endl;
    return *this;
  }
};

int main() {
    C b(1);
    C e = C(2);
    C f{3};
    f = C(4);

    return 0;
}
```

Because `b3style` does not implement the entirety of C++ syntax, it only considers cases `a`, `b`, and `d`, corresponding respectively to `value-declaration`, `class-initializer`, and `value-definition`.

## value-declaration and class-initializer

These two forms are essentially identical; the only difference is the constructor invoked. The former passes only `this`, while the latter may pass additional arguments. For empty constructors, the emitted code is completely identical, making the term `value-declaration` somewhat of a misnomer:

```cpp
class C { };
C c1;
C c2();
```

For a `class-initializer`, constructors and destructors are unconditionally emitted. But for a `value-declaration`, the compiler must verify whether the type is indeed a class. If so, it invokes the constructor and queues the destructor just like a `class-initializer`, supplying an empty argument list:

[service/compiler/b3style/logic/nodes/value_declaration.vb](https://github.com/Hzj-jie/osi.net/blob/c59d63f4e438f0a7fd3ba9b90b52ad8e11458f31/service/compiler/b3style/logic/nodes/value_declaration.vb#L15):

```vb.net
            Dim type As String = n.child(0).input_without_ignored()
            Dim name As String = value_definition.name_of(n.child(1))
            If struct.define_in_stack(type, name, o) Then
                If scope.current().classes().is_defined(type) Then
                    ' No extra parameters to call the constructor.
                    value_list.with_empty()
                    Return class_initializer.construct_and_destruct(name, o)
                End If
                Return True
            End If
            Return declare_primitive_type(type, name, o)
```

`class_initializer.construct_and_destruct` simply refactors the previous implementation into a reusable helper:

[service/compiler/b3style/logic/nodes/class_initializer.vb](https://github.com/Hzj-jie/osi.net/blob/c59d63f4e438f0a7fd3ba9b90b52ad8e11458f31/service/compiler/b3style/logic/nodes/class_initializer.vb#L17):

```vb.net
            If Not function_call.without_return(b2style.function_call.build_struct_function(
                                                    name,
                                                    scope.class_def.construct),
                                                o) Then
                Return False
            End If
            ' Functions are always in global namespace.
            ' Use of this function should only be in the !_disable_namespace.
            scope.current().when_end_scope(
                Sub()
                    value_list.with_empty()
                    assert(function_call.without_return(b2style.function_call.build_struct_function(
                                                            name,
                                                            scope.class_def.destruct),
                                                        o))
                End Sub)
            scope.current().
                  call_hierarchy().
                  to(scope.current_namespace_t.fully_qualified_name(scope.class_def.construct))
            scope.current().
                  call_hierarchy().
                  to(scope.current_namespace_t.fully_qualified_name(scope.class_def.destruct))
            Return True
```

This invokes the constructor, registers the destructor on scope exit via `when_end_scope`, and marks both methods as referenced in the call hierarchy to protect them from dead-code elimination.

A test was added in [service/tests/compiler/b3style/codegen/destruction_in_declaration.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b3style/codegen/destruction_in_declaration.txt):

```cpp
#include <b2style.h>
#include <b2style/stdio.h>

class C {
  void construct() {
    b2style::std_out("construct");
  }

  void destruct() {
    b2style::std_out("destruct");
  }
};

void main() {
  C c;
  b2style::std_out("finish");
}
```

This test produces `constructfinishdestruct`: [service/tests/compiler/b3style/b3style_test.vb#L29](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b3style/b3style_test.vb#L29):

```vb.net
    <test>
    Private Shared Sub class_destruct_in_declaration()
        Dim io As New console_io.test_wrapper()
        Dim e As executor = Nothing
        assertion.is_true(b3style.with_functions(New interrupts(+io)).compile(
                          _b3style_test_data.destruction_in_declaration.as_text(), e))
        assertion.is_not_null(e)
        e.assert_execute_without_errors()
        assertion.equal(io.output(), String.Concat("construct", "finish", "destruct"))
    End Sub
```

These changes can be viewed in [commit/c59d63f4e438f0a7fd3ba9b90b52ad8e11458f31](https://github.com/Hzj-jie/osi.net/commit/c59d63f4e438f0a7fd3ba9b90b52ad8e11458f31).

## value-definition

Unlike the previous two cases, `value-definition` does not invoke the class constructor—it copies the value from the right-hand expression—yet it still requires destructor invocation when going out of scope. Therefore, `class-initializer.construct_and_destruct` was split into separate `construct` and `destruct` routines:

[service/compiler/b3style/logic/nodes/class_initializer.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/class_initializer.vb):

```vb.net
        Public Shared Function construct(ByVal name As String, ByVal o As logic_writer) As Boolean
            assert(Not name.null_or_whitespace())
            assert(Not o Is Nothing)

            If Not function_call.without_return(b2style.function_call.build_struct_function(
                                                    name,
                                                    scope.class_def.construct),
                                                o) Then
                Return False
            End If
            scope.current().
                  call_hierarchy().
                  to(scope.current_namespace_t.fully_qualified_name(scope.class_def.construct))

            Return True
        End Function

        Public Shared Sub destruct(ByVal name As String, ByVal o As logic_writer)
            ' Functions are always in global namespace.
            ' Use of this function should only be in the !_disable_namespace.
            scope.current().when_end_scope(
                Sub()
                    value_list.with_empty()
                    assert(function_call.without_return(b2style.function_call.build_struct_function(
                                                            name,
                                                            scope.class_def.destruct),
                                                        o))
                End Sub)
            scope.current().
                  call_hierarchy().
                  to(scope.current_namespace_t.fully_qualified_name(scope.class_def.destruct))
        End Sub
```

`value-definition` invokes `value-declaration` to perform the declaration and `value-clause` to perform assignment: [service/compiler/b3style/logic/nodes/value_definition.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/value_definition.vb):

```vb.net
             Return value_declaration.of(n, o) AndAlso
                    value_clause.stack_name_build(n.child(1), n.child(3), o)
```

Accordingly, `value-declaration` was updated so that `value_declaration.of` also registers destructor execution, providing two modes: one that invokes the constructor and one that does not:

```vb.net
        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            Return build(n,
                         Function(ByVal name As String) As Boolean
                             value_list.with_empty()
                             Return class_initializer.construct(name, o)
                         End Function,
                         o)
        End Function

        Public Shared Function [of](ByVal n As typed_node, ByVal o As logic_writer) As Boolean
            Return build(n,
                         Function(ByVal name As String) As Boolean
                             Return True
                         End Function,
                         o)
        End Function
```

The shared `build` function accepts a delegate responsible for constructor generation, passing an empty callback for `value-definition`: [service/compiler/b3style/logic/nodes/value_declaration.vb#L15](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/value_declaration.vb#L15):

```vb.net
        Private Shared Function build(ByVal n As typed_node,
                                      ByVal class_construct As Func(Of String, Boolean),
                                      ByVal o As logic_writer) As Boolean
            assert(Not n Is Nothing)
            assert(Not class_construct Is Nothing)
            assert(Not o Is Nothing)
            assert(n.child_count() >= 2)
            Dim type As String = n.child(0).input_without_ignored()
            Dim name As String = value_definition.name_of(n.child(1))
            If struct.define_in_stack(type, name, o) Then
                If scope.current().classes().is_defined(type) Then
                    If class_construct(name) Then
                        class_initializer.destruct(name, o)
                        Return True
                    End If
                    Return False
                End If
                Return True
            End If
            Return declare_primitive_type(type, name, o)
        End Function
```

A test was added in [service/tests/compiler/b3style/codegen/destruction_in_definition.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b3style/codegen/destruction_in_definition.txt):

```cpp
#include <b2style.h>
#include <b2style/stdio.h>

class C {
  void construct() {
    b2style::std_out("construct");
  }

  void destruct() {
    b2style::std_out("destruct");
  }
};

C create() {
  b2style::std_out("create");
  C c;
  return c;
}

void main() {
  C c = create();
  b2style::std_out("finish");
}
```

This test outputs `createconstructfinishdestruct`.

The main changes can be found in [commit/8d001869317df9bf4c7b163b11ebaf2d4082952d](https://github.com/Hzj-jie/osi.net/commit/8d001869317df9bf4c7b163b11ebaf2d4082952d).
