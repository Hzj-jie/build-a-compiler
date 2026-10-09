# Building a Programming Language from 0.1 - Inheriting Class Member Variables

When a class inherits from a base class, handling its member variables is relatively straightforward: we simply copy all the member variables from its base classes. For example:

```cpp
class B {
  int x;
};

class C : B {
  int y;
};

```

is equivalent to generating two structs:

```cpp
struct B {
  int x;
};

struct C {
  int x;
  int y;
};

```

As far as the variables themselves are concerned, `C` and `B` have no inherent runtime connection.

For the compiler, however, that relationship cannot be ignored: no one is going to tell the compiler manually that `C` contains both `int x` and `int y`—the compiler has to remember that on its own.

Let's start with the grammar as usual. Since the colon token already exists in the lexer, no lexer changes are needed ([service/compiler/b2style/codegen/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/codegen/syntaxer_rule.txt)):

```text
type-name-with-comma type-name comma
inherited-types [type-name-with-comma]* type-name
class-inheritance colon inherited-types
class kw-class type-name [class-inheritance]? start-paragraph [struct-body, function]* end-paragraph semi-colon

```

A class may inherit from one or more base classes, or none at all, so `class-inheritance` is optional and consists of a colon followed by a list of types.

How classes are processed during compilation also needs to change. Previously, we simply rewrote a class's member variables into a `struct` of the same name; now, we first collect its own variables, merge in all variables from its inherited base classes, and finally generate the `struct` code.

To do this, we first need a data structure to record the contents of a class—for now, just its member variables ([osi.net/class_def.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/class_def.vb)):

```vb.net
Public NotInheritable Class class_def
    Private ReadOnly name As name_with_namespace
    ' The type-name pair directly passes to bstyle/struct.
    Private ReadOnly _vars As New vector(Of builders.parameter)()

    Public Function append(ByVal other As class_def) As class_def
        assert(Not other Is Nothing)
        _vars.emplace_back(other._vars)
        Return Me
    End Function

    Public Function with_var(ByVal p As builders.parameter) As class_def
        assert(Not p Is Nothing)
        assert(Not p.ref)
        _vars.emplace_back(p)
        Return Me
    End Function

    Public Function vars() As stream(Of builders.parameter)
        Return _vars.stream()
    End Function

```

This class provides two ways to add member variables: `with_var` for variables declared in the class itself, and `append` for variables inherited from a base class.

Next, `scope.classes` needs to store class definitions and provide a lookup method so that when generating a class definition, we can look up its base classes in `scope.classes` and merge them via `append` ([service/compiler/b2style/scope.class.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/scope.class.vb)):

```text
Partial Public NotInheritable Class scope
    Private NotInheritable Class class_t
        Private ReadOnly m As New unordered_map(Of name_with_namespace, class_def)()

        Public Function define(ByVal name As name_with_namespace, ByVal def As class_def) As Boolean
            assert(Not def Is Nothing)
            If Not def.check_duplications() Then
                Return False
            End If
            If m.emplace(name, def).second() Then
                Return True
            End If
            raise_error(error_type.user, "Class ", name, " has been defined already.")
            Return False
        End Function

        Public Function resolve(ByVal name As String, ByRef o As class_def) As Boolean
            Return m.find(name_with_namespace.of(name), o)
        End Function
    End Class

```

The code generator for `class` then uses both of the above to combine the member variables ([service/compiler/b2style/rewriter/nodes/class.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/class.vb)):

```vb.net
Protected Overrides Function dump(ByVal n As typed_node, ByRef s As String) As Boolean
    Dim o As New StringBuilder()
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() >= 5)
    Dim class_name As String = n.child(1).input()
    Dim cd As New class_def(class_name)
    bstyle.struct.parse_struct_body(n).foreach(AddressOf cd.with_var)
    Dim classes As scope.class_proxy = scope.current().classes()
    If n.child(2).type_name.Equals("class-inheritance") AndAlso
       Not l.of_all_children(n.child(2).child(1)).
             dump().
             stream().
             with_index().
             map(Function(ByVal t As tuple(Of UInt32, String)) As Boolean
                     Dim bcd As class_def = Nothing
                     If Not classes.resolve(t.second(), bcd) Then
                         Return False
                     End If
                     cd.append(bcd).
                        with_var(bstyle.struct.create_id(t.second()))
                     Return True
                 End Function).
             aggregate(bool_stream.aggregators.all_true) Then
        Return False
    End If
    If Not scope.current().classes().define(class_name, cd) Then
        Return False
    End If
    ' Append struct-body back into the structure.
    o.Append("struct ").
      Append(class_name).
      Append("{")
    cd.vars().
       foreach(Sub(ByVal var As builders.parameter)
                   o.Append(var.type).
                     Append(" ").
                     Append(var.name).
                     Append(";")
               End Sub)
    o.Append("};")

```

The remaining changes are minor details and can be found in:

[class-inheritance syntax definition (#19) · Hzj-jie/osi.net@6e2ac2e](https://github.com/Hzj-jie/osi.net/commit/6e2ac2e05270a12be4108a763ba193b6038a97da)

[Merge class variables (#21) · Hzj-jie/osi.net@fe7eae2](https://github.com/Hzj-jie/osi.net/commit/fe7eae2efa232da1a7067bb424e593de2f08079c)

Finally, let's test this out, focusing on basic multiple inheritance and inheriting from template classes.

[service/tests/compiler/b2style/self-test/class-inheritance-base-variables.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/class-inheritance-base-variables.txt)

```cpp
#include <b2style.h>
#include <testing.h>

class A {
  int a;
};

class B {
  int b;
};

class C : B {
  int c;
};

class D : A, C {
  int d;
};

void main() {
  D d;
  d.a=1;
  d.b=2;
  d.c=3;
  d.d=4;
  // The "type-id"s are not tested.
  b2style::testing::assert_equal<int>(d.a, 1);
  b2style::testing::assert_equal<int>(d.b, 2);
  b2style::testing::assert_equal<int>(d.c, 3);
  b2style::testing::assert_equal<int>(d.d, 4);
  b2style::testing::finished();
}

```

[service/tests/compiler/b2style/self-test/class-inheritance-template-variables.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/class-inheritance-template-variables.txt)

```cpp
#include <b2style.h>
#include <testing.h>

template <T>
class B {
  T x;
};

class C : B<int> {
  int y;
};

class C2 : B<string> {
  int y;
};

void main() {
  C c;
  c.x= 1;
  c.y=2;

  C2 c2;
  c2.x="a";
  c2.y=3;

  b2style::testing::assert_equal<int>(c.x, 1);
  b2style::testing::assert_equal<int>(c.y, 2);
  b2style::testing::assert_equal<string>(c2.x, "a");
  b2style::testing::assert_equal<int>(c2.y, 3);
  b2style::testing::finished();
}

```

Notice that when merging an inherited base class, we also append the base class's ID variable. This is to facilitate calling base class methods later on and has no effect in this change yet:

```vb.net
cd.append(bcd).
   with_var(bstyle.struct.create_id(t.second()))

```
