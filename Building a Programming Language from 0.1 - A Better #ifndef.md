# Building a Programming Language from 0.1 - A Better #ifndef

[Zijie He: Building a Programming Language from 0.1 - #include <stdio.h>](https://zhuanlan.zhihu.com/p/414895989) introduced a basic `#ifndef` / `#define` / `#endif` mechanism to prevent including the same header file multiple times. In the implementation, these keywords are still treated as regular "statements" rather than a separate "preprocessor" pass.

While implementing `b3style`, we run into an issue. For example, [service/compiler/b2style/lib/b2style/stdio.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/stdio.h) contains the following function:

```cpp
namespace b2style {
void std_out(string i) {
  logic "interrupt stdout b2style__i @@prefixes@temps@string";
}
}  // namespace b2style
```

This code is simple: it invokes the `stdout` interrupt on the input parameter `i`. However, it relies on an internal detail of `b2style`: because this function resides inside the `b2style` namespace, when it is transpiled into `bstyle` statements, every variable automatically gets prefixed with `b2style__`. Thus, in `bstyle`, this function becomes:

```cpp
void b2style__std_out(string b2style__i) {
  logic "interrupt stdout b2style__i @@prefixes@temps@string";
}
```

which makes it a valid `bstyle` function.

Because a `logic` statement directly emits the following string literal into the generated `logic` code, the `logic` code will fail to compile if the variable names do not match. That is why `interrupt stdout b2style__i @@prefixes@temps@string` has to use `b2style__i`.

However, the `b3style` compiler translates source code directly into `logic` code, making namespace handling much simpler—at the very least, there is no need to prepend awkward `__` prefixes. The implementation details are quite fine-grained and can mainly be found in [commit/3e7825eb850e6bbb7283744d8152fd0a14aa9c2e](https://github.com/Hzj-jie/osi.net/commit/3e7825eb850e6bbb7283744d8152fd0a14aa9c2e). At the same time, the `bstyle__` prefixes in `bstyle`'s standard library can also be removed ([commit/01817bed651037f4a4bd56143aa2ce71fa4881a5](https://github.com/Hzj-jie/osi.net/commit/01817bed651037f4a4bd56143aa2ce71fa4881a5)). This creates a discrepancy between the inline `logic` statements needed by `b2style` and `b3style`.

Even so, the vast majority of standard library code does not need inline `logic` statements, and in principle `b3style` can reuse almost all of `b2style`'s standard library. Therefore, we need a way for `b2style` and `b3style` to selectively compile the parts of the standard library they each require.

The simplest approach is naturally to extend `#ifndef`, which breaks down into roughly three parts:

1. Although not strictly required, having `#ifdef` is much nicer than having to write `#ifndef ... #else ... #endif` everywhere.
2. `#else`, for inverse selection.
3. Logical "OR" across multiple conditions.

```cpp
// Logical "AND" can be achieved by nesting multiple #ifdef directives, so it is not strictly necessary.
#ifdef A
#ifdef B
#define A_AND_B
#endif
#endif

#ifdef A_AND_B
// ...
#endif

// Alternatively, #else can be used for inverse selection.
#ifndef A || B
#else
  // #ifdef A && B
#endif
```

Unlike regular `if-else` statements, these conditions are evaluated at compile time, so adding logical "AND" or more complex constructs like parentheses would significantly increase compiler complexity—which is also why we keep the syntax minimal.

With this in place, each compiler can provide predefined macros—for instance, `B2STYLE` in `b2style` and `B3STYLE` in `b3style`. Standard libraries or any user code can then use these macros to supply compiler-specific implementations.

## Lexical Changes

In addition to adding `#ifdef` and `#else`, we also need to move `or \|\|` into [service/compiler/shared/nlexer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/shared/nlexer_rule.txt).

```text
+ kw-ifdef #ifdef
+ kw-delse #else
  kw-ifndef #ifndef
  kw-define #define
  kw-endif #endif
  @@ -48,6 +50,7 @@ start-square-bracket \[
  end-square-bracket \]
  semi-colon ;
  assignment =
+ or \|\|
```

## Syntax Changes

This change also fixes a previous bug. When checking whether a definition existed, the compiler always assumed it was a single token ("word"), whereas the syntax rule defined it as a `name`, which could span multiple tokens:

```vb.net
If is_defined(n.child(1).word().str()) Then
```

So it should be changed to `raw-name`.

[service/compiler/shared/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/shared/syntaxer_rule.txt)

```text
- ifndef-wrapped kw-ifndef name [root-type]* kw-endif
- define kw-define name
+ define-with-separator raw-name or
+ defines [define-with-separator]* raw-name
+ delse-wrapped kw-delse [root-type]*
+ ifdef-wrapped kw-ifdef defines [root-type]* [delse-wrapped]? kw-endif
+ ifndef-wrapped kw-ifndef defines [root-type]* [delse-wrapped]? kw-endif
+ define kw-define raw-name
- base-root-type [root-type-with-semi-colon, function, include, define, ifndef-wrapped, struct]
+ base-root-type [root-type-with-semi-colon, function, include, define, ifdef-wrapped, ifndef-wrapped, struct]
```

## Code Generator Changes

Now we implement the three features above: `#ifdef`, `#else`, and multiple definitions. The core code is not much different from before: [service/compiler/scope/scope.define.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope/scope.define.vb)

```vb.net
        Private NotInheritable Class if_wrapped
            Implements code_gen(Of WRITER)

...

            Private Function build(ByVal n As typed_node,
                                   ByVal o As WRITER) As Boolean Implements code_gen(Of WRITER).build
                assert(Not n Is Nothing)
                assert(n.child_count() >= 3)
                For i As UInt32 = 0 To n.child(1).child_count() - uint32_1
                    Dim s As String
                    If i = n.child(1).child_count() - uint32_1 Then
                        s = n.child(1).last_child().word().str()
                    Else
                        s = n.child(1).child(i).child(0).word().str()
                    End If
                    If Not bypass(s) Then
                        For j As UInt32 = 2 To n.child_count() - uint32_2
                            If Not code_gen_of(n.child(j)).build(o) Then
                                Return False
                            End If
                        Next
                        Return True
                    End If
                Next
                If n.child(n.child_count() - uint32_2).type_name.Equals("delse-wrapped") Then
                    For j As UInt32 = uint32_1 To n.child(n.child_count() - uint32_2).child_count() - uint32_1
                        If Not code_gen_of(n.child(n.child_count() - uint32_2).child(j)).build(o) Then
                            Return False
                        End If
                    Next
                    Return True
                End If
                Return True
            End Function
        End Class
```

The first loop checks multiple definitions. Because only logical "OR" is supported, we do not even need to inspect the separator token in between—the syntax rules have already validated it.

Then, if the `#ifdef` / `#ifndef` condition above was not satisfied, the `If` block checks whether an `#else` branch (`delse-wrapped`) is present and generates code for it if so.

Finally, we register this new `#ifdef` handler in `bstyle`, `b2style`, and `b3style`. For example, in [service/compiler/bstyle/bstyle.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/bstyle.vb):

```vb.net
+ with(scope.define_t.code_gens.ifdef_wrapped(AddressOf code_gen_of)).
  with(scope.define_t.code_gens.ifndef_wrapped(AddressOf code_gen_of)). 
```

## Adding Compiler Predefinitions

Each compiler can add predefined symbols to its root `scope`. For example, in [service/compiler/bstyle/scope.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.vb):

```text
  Public Sub New()
      Me.New(Nothing)
      incs = New includes_t()
      fc = New call_hierarchy_t()
      d = New define_t()
      f = New function_t()
      t = New temp_logic_name_t()

+     defines().define("BSTYLE")
  End Sub
```

Of course, we could also add "feature" definitions—for example, `b2style` could define `BSTYLE_SUPPORTED` to indicate that although it is not `bstyle`, it supports `bstyle` syntax. However, there is no use case for that yet, so it was not added.

## Testing

The main tests are located in [service/tests/compiler/b2style/self-test](https://github.com/Hzj-jie/osi.net/tree/master/service/tests/compiler/b2style/self-test), and `bstyle` also has a standalone test in [service/tests/compiler/bstyle/codegen/predefined-def.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/codegen/predefined-def.txt):

```cpp
#include <bstyle.h>

#ifdef BSTYLE
void main() {}
#endif
```

For example, [service/tests/compiler/b2style/self-test/ifdef-else-ifdef.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/ifdef-else-ifdef.txt):

```cpp
#include <b2style.h>
#include <testing.h>

#ifdef NO
void main() {
  b2style::testing::assert_equal<bool>(__STATEMENT__, true, false);
}
#else #ifdef NO_AGAIN
void main() {
  b2style::testing::assert_equal<bool>(__STATEMENT__, false, true);
}
#else
void main() {
  b2style::testing::finished();
}
#endif
#endif
```

And [service/tests/compiler/b2style/self-test/multi-defines.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/multi-defines.txt):

```cpp
#include <b2style.h>
#include <testing.h>

#define YES
#define YES2

#ifndef YES || NO
void f() {
}
#endif

#ifdef YES2 || NO
void main() {
  f();
  b2style::testing::finished();
}
#endif
```

These are a few representative test cases.

The main changes can be found in [commit/446df3799dc1e4b768d245b7312fc7722f4ba170](https://github.com/Hzj-jie/osi.net/commit/446df3799dc1e4b768d245b7312fc7722f4ba170).
