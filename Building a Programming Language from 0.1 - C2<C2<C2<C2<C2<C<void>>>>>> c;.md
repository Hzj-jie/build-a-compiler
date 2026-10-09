# Building a Programming Language from 0.1 - C2<C2<C2<C2<C2<C<void>>>>>> c;

There was a quirky issue in C++03 where `C<C2<T>>` was a syntax error and had to be written as `C<C2<T> >`—note that magical space.

As mentioned in [Zijie He: Building a Programming Language from 0.1 - i++++ and set<set<T>>, i>>1](https://zhuanlan.zhihu.com/p/103972204), this is caused by maximal munch in lexical analysis, and the solution is to move `>>` from lexical analysis to syntax analysis. Because syntax analysis operates top-down with a high-level view of the grammar, it can determine from the surrounding context whether `>>` should be interpreted as a right-shift operator or the closing brackets of a template type. (Of course, why `<>` was chosen to denote templates in the first place is another question entirely.)

Accordingly, [osi.net/commit/d380a6b04f23f5ce7c2a41e3543d5e84c879a4a1](https://github.com/Hzj-jie/osi.net/commit/d380a6b04f23f5ce7c2a41e3543d5e84c879a4a1) implements this change and supports writing nested templates without spaces.

```cpp
#include <b2style.h>
#include <testing.h>

template <T>
class C {
  void f() {
    b2style::testing::assert_true(true);
  }
};

template <T>
class C2 {
  void f() {
    T t;
	t.f();
  }
};

void main() {
  C2<C2<C2<C2<C2<C<void>>>>>> c;
  c.f();
  b2style::testing::finished();
}

```

This example uses `b2style_self_test`, which we will cover next; in short, it allows tests to be defined directly in the source code, simplifying the process of writing tests for new features.
