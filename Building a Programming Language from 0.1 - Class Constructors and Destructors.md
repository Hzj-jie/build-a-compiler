# Building a Programming Language from 0.1 - Class Constructors and Destructors

Before we begin, let's take a look at the bizarre syntax surrounding constructors in C++:

```cpp
#include <iostream>

using namespace std;

class C {
 public:
  C() {
    cout << "C" << endl;
  }
};

int main()
{
    C c;  // 1
    C c2 = C();  // 2
    C c3{};  // 3
    C c4();  // 4
    c4();  // 5

    return 0;
}

C c4() {
  return C();
}

```

Lines 1, 2, and 3 all invoke the default constructor. Line 2 is particularly interesting because it does not invoke `operator=` at the same time; you can verify this by trying the following example:

```cpp
#include <iostream>

using namespace std;

class C {
 public:
  C() { cout << "C" << endl; }
  C& operator = (const C& c) { cout << "operator=" << endl; return *this; }
};

int main()
{
    C c{};
    C c2 = C();
    C c3;
    c3 = C();

    return 0;
}

```

Line 3 is the new uniform initialization syntax introduced in C++11, which also has variants like `C a = {}`. Line 4, however, is not a constructor call at all, but a function forward declaration; and line 5 calls the function declared in line 4.

Furthermore, if line 1 uses a defaulted (`=default`) constructor, its behavior changes again to `default-initialization`, which means the output of the following program under the standard may differ from what you would expect:

```cpp
#include <stdio.h>

int main()
{{
    int i = 100;
    printf("%d\n", i);
}{
    int i;
    printf("%d\n", i);
}{
    int i = int(0);
    printf("%d\n", i);
}
    return 0;
}

```

Outputting `100 100 0` is actually standard-conforming—for example, on early versions of Borland Turbo C++, or under certain compiler flags. See [C++ zero initialization - Why is `b` in this program uninitialized, but `a` is initialized?](https://stackoverflow.com/questions/54350114/c-zero-initialization-why-is-b-in-this-program-uninitialized-but-a-is-i) for details.

There are plenty more complex cases once POD types are involved, which we won't go into here; see [Do the parentheses after the type name make a difference with new?](https://stackoverflow.com/questions/620137/do-the-parentheses-after-the-type-name-make-a-difference-with-new).

Doesn't that make your head spin? The root cause of all this is that `C c4();` (case 4) does not call a constructor, which led to workarounds 2 and 3, while case 1 exists largely for C compatibility since C has no concept of constructors.

Let's start with what constructors and destructors actually are. Neither function is particularly magical: they are simply two functions automatically called by the language—one upon construction, and the other upon destruction. C++'s `operator new` complicates matters by bundling memory allocation and constructor invocation together. `operator delete` is similar, except it takes no extra arguments since destructors take no parameters. Roughly speaking, writing:

```cpp
C* c = new C(100);

```

is equivalent to:

```cpp
C* c = static_cast<C*>(malloc(sizeof(C)));
new (c) C(100);

```

The `(c)` in the second snippet is placement new, specifying the memory address at which to invoke the `C::C(int)` constructor—a syntactic requirement rather than a fundamental one. If we did away with special constructor syntax altogether and used an ordinary method named something like `init`, the example could be written as:

```text
C* c = new C();
c->init(100);
// Or
C* c = static_cast<C*>(malloc(sizeof(C)));
c->init(100);

```

Both become the exact same thing, and the same applies to destructors.

Returning to the current `b2style` implementation: both heap and stack allocations are automatic, and there are no `new` or `delete` operators. That means we simply need to invoke the constructor for initialization at `value-declaration` time and invoke the destructor when leaving the `scope`. To keep things simple, if a class does not define a constructor or destructor, we generate default ones for it.

Generating default constructors and destructors is relatively straightforward in [osi.net/class.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/class.vb):

```vb.net
Dim has_constructor As Boolean = False
Dim has_destructor As Boolean = False
streams.range(3, n.child_count() - uint32_2).
        map(class_body).
        filter(AddressOf is_function).
        foreach(Sub(ByVal node As typed_node)
                    assert(Not node Is Nothing)
                    assert(node.child_count() = 5 OrElse node.child_count() = 6)
                    ' No namespace is necessary, the first parameter contains namespace.
                    If node.child(1).input().Equals("construct") Then
                        has_constructor = True
                    ElseIf node.child(1).input().Equals("destruct") Then
                        has_destructor = True
                    End If
                    o.Append(node.child(0).input()).
                      Append(" ").
                      Append(_namespace.with_global_namespace(node.child(1).input())).
                      Append("(").
                      Append(class_name).
                      Append("& this")
                    ' With parameter list.
                    If node.child_count() = 6 Then
                        o.Append(", ").
                          Append(node.child(3).input())
                    End If
                    o.Append(")").
                      Append(node.last_child().input()).
                      AppendLine() ' beautiful output.
                End Sub)
If Not has_constructor Then
    o.Append("void construct(").Append(class_name).Append("& this){}")
End If
If Not has_destructor Then
    o.Append("void destruct(").Append(class_name).Append("& this){}")
End If

```

Constructors and destructors are simply specially named functions: `void construct(C& this, ...)` and `void destruct(C& this)`. Because `~` is not a valid character in function names, we don't use the `~ClassName()` syntax. Both functions also explicitly return `void` to avoid introducing unnecessary syntax rules.

Invoking them is handled by a new grammar rule, `class-initializer`. Because `b2style` has no concept of function forward declarations, we can safely use the syntax `C c(parameters)`. To keep concepts simple, `C c` does not invoke the constructor or destructor, whereas `C c(parameters)` invokes both the constructor and the destructor.

The grammar changes are:

```text
class-initializer type-name name start-bracket [value-list]? end-bracket
b2style-sentence-with-semi-colon [..., class-initializer]? semi-colon

```

And `class_initializer` simply rewrites the declaration + constructor call into two `bstyle` statements: [osi.net/class_initializer.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/class_initializer.vb)

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() = 4 OrElse n.child_count() = 5)
    Dim class_name As String = n.child(0).input()
    If Not scope.current().classes().resolve(class_name) Then
        Return False
    End If
    If Not (o.append(n.child(0)) AndAlso
            o.append(n.child(1)) AndAlso
            o.append(";") AndAlso
            o.append("construct(") AndAlso
            o.append(n.child(1)) AndAlso
            If(n.child_count() = 5, o.append(",") AndAlso o.append(n.child(3)), True) AndAlso
            o.append(");")) Then
        Return False
    End If
    scope.current().when_end_scope(Sub()
                                       assert(o.append("destruct(") AndAlso
                                              o.append(n.child(1)) AndAlso
                                              o.append(");"))
                                   End Sub)
    scope.current().call_hierarchy().to("construct")
    scope.current().call_hierarchy().to("destruct")
    Return True
End Function

```

The `scope.current().when_end_scope` call at the end registers an action to run when the current `scope` exits.

That covers the essentials; the main changes are in [osi.net/commit/d19ea1c095b418ae59fe20a21f92041e349b0dd1](https://github.com/Hzj-jie/osi.net/commit/d19ea1c095b418ae59fe20a21f92041e349b0dd1).

In this initial version, `class-initializer` was restricted to classes, though that restriction didn't really serve any purpose. It was removed in [Allow class-initializer to be used by non-class types · Hzj-jie/osi.net@0d9764c](https://github.com/Hzj-jie/osi.net/commit/0d9764cca935da68e225a736f7a0ac1e767a4af2), allowing custom constructors and destructors to be defined for non-class types simply by writing `void construct` and `void destruct`:

```cpp
#include <b2style.h>
#include <testing.h>

void construct(int& this, int v) {
  this = v + 1;
}

void construct(int& this) {
  this.construct(1);
}

void destruct(int& this) {}

void main() {
  int x(100);
  int y();
  // TODO: Why b2style::testing::assert_equal<int>(x, 101) does not work?
  b2style::testing::assert_equal<int, int>(x, 101);
  b2style::testing::assert_equal<int, int>(y, 2);
  b2style::testing::finished();
} 

```

Admittedly, this kind of behavior is rather unusual and rarely seen in practice.
