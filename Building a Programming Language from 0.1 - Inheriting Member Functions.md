# Building a Programming Language from 0.1 - Inheriting Member Functions

Previously, we implemented:

[Zijie He: Building a Programming Language from 0.1 - Delegates and Their Templates](https://zhuanlan.zhihu.com/p/462684519)

[Zijie He: Building a Programming Language from 0.1 - Class Constructors and Destructors](https://zhuanlan.zhihu.com/p/463428191)

[Zijie He: Building a Programming Language from 0.1 - Inheriting Class Member Variables](https://zhuanlan.zhihu.com/p/466550826)

In theory, inheriting member functions should just be the final step. Unfortunately, our current implementation has a major obstacle: struct flattening. Let's look at a simple example first:

```cpp
struct S {
  int x;
  string y;
};
void f(S s) {}

```

Here, `void f(S s)` is flattened into `void f(int s.x, string s.y)` (in terms of a `callee` in the `logic` language, though that conveys the general idea).

With class inheritance, an overridden function in a derived class must have the same signature as in the base class. This creates an awkward problem: even though we can use function pointers to let a base class call a derived class's method, the derived class's implementation cannot access its own newly added member variables because they are not in the flattened parameter list! Thus, even with function pointers, we cannot implement C++-style polymorphic inheritance unless we replace struct-flattening parameter passing with true pointers or address references.

Before doing that, however, let's step back and consider what function inheritance is actually used for. It serves two main purposes:

1) Extending a class's functionality

This is very similar to inheriting member variables: adding new capabilities on top of a base class. For example, an animal can eat, and a cat is an animal that can also meow. So `Animal` is the base class with an `eat` function, and `Cat` is a derived class of `Animal` that adds a `meow` function.

2) Allowing a base class pointer/reference to invoke derived class methods

This is common in interface-like types, or when a base class relies on a derived class to supply specific behavior—for instance, a `Logger` class with `error`, `warn`, `info`, and `debug` methods that all rely on a `write` method implemented by a subclass to output messages.

The first purpose is unaffected by the struct-flattening limitation above: methods added by a derived class can only be called on the derived class, as the base class knows nothing about them. Only the second purpose runs into trouble.

So the question is: is the second mechanism—virtual functions via function pointers or vtables—truly indispensable? Under our current design, does it provide anything that cannot be achieved by other means?

Take a look at the following example, which compiles and runs using the features we have already implemented: [service/tests/compiler/b2style/self-test/template-override.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/template-override.txt)

```cpp
#include <b2style.h>
#include <testing.h>

class OutWriter {
  int c;

  void write(string msg) {
    this.c++;
    b2style::std_err(msg);
  }
};

class ErrWriter {
  int c;

  void write(string msg) {
    this.c++;
    b2style::std_err(msg);
  }
};

template <W>
class Logger {
  W w;

  void log(string msg) {
    this.w.write(msg);
  }

  int count_of_log_lines() {
    return this.w.c;
  }
};

typedef Logger<OutWriter> OutLogger;
typedef Logger<ErrWriter> ErrLogger;

void main() {
  OutLogger ol;
  ErrLogger el;

  ol.log("out");
  ol.log("out");
  ol.log("out");
  el.log("err");

  b2style::testing::assert_equal<int>(ol.count_of_log_lines(), 3);
  b2style::testing::assert_equal<int>(el.count_of_log_lines(), 1);
  b2style::testing::finished();
}

```

It seems the desired functionality can already be achieved! Of course, this idea is nothing new: [templates_vs_inheritance.html](http://groups.di.unipi.it/~nids/docs/templates_vs_inheritance.html), an article from June 2014, discusses this exact pattern—and the concept itself has likely been around ever since templates were introduced in C++ 3.0 back in 1991 ([history-of-c-templates-from-c-style-macros-to-concepts](https://belaycpp.com/2021/10/01/history-of-c-templates-from-c-style-macros-to-concepts/#:~:text=Templates%20were%20introduced%20in%20Release,the%20language%2C%20in%20October%201991)).

When comparing the two, people often say "inheritance is dynamic" while "templates are static." That distinction is somewhat tenuous, though: nothing in a program is truly dynamic since all code must be written ahead of time; "dynamic" dispatch just means the compiler generates extra machinery so the program can select among pre-written implementations based on runtime state. The real difference comes back to template expansion, which we discussed earlier: virtual functions allow new implementations to be added by linking against a separate library, whereas templates must always be expanded and recompiled from source.

In the current `b2style` implementation, that limitation doesn't matter at all, because the `primitive` processor does not support linking external libraries anyway—everything is compiled into a single self-contained package.

With that realization, the problem becomes much simpler: virtual dispatch (the second purpose) is not needed for now, so we only need to support the first purpose—extending a class's functionality.

This is relatively straightforward. First, on the syntax side, we want explicit markers indicating whether a function can be overridden. Early C++ only had the `virtual` keyword, making it easy to accidentally shadow or fail to override the intended base method. Here we keep it simple with two keywords, `overridable` and `override`: a function that may be overridden is marked `overridable`, and a function that overrides it is marked `override`.

- In practice, the current implementation is very basic and does not perform strict `overridable` checks yet; adding the keywords now just makes future checks easier to bolt on.
- [Allow overridable and override function (#23) · Hzj-jie/osi.net@91e9ae8](https://github.com/Hzj-jie/osi.net/commit/91e9ae82502f11bc8da947237e1bd0c939dd7a26#diff-6960944a90c02d86ebf7f4c6ab89668ec5838b4ac7256e4db41d6c59916ac72e) also contained many other changes that were later removed; only the changes to `syntaxer_rule.txt` remain relevant.

Base class functions thus fall into two categories: those inherited directly, and those that can be overridden (though "overriding" here is closer to `Shadows` in VB.NET than to traditional virtual `override`). These correspond to the two function-inheritance helpers in `class_def` ([osi.net/class_def.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/class_def.vb)):

```vb.net
 Public Function inherit_from(ByVal other As class_def) As class_def
    assert(Not other Is Nothing)
    _vars.emplace_back(other._vars)
    inherit_non_overrides(other)
    inherit_overrides(other)
    Return Me
End Function

Private Function forward_to(ByVal other As class_def) As Func(Of function_def, function_def)
    assert(Not other Is Nothing)
    Return Function(ByVal f As function_def) As function_def
               assert(Not f Is Nothing)
               f = f.with_class(Me)
               scope.current().call_hierarchy().to(f.name().in_global_namespace())
               Return f.with_content(f.declaration() + "{" + f.forward_to(other) + "}")
           End Function
End Function

Private Sub inherit_non_overrides(ByVal other As class_def)
    assert(Not other Is Nothing)
    _funcs.emplace_back(other.funcs().
                              filter(Function(ByVal f As function_def) As Boolean
                                         assert(Not f Is Nothing)
                                         ' Never directly forward constructor and destructor.
                                         Return Not f.is_virtual() AndAlso
                                                Not f.name().name().Equals(construct) AndAlso
                                                Not f.name().name().Equals(destruct)
                                     End Function).
                              map(forward_to(other)).
                              collect_to(Of vector(Of function_def))())
End Sub

Private Sub inherit_overrides(ByVal other As class_def)
    assert(Not other Is Nothing)
    _funcs.emplace_back(other.funcs().
                              filter(Function(ByVal f As function_def) As Boolean
                                         assert(Not f Is Nothing)
                                         Return f.is_virtual()
                                     End Function).
                              except(funcs().filter(Function(ByVal f As function_def) As Boolean
                                                        assert(Not f Is Nothing)
                                                        Return f.is_override()
                                                    End Function)).
                              map(forward_to(other)).
                              collect_to(Of vector(Of function_def))())
End Sub

```

When a function is "inherited" by a derived class, the compiler generates a wrapper method of the same name on the derived class that forwards the call to the base class method, using `reinterpret_cast` (discussed earlier) to cast `this` to the base class type. When the derived class overrides the base class method, the derived class's own method is emitted directly without forwarding to the base class.

That is the gist of it. Aside from the commit mentioned above, the main implementation is in [A very basic shadow-like function inheritance (#28) · Hzj-jie/osi.net@44b9551](https://github.com/Hzj-jie/osi.net/commit/44b9551a0aeac59bddd1f0c88b75f6fe6058a003).

There was actually an earlier revision—[018516feca7d354e60ed9024f4dd14617d36c28b/service/compiler/b2style/class_def.vb](https://github.com/Hzj-jie/osi.net/blob/018516feca7d354e60ed9024f4dd14617d36c28b/service/compiler/b2style/class_def.vb)—that implemented conventional function-pointer-based virtual inheritance, but it was abandoned due to the inability to access derived class member variables mentioned earlier.

Other related changes along the way:

[Support using references in the reinterpret_cast](https://github.com/Hzj-jie/osi.net/commit/2d63370b67b52234cda37fc5f7defb0984dd965f) (#22), which allowed `reinterpret_cast(this, B&)`. This was later removed as it wasn't particularly useful.

[b2style_init function (#26)](https://github.com/Hzj-jie/osi.net/commit/bec2d5163a6271d7e73fe2502a8bbbacdb792a51#), which automatically inserted a call to a compiler-generated `b2style_init` function (used to initialize things like vtable pointers) inside class constructors; this was also removed later.
