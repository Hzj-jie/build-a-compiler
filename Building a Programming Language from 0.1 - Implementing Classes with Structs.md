# Building a Programming Language from 0.1 - Implementing Classes with Structs

Now that [`struct` has been implemented in `bstyle`](https://zhuanlan.zhihu.com/p/430741003) and [parameter references have been implemented in `logic`](https://zhuanlan.zhihu.com/p/431336597), implementing classes in `b2style` is no longer complicated. From the language user's perspective, a class is a struct together with its member functions. From the language implementation perspective, a class also involves function parameter references and function overloading. Therefore, as long as a language supports structs, function parameter references, and function overloading, we can use that language to implement a language that supports classes. The `bstyle` language satisfies all of the requirements above.

First, let's look at an example, which is also a test case:

[service/tests/compiler/b2style/codegen/class.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/class.txt)

```cpp
#include <b2style.h>

class C {
  int x;

  void add_x(int y) {
    this.x += y;
  }

  int with_x(int y) {
    return this.x + y;
  }

  int return_x() {
    return this.x;
  }
};

void main() {
  C c;
  c.add_x(100);
  b2style::std_out(c.with_x(100));
  b2style::std_out(c.return_x());
}

```

Implementing this `class C` using a struct and free functions looks like this:

```cpp
#include <b2style.h>

struct C {
  int x;
};

void add_x(C& this, int y) {
  this.x += y;
}

int with_x(C& this, int y) {
  return this.x + y;
}

int return_x(C& this) {
  return this.x;
}

void main() {
  C c;
  add_x(c, 100);
  b2style::std_out(with_x(c, 100));
  b2style::std_out(return_x(c));
}

```

1. Convert the `class` into a `struct`, keeping only its member variables.
2. Convert the member functions of the `class` into ordinary functions by adding a reference parameter `this`.
3. Convert member function calls on a class instance into ordinary function calls with the struct instance as the first argument.

Note that because function overloading is already supported, even if multiple classes have member functions with the same name, the first parameter `this` guarantees that the right function is called. As we will see later, the namespaces of these functions can also be omitted based on the same rule.

Because the `logic` language only implements references and not pointers, `this` is not a pointer—and indeed, making `this` a pointer wouldn't make sense here. At the same time, any access to class members must be prefixed with `this.`. We could track member variables in `b2style`'s `scope` similarly to [bstyle/scope.variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb) to make `this.` optional, but since that is not critical, it was not implemented.

Adding lexical and syntactic rules

```text
# service/compiler/b2sytle/codegen/nlexer_rule.txt
  kw-struct struct
+ kw-class class
  dot .
  name [\w,_,::][\w,\d,_,::,.]*

# service/compiler/b2style/codegen/syntaxer_rule.txt
- root-type [... typedef-with-semi-colon, struct, heap-declaration-with-semi-colon]
+ root-type [... typedef-with-semi-colon, struct, class, heap-declaration-with-semi-colon]
  # TODO: Support value-definition
  struct kw-struct name start-paragraph [value-declaration-with-semi-colon]* end-paragraph semi-colon
+ class kw-class name start-paragraph [value-declaration-with-semi-colon, function]* end-paragraph semi-colon

```

Implementing the `class` rewriter

The `class` rewriter rewrites a `class` definition into a `struct` and several functions, distinguishing whether each child node of the class is a function or a variable declaration. [osi.net/class.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/class.vb)

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() >= 5)
    o.append("struct")
    Dim class_name As String = l.typed_code_gen(Of name)().name(n.child(1))
    o.append(class_name)
    o.append("{")
    ' Append variables into the structure.
    If Not streams.range(3, n.child_count() - uint32_2).
                   map(Function(ByVal i As Int32) As typed_node
                           Return n.child(CUInt(i))
                       End Function).
                   filter(AddressOf is_value_declaration).
                   map(Function(ByVal node As typed_node) As Boolean
                           Return l.of(node).build(o)
                       End Function).
                   aggregate(bool_stream.aggregators.all_true) Then
        Return False
    End If
    o.append("};")
    ' Append functions after the structure.
    If Not streams.range(3, n.child_count() - uint32_2).
                   map(Function(ByVal i As Int32) As typed_node
                           Return n.child(CUInt(i))
                       End Function).
                   filter(AddressOf is_function).
                   map(Function(ByVal node As typed_node) As Boolean
                           assert(Not node Is Nothing)
                           assert(node.child_count() = 5 OrElse node.child_count() = 6)
                           If Not l.of(node.child(0)).build(o) Then
                               Return False
                           End If
                           If Not l.of(node.child(1)).build(o) Then
                               Return False
                           End If
                           o.append("(").
                             append(class_name).
                             append("& this")
                           If node.child_count() = 6 Then
                               o.append(", ")
                               If Not l.of(node.child(3)).build(o) Then
                                   Return False
                               End If
                           End If
                           o.append(")")
                           If Not l.of(node.last_child()).build(o) Then
                               Return False
                           End If
                           Return True
                       End Function).
                   aggregate(bool_stream.aggregators.all_true) Then
        Return False
    End If
    Return True
End Function

```

Modifying the `function_call` rewriter

Function calls also need special handling when a method is invoked on a class instance. [osi.net/function_call.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/function_call.vb)

```vb.net
If n.child(0).word().str().Contains(".") Then
    Dim dot_pos As Int32 = n.child(0).word().str().LastIndexOf(".")
    ' dot is not allowed to be the first character.
    assert(dot_pos > 0)
    ' This is not handled by nlexer_rule.
    If dot_pos = n.child(0).word().str().Length() - 1 Then
        raise_error(error_type.user, "Name is not allowed to be suffixed by dot.")
        Return False
    End If
    o.append(n.child(0).word().str().Substring(dot_pos + 1)).
      append("(").
      append(n.child(0).word().str().Substring(0, dot_pos))
    If n.child_count() = 4 Then
        o.append(", ")
        If Not l.of(n.child(2)).build(o) Then
            Return False
        End If
    End If
    o.append(")")
    Return True
End If

```

The changes above are mainly in [osi.net/commit/f0ea5c8ec6a0b7c5c6da2f7faf10a9841c7c33a6](https://github.com/Hzj-jie/osi.net/commit/f0ea5c8ec6a0b7c5c6da2f7faf10a9841c7c33a6).

However, there is a problem with this: `b2style` supports namespaces, so any function inside a namespace is renamed to `Namespace__Function`. This does not match how member functions are called on class instances across namespaces. For example: [service/tests/compiler/b2style/codegen/class_in_namespace.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/class_in_namespace.txt)

```cpp
#include <b2style.h>

namespace N {

class C {
  ::int x;

  ::void print() {
    ::b2style::std_out(this.x);
  }
};

C c;

::void print() {
  c.x = 2;
  c.print();
}

}  // namespace N

```

Because functions support overloading and the first parameter of a class member function is the struct corresponding to the class itself (which already includes its namespace), these member functions do not need to be placed inside the enclosing namespace. [osi.net/commit/da7f1e081574ac4d2be487266ec398838e0b10c1](https://github.com/Hzj-jie/osi.net/commit/da7f1e081574ac4d2be487266ec398838e0b10c1)

```vb.net
' b2style/rewriter/nodes/class.vb
  If Not l.of(node.child(0)).build(o) Then
      Return False
  End If
- If Not l.of(node.child(1)).build(o) Then
-    Return False
- End If
- o.append("(").
+ ' No namespace is necessary, the first parameter contains namespace.
+ o.append(namespace_.bstyle_format_in_global_namespace(node.child(1).word().str())).
+   append("(").
+   append(class_name).
+   append("& this")
+   append("&").
+   append(namespace_.bstyle_format("this"))
  If node.child_count() = 6 Then
      o.append(", ")
      If Not l.of(node.child(3)).build(o) Then

' ----------------------------------

' b2style/rewriter/nodes/function_call.vb
  If dot_pos = name.Length() - 1 Then
      raise_error(error_type.user,
                  "Name [",
                  name,
                  "] is not allowed to be suffixed by dot.")
      Return False
  End If
- o.append(n.child(0).word().str().Substring(dot_pos + 1)).
-   append("(").
-   append(n.child(0).word().str().Substring(0, dot_pos))
+ o.append(namespace_.bstyle_format_in_global_namespace(name.Substring(dot_pos + 1))).
+   append("(").
+   append(namespace_.bstyle_format(name.Substring(0, dot_pos)))
  If n.child_count() = 4 Then
      o.append(", ")
      If Not l.of(n.child(2)).build(o) Then

```

This implementation also happens to support a feature that does not exist in C++:

```cpp
#include <b2style.h>

class C {
  int N::x;

  void N::print() {
    b2style::std_out(this.N::x);
  }
};

void main() {
  C c;
  c.N::x = 100;
  c.N::print();
  c.::N::x = 200;
  c.::N::print();
}

```

This test can be found in [support explicitly provide namespace in class variables and functions · Hzj-jie/osi.net@fd69d4d](https://github.com/Hzj-jie/osi.net/commit/fd69d4d713419b6f87ac9a2e8cd8c4847f632149).
