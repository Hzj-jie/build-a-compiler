# Building a Programming Language from 0.1 - reinterpret_cast

Although we implemented classes earlier, the current implementation is a bit strange in that it does not support inheritance—resembling VB6 in some ways. Of course, the lack of support is simply because the prerequisites weren't ready yet, though we have been building them up step by step, such as templates in the previous article. Before all the pieces are integrated together, it might be hard to see how these granular features connect to class inheritance; the same applies to `reinterpret_cast` implemented here, which will become clear when we get to inheritance later.

First, let's look at what `reinterpret_cast` is. As you can roughly infer from its name, it is a type cast that instructs the compiler to `re-interpret` a variable's type. Without dwelling on the formal specification, let's highlight two key characteristics of this cast: 1) it modifies the compiler's state at the point where it is encountered during compilation, and 2) the cast itself generates no runtime code whatsoever. Take this example:

```cpp
#include <iostream>
using namespace std;
int main() {
    long long x = 100;
    double y = *reinterpret_cast<double*>(&x);
    cout << x << " " << y << endl;
    return 0;
}

```

Although the compiler does not allow `double y = reinterpret_cast<double>(x);`, pointer-to-pointer conversions are always permitted (though if member functions of a class are involved, subsequent behavior may be undefined). The example above converts a `long long*` into a `double*`. From that point on, the compiler treats `y` as a `double`, and all subsequent operations—such as function overload resolution—are based on this type. Thus, the output is:

```text
100 4.94066e-322

```

This feature is very useful for implementing class inheritance in `b2style`, so we can implement it in `bstyle`. Of course, given `bstyle`'s current feature set, this feature differs slightly from the one in C++.

First, because `bstyle` does not support pointers, applying `reinterpret_cast` to an instance of a type to produce a new value of another type would break function parameter references; therefore, `reinterpret_cast` in `bstyle` has no "return value" and mutates the variable's type in place.

Second, because `bstyle` does not have templates, the syntax `reinterpret_cast(variable, Type);` is more appropriate.

Nevertheless, the underlying principle and functionality are identical: this "function" changes the type of `variable` inside the compiler without generating any runtime code.

Let's recall the role of a variable's type during `bstyle` compilation. There are two main uses: 1) expanding structs, and 2) resolving the right function overload. Therefore, executing `reinterpret_cast` affects these two compiler behaviors. Both pieces of information are recorded in [scope.variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb), so aside from the syntax rules, the changes are concentrated in that file.

The changes to the `nlexer` and `syntaxer` rules are straightforward:

```text
kw-reinterpret-cast reinterpret_cast

reinterpret-cast kw-reinterpret-cast start-bracket variable-name comma type-name end-bracket
base-root-type [... reinterpret-cast-with-semi-colon]
base-sentence-with-semi-colon [... reinterpret-cast]? semi-colon

```

The [`code_gen<writer>` for `reinterpret_cast`](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/reinterpret_cast.vb) is also very simple, merely forwarding the two inputs to `scope.current().variables().redefine`:

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements code_gen(Of writer).build
    Return scope.current().variables().redefine(n.child(4).children_word_str(), n.child(2).children_word_str())
End Function

```

Now let's look at `scope.variable_t.redefine` and `scope.variable_proxy.redefine`. The former only handles the current scope, while the latter also searches parent scopes, since code can apply `reinterpret_cast` to variables defined in enclosing scopes.

`scope.variable_t.redefine` is simple—just a map lookup and update:

[service/compiler/bstyle/scope.variable.vb#L37](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb#L37)

```vb.net
Public Function redefine(ByVal type As String, ByVal name As String) As Boolean
    assert(Not type.null_or_whitespace())
    assert(Not name.null_or_whitespace())
    ' Types are always resolved during the define / build stage, so scope.current() equals to the scope
    ' where the variable_t instance Is being defined.
    type = scope.current().type_alias()(type)
    ' The name should not be an array with index.
    assert(Not variable.is_heap_name(name))
    If s.find(name) = s.end() Then
        Return False
    End If
    s(name) = type
    Return True
End Function

```

`scope.variable_proxy.redefine` is slightly more involved, as it also needs to check for heap variables:

[service/compiler/bstyle/scope.variable.vb#L77](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb#L77)

```vb.net
Public Function redefine(ByVal type As String, ByVal name As String) As Boolean
    If variable.is_heap_name(name) Then
        raise_error(error_type.user,
                    "Redefine works for heap name without index, but got ",
                    name,
                    " (new type ",
                    type,
                    ")")
        Return False
    End If
    Dim s As scope = Me.s
    While Not s Is Nothing
        If s.v.redefine(type, name) OrElse s.v.redefine(type, heap_name_of(name)) Then
            Return True
        End If
        s = s.parent
    End While
    raise_error(error_type.user,
                "Variable ",
                name,
                " (new type ",
                type,
                ") has not been defined yet.")
    Return False
End Function

```

There is currently a limitation here: a heap variable has only a single entry in `scope` because its length is dynamic. Therefore, we only allow changing the type of the entire heap array at once, rather than changing the type of a single element at a specific index. This seems reasonable enough, though supporting per-element redefinition wouldn't be impossible—just slightly more complicated. Consequently, this restriction is enforced in code rather than in the `syntaxer` grammar.

With that, the implementation is essentially complete. The full change is in [Implement reinterpret_cast in bstyle · Hzj-jie/osi.net@11bc2fd](https://github.com/Hzj-jie/osi.net/commit/11bc2fdb404bbbdfa7cf473b26a378c02bf3480b).

What remains is adding some tests. Let's walk through one of them in detail:

```cpp
#include <b2style.h>

class S {
  int x;
  void print() {
    b2style::std_out(this.x);
  }
  void f() {
    this.x = 0;
  }
};

class S2 {
  int x;
  void f() {
	this.x = 10;
  }
};

void main() {
  S s[1];
  s[0].x = -10;
  s[0].print();
  S2__struct__type__id__type s.S2__struct__type__id[1];  // 1
  // Note: this is not a typical way of using reinterpret_cast, it's pretty much useless without the support of
  // inheritance.
  reinterpret_cast(s, S2);  // 2
  s[0].f();  // 3
  reinterpret_cast(s, S);  // 4
  s[0].print();  // 5
}

```

The five key lines are marked with numbered comments:

1. This defines the type-ID member for `S2`. Briefly, to ensure that even empty structs can properly participate in function overload resolution, `bstyle` generates a uniquely named 1-byte type `S__struct__type__id__type` for each struct `S` and adds a member of this type named `S__struct__type__id` to the struct. Thus, a variable declared as type `S` only defines `S`'s struct type-ID variable in `logic`; to reinterpret it as type `S2`, we need to manually define `S2`'s type-ID variable in `logic` as well. This is a bit clunky, but since this is not how `reinterpret_cast` is normally used once inheritance is in place, a comment suffices. The earlier change introducing struct type IDs is in [osi.net/commit/d6bad6917912ea4c424d764149f539a0d75a8d09](https://github.com/Hzj-jie/osi.net/commit/d6bad6917912ea4c424d764149f539a0d75a8d09).
2. Now that `s.S2__struct__type__id` exists, `s` can be reinterpreted and used as type `S2`.
3. Call `S2`'s `f()` method on `s[0]`—this is the most important step.
4. Cast `s` back to type `S`.
5. Call `S`'s `print()` method on `s[0]`.

As a result, the output of execution is `"-1010"` rather than `"-100"`. It is already starting to feel like polymorphism!

Note that this implementation does not check whether two types are layout-compatible for casting; for structs, it simply relies on whether the flattened member variables of the target struct exist in `logic`.
