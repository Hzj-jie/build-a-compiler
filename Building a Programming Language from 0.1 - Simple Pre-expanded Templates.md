# Building a Programming Language from 0.1 - Simple Pre-expanded Templates

Templates go by different names across various languages—for example, `generics` in Java and C#, and `templates` in C++ and D. From the user's perspective, there doesn't seem to be much difference aside from syntax. However, it is worth distinguishing between the two main implementation approaches: one is called **type erasure**, and the other is called **type expansion** (monomorphization).

As the names suggest, in a type erasure implementation, template type parameters do not exist inside the compiled template itself; in a type expansion implementation, template type parameters do exist in the concrete implementation of each instantiated type. This explains why type-erased generics require type constraints (bounds), whereas type-expanded templates cannot be easily compiled into standalone binary packages. Putting it this way is still quite abstract, so let's look at an example.

## Type Erasure

```java
// java
public class MyClass {
    interface I {
      void print();
    }
    
    static class C<T extends I> {
      void print(T t) { t.print(); }
    }
    
    static class D implements I {
      public void print() { System.out.println("D"); }
    }
    
    public static void main(String args[]) {
        C<D> c = new C<>();
        D d = new D();
        c.print(d);
    }
}

```

Here, `class C<T extends I>` is essentially equivalent to:

```java
static class C {
  void print(I i) { i.print(); }
}

```

Of course, there are additional implementation details. For instance, C#'s type constraints support limited "non-type" constraints such as `where T: new()`, and methods like `T getter()` can return a reference of type `T` rather than `I`. C# also has slightly more explicit support for variance, using the `in` and `out` keywords to indicate whether supertypes or subtypes of a generic type parameter can be used in specific positions, whereas Java uses `extends` and `super` (wildcards) to let users control this constraint. Despite these differences in detail, the compiler can simply compile `class C<D extends I>` above into a package for `class C`, accompanied by a small amount of metadata—such as whether `D` has a `new()` constraint.

## Type Expansion

```cpp
// c++
#include <iostream>
template <typename T>
class C {
  public: void print(const T& t) { t.print(); }
};

class D {
  public: void print() const { std::cout << "D" << std::endl; }
};

int main() {
  C<D>().print(D());
}

```

Here, `typename T` in `C` does not need any "constraint" to call `T`'s `print()` method. If `D2` in `C<D2>` does not have a `print()` method, compilation will fail. Thus, `C<D>` here can essentially be thought of as:

```cpp
class C_D {
  public: void print(const D& t) { t.print(); }
};

int main() {
  C_D c;
  D d;
  c.print(d);
}

```

If another specialization `C<D2>` is referenced somewhere, the compiler generates `class C_D2`, and `void print(const D2& t) { t.print(); }` allows the compiler to directly check whether `D2::print` exists.

Consequently, `template <typename T> class C<T>` does not exist in the compiler's output at all, cannot be compiled into a standalone binary package, and cannot be instantiated by the compiler from a header declaration alone. That is why, for instance, we write `#include <vector>` rather than `#include <vector.h>` (with a separate precompiled library).

Although type constraints are no longer required, type-expanded template implementations have obvious drawbacks. Aside from not being compilable into standalone packages, the same template code is duplicated many times: every distinct type instantiation of a template generates an independent copy of the code, which significantly impacts compilation speed and generated code size. On the other hand, the advantage is zero runtime performance overhead: the expanded template class can directly call any method of the template type parameter without looking up any runtime type information.

Since `b2style` is a C++-like language, it implements C++-style type-expanded templates. Full C++ templates are extraordinarily complex—for example, a template's "type parameter" doesn't even have to be a type, but can be a value of a type (non-type template parameter):

```cpp
#include <iostream>
template <int I>
class C {
  public: void print() { std::cout << I << std::endl; }
};

int main() {
  C<1>().print();
  C<2>().print();
}

```

Or template template parameters and variadic templates:

```cpp
#include <iostream>
#include <set>
#include <vector>
template <template <typename...> typename T>
class C {
 public: template <typename... D> void print(const T<D...>& t) { std::cout << t.size() << std::endl; }
};

int main() {
  C<std::set>().print(std::set<int>());
  C<std::set>().print(std::set<long>());
  C<std::vector>().print(std::vector<int>());
}

```

These features are far too complex for `b2style` and not particularly necessary. Our simplified implementation supports up to the following example, which includes template type expansion and nested templates. This example is itself a test case ([service/tests/compiler/b2style/codegen/nested_template.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/nested_template.txt)):

```cpp
#include <b2style.h>

class C {
  void print() {
    b2style::std_out("C");
  }
};

template <T>
class D {
  void print(T t) {
    t.print();
  }
};

template <T, T2>
class E {
  void print(T t, T2 t2) {
    t.print(t2);
  }
};

void main() {
  C c;
  D<C> d;
  E<D<C>, C> e;
  e.print(d, c);
}

```

Before starting the implementation, there were several miscellaneous preparatory changes aimed at separating `name` from `type-name`, mainly in:

[Implement type-name in shared/syntaxer_rule.txt · Hzj-jie/osi.net@603b3b1](https://github.com/Hzj-jie/osi.net/commit/603b3b1dd5a9aa046f63c2bc932ae4c1428a5978)

[Move name and type-name into syntaxer · Hzj-jie/osi.net@2528732](https://github.com/Hzj-jie/osi.net/commit/2528732b27678903752c932b360df793d3c0d565)

Next come the `nlexer` and `syntaxer` rule files. In `nlexer`, there is only one new keyword, `template`, while in `syntaxer` we need to cover both template definitions and template type usages:

```text
raw-type-name [double-colon]? [raw-name-with-double-colon]* raw-name
template-type-param-list [template-type-param-with-comma]* template-type-param
template-type-param-with-comma template-type-param comma
template-type-param raw-type-name
template-type-name raw-type-name less-than template-type-param-list greater-than
type-name [template-type-name, raw-type-name]

root-type [..., class, template]
type-param-list [type-param-with-comma]* type-param
type-param-with-comma type-param comma
type-param raw-name
template kw-template less-than type-param-list greater-than class

```

This was the first version of the implementation, which did not yet support nested templates, so `template-type-param` was defined as `raw-type-name`; in the subsequent version, it was changed to `template-type-param type-name`.

The code is naturally divided into two parts as well: recording template definitions, and expanding templates.

## Template Definition

This uses [template_template.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/template_template.vb)—a slightly tongue-twisting name, but it really is a "template of a template." This class has two basic functions:

```vb.net
' Construct an instance from a template typed_node
Public Shared Function [of](ByVal n As typed_node, ByRef o As template_template) As Boolean

' Expand a class implementation using concrete type arguments
Public Function apply(ByVal types As vector(Of String), ByRef impl As String) As Boolean

```

Essentially, it replaces every `raw-type-name` in the `typed_node` that matches a template type parameter with a placeholder, and then substitutes the placeholders with the corresponding concrete types when `apply` is called. The rest consists of implementation details, such as checking that the number of type arguments matches, and using `class_name_t` and `ref<string>` to avoid repeated string lookups. This process uses depth-first search (`typed_node::dfs`) to automatically traverse and locate all `raw-type-name` nodes:

```vb.net
Private ReadOnly w As New template_writer()
Private ReadOnly class_name As class_name_t
Private ReadOnly types As New vector(Of ref(Of String))()

Private Sub New(ByVal n As typed_node, ByVal types as vector(Of String))
' ...
    n.dfs(Sub(ByVal node As typed_node, ByVal stop_navigating_sub_nodes As Action)
              assert(Not node Is Nothing)
              assert(Not stop_navigating_sub_nodes Is Nothing)
              If Object.ReferenceEquals(n.child(1), node) Then
                  ' type-name of class.
                  assert(w.append(class_name))
                  stop_navigating_sub_nodes()
                  Return
              End If
              If Not node.type_name.Equals("raw-type-name") Then
                  Return
              End If

              For i As UInt32 = 0 To types.size() - uint32_1
                  If node.input().Equals(types(i)) Then
                      assert(w.append(Me.types(i)))
                      stop_navigating_sub_nodes()
                      Return
                  End If
              Next
          End Sub,
          Sub(ByVal leaf As typed_node)
              assert(Not leaf Is Nothing)
              assert(w.append(leaf.word().str()))
          End Sub)
End Sub

```

## Recording Template Definitions in `b2style.template`

Similar to the implementation of `class`, `b2style`'s `scope.template_t` records all templates processed so far. Each `scope` has its own instance of `template_t`, so template definitions respect lexical scope.

Because the core logic is already implemented in `template_template`, `scope.template_t` is relatively simple and just uses [reparser](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/reparser.vb) to compile the generated `class` string once more through the `class` rules.

```vb.net
Private NotInheritable Class class_reparser
    Inherits code_gens(Of typed_node_writer).reparser(Of b2style.parser)

    Private ReadOnly m As unordered_map(Of String, definition)

    Public Sub New(ByVal m As unordered_map(Of String, definition))
        assert(Not m Is Nothing)
        Me.m = m
    End Sub

    Protected Overrides Function dump(ByVal n As typed_node, ByRef s As String) As Boolean
        assert(Not n Is Nothing)
        assert(n.child_count() = 4)
        Dim name As String = template_type_name_name(n)
        Dim d As definition = Nothing
        ' The definition should be searched already in resolve.
        assert(m.find(name, d))
        assert(Not d Is Nothing)
        Dim types As vector(Of String) = template_type_name_types(n)
        ' TODO: Should resolve type-aliases.
        If Not d.injected_types.emplace(types).second() Then
            ' Injected already.
            s = empty_string
            Return True
        End If
        Return d.template.apply(types, s)
    End Function
End Class

```

The concept of `reparser` here is simply: "the current `b2style` code generator outputs code that is still `b2style` code, and then invokes `b2style` again to compile that generated code."

## Template Expansion

When processing a `template_type_name` node, the corresponding `code_gen` injects the expanded class implementation into the code and replaces the template specialization syntax with the mangled name of the newly generated class—for example, replacing `C<D>` with `C_D`.

```vb.net
 Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not o Is Nothing)
    Dim extended_type As String = Nothing
    If Not scope.current().template().resolve(n, extended_type) Then
        Return False
    End If
    o.append(extended_type)
    Return True
End Function

```

Those are the main changes, which can be found in [Implement template in b2style · Hzj-jie/osi.net@a91679d](https://github.com/Hzj-jie/osi.net/commit/a91679d412bde7899f71bedcf1c425f76aea10e9).

## Supporting Nested Templates

As mentioned earlier, the initial implementation above did not support nested templates, so a moderate follow-up change was made to support nesting. The key change lies in the timing of template expansion, allowing `scope.template_t.resolve` to recursively call itself when resolving inner template arguments:

```vb.net
Private Shared Function template_type_name_types(ByVal n As typed_node,
                                                 ByRef o As vector(Of String)) As Boolean
    assert(Not n Is Nothing)
    assert(n.child_count() = 4)
    o.renew()
    Dim v As vector(Of String) = o
    Return streams.range(0, n.child(2).child_count()).
                   map(Function(ByVal index As Int32) As typed_node
                           Return n.child(2).child(CUInt(index))
                       End Function).
                   map(Function(ByVal tn As typed_node) As typed_node
                           If tn.type_name.Equals("template-type-param-with-comma") Then
                               Return tn.child(0)
                           End If
                           Return tn
                       End Function).
                   map(Function(ByVal tn As typed_node) As Boolean
                           assert(tn.type_name.Equals("template-type-param"))
                           tn = tn.child().child()
                           If tn.type_name.Equals("raw-type-name") Then
                               v.emplace_back(tn.children_word_str())
                               Return True
                           End If
                           Dim r As String = Nothing
                           If Not scope.current().template().resolve(tn, r) Then
                               Return False
                           End If
                           v.emplace_back(r)
                           Return True
                       End Function).
                   aggregate(bool_stream.aggregators.all_true)
End Function

```

These changes are mainly in [Allow nested template · Hzj-jie/osi.net@0ef52aa](https://github.com/Hzj-jie/osi.net/commit/0ef52aadb364cc14398d6940489c3e3cc8c77f11).

Notice that because this template implementation is essentially simple string substitution, supporting C++-style non-type template arguments (such as an `int` constant as a template parameter) would not be complicated either—the underlying principle is identical.

There is one minor issue with the current implementation: because `typedef` is implemented in `bstyle`, `b2style` does not know that `C<int>` and `C<Integer>` are actually the same type, so it generates two copies of the code. Unless [scope.type_alias](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.type_alias.vb) is also implemented in `b2style`, this issue cannot be resolved. Of course, it has no functional impact other than increasing the size of the generated code and the compilation time.
