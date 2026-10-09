# Building a Programming Language from 0.1 - T...

Even though `b2style` supports combining `delegate` and `template`, without variadic templates, functions with different numbers of parameters still require separate `delegate` template definitions. First, let's look at this change: [Allow templates with same name but different count of types · Hzj-jie/osi.net@b7de9b0](https://github.com/Hzj-jie/osi.net/commit/b7de9b0fea014a7b9b489f5ab3df38c651def251).

The key part is this function in [osi.net/template_template.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/template_template.vb#L83):

```vb.net
Public Shared Function template_name(ByVal n As typed_node, ByVal type_count As UInt32) As String
    assert(Not n Is Nothing)
    Return strcat(n.children_word_str(), "__", type_count)
End Function

```

By appending the number of type parameters to the template name, we allow templates with the same name but different parameter counts. This makes it possible to write:

```text
template <RT>
delegate RT function();

template <RT, T>
delegate RT function(T);

template <RT, T, T2>
delegate RT function(T, T2);

```

Still, this is somewhat inelegant: since there is no limit on the number of function parameters, we would have to keep adding more template overloads endlessly—much like `Func` and `Action` in .NET, which grew from 4 parameters to 10, and then to 16, with no real end in sight.

Variadic template parameters are quite different from variadic function arguments: the latter can be implemented using arrays or by pushing a variable number of arguments onto the stack (which is less safe). Variadic template parameters are much safer, since they are purely a compile-time concept. Implementing them is not particularly difficult—in fact, a working implementation already exists, even if the details are still a bit rough around the edges: [var-template-param by Hzj-jie · Pull Request #9 · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/pull/9/files).

In terms of usage, however, variadic template parameters require substantial additional syntax support. In `b2style`, they can only be used as parameter lists in a `delegate`, because a variadic parameter pack isn't useful anywhere else:

```cpp
template<T...>
delegate T... function();  // Invalid: a function cannot have more than one return type.

template<T...>
class C {
  T... t;  // Also invalid: a single variable cannot have multiple types.
  void f(T... t) { }  // Naturally invalid as well: variadic function arguments are not supported.
};

template<T...>
delegate void function(T...);  // Well, only this one works.

```

The same restrictions exist in C++ as well:

```text
// Huh? This is invalid
template <typename T1, typename... T>
T... f(T1 x) {
  return 0;
}

// Huh? Yet this is valid
template <typename T1, typename... T>
T1 f(T... x) {
  return 0;
}

```

Given all these restrictions, why not just build the `function` template directly into `b2style`—and if we did that, what would we even need `delegate` templates for?

Viewed in that light, implementing variadic template parameters on top of the current foundation doesn't seem very worthwhile.
