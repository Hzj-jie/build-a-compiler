# Building a Programming Language from 0.1 - Linking and Recursion in the logic Language

[Building a Programming Language from 0.1 - Anchors, Scope, and Types in the logic Language](https://zhuanlan.zhihu.com/p/102868596) mentioned that the `logic` language lacks a linking phase, which means syntax like the following cannot be supported:

```cpp
int g(int);

int f(int x) {
  return g(x + 1);
}

int g(int x) {
  if (x > 100) {
    return x;
  }
  return f(x + 1);
}
```

However, a recursive call such as the following does not require a linking phase:

```cpp
int f(int x) {
  if (x < 100) {
    return f(x + 1);
  }
  return 100;
}
```

As explained previously regarding anchors—setting aside the term "linking" itself—the purpose of two-pass compilation is to determine jump targets when generating the final binary based on the actual location of functions, while setting placeholders for each function call during compilation based on declarations. While header files are a C concept and other languages might not have header files, this does not affect the core concept of two-pass compilation. After all, a compiler reads input linearly; in the first example, when parsing `g(x + 1)`, the compiler has no knowledge of where the function might eventually be located, only knowing that such a declaration exists, and thus cannot generate the jump.

The second example with recursion is different. When reaching `return f(x + 1);`, the entry location of function `f` is already fixed. Therefore, the only determining factor for whether a recursive function can be compiled is *when* the function is defined.

Although C syntax is used here for ease of explanation, this logic is actually implemented inside the `logic` language: [service/compiler/logic/callee.vb](https://github.com/Hzj-jie/osi.net/blob/a77b1da81bef9613d00a780728875c0aed58031f/service/compiler/logic/callee.vb#L45). If the function is defined in `anchors` before the function body is compiled, recursive calls can be supported. If those three lines were moved after `End Using`, recursive calls would fail to compile.

A test was implemented in [commit/a77b1da81bef9613d00a780728875c0aed58031f](https://github.com/Hzj-jie/osi.net/commit/a77b1da81bef9613d00a780728875c0aed58031f) using the exact recursive function above:

```cpp
#include <b2style.h>
#include <testing.h>

int f(int x) {
  if (x < 100) {
    return f(x + 1);
  }
  return 100;
}

void main() {
  b2style::testing::assert_equal<int, int>(f(0), 100);
  b2style::testing::finished();
}
```

Because this functionality is independent of bstyle/b2style/b3style, even though b3style is not yet fully implemented, this test already passes under `b3style_b2style_b3style_self_test` (where b3style replaces bstyle, compiling the code that compiles b2style into bstyle down to the logic language).
