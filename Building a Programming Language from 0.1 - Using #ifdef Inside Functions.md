# Building a Programming Language from 0.1 - Using #ifdef Inside Functions

Strictly speaking, the title is somewhat imprecise: the required change is to allow `#ifdef` or similar macro-like statements to wrap individual statements, rather than being restricted to classes or functions. They are referred to here as "macro-like statements" because `#ifdef` is fundamentally a statement in itself—`bstyle`, `b2style`, and `b3style` handle these constructs in a single compilation pass without requiring an external C/C++ preprocessing phase.

Previously, usage of `#ifdef` or similar directives looked like this:

```cpp
#ifdef B3STYLE
string _str_mid(string s, int i, int l) {
  string r;
  logic "cut_len r s i l";
  return r;
}
#else
string _str_mid(string s, int i, int l) {
  string r;
  logic "cut_len b2style__r b2style__s b2style__i b2style__l";
  return r;
}
#endif

string str_mid(string s, int i, int l) {
  ::assert(__STATEMENT__ + "@" + __FILE__, (i >= 0) && (l >= 0));
  return _str_mid(s, i, l);
}
```

While not terrible, duplicating entire function signatures and definitions serves little purpose. The ideal syntax should be:

```cpp
string str_mid(string s, int i, int l) {
  ::assert(__STATEMENT__ + "@" + __FILE__, (i >= 0) && (l >= 0));
  string r;
#ifdef B3STYLE
  logic "cut_len r s i l";
#else #ifdef B2STYLE
  logic "cut_len b2style__r b2style__s b2style__i b2style__l";
#else
  // TODO: Trigger an #error.
#endif
#endif
  return r;
}
```

This clearly eliminates substantial boilerplate.

The processing of `#ifdef` statements does not inherently care what kind of constructs are wrapped; it simply delegates them to other statement code generators: [service/compiler/scope/scope.define.vb#L109](https://github.com/Hzj-jie/osi.net/blob/8b75a8979d2d9ff3f7934da3184c02e46bfb20ca/service/compiler/scope/scope.define.vb#L109).

```vb.net
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
            For j As UInt32 = 2 To n.child_count() - uint32_3
                If Not code_gen_of(n.child(j)).build(o) Then
                    Return False
                End If
            Next
            Return n.child(n.child_count() - uint32_2).type_name.Equals("delse-wrapped") OrElse
                   code_gen_of(n.child(n.child_count() - uint32_2)).build(o)
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
```

Therefore, the only change needed was at the grammar level to support statements inside `#ifdef`:

```diff
--- a/service/compiler/shared/syntaxer_rule.txt
+++ b/service/compiler/shared/syntaxer_rule.txt
@@ -10,7 +10,7 @@ param-with-comma param comma
 paragraph [sentence, multi-sentence-paragraph]
 sentence [sentence-with-semi-colon, condition, while, for-loop]
 
-multi-sentence-paragraph start-paragraph [paragraph]* end-paragraph
+multi-sentence-paragraph start-paragraph [paragraph, ifdef-wrapped, ifndef-wrapped]* end-paragraph
 
 value-definition type-name name assignment value
 value-declaration type-name name
@@ -47,9 +47,9 @@ include [include-with-string, include-with-file]
 
 define-with-separator raw-name or
 defines [define-with-separator]* raw-name
-delse-wrapped kw-delse [root-type]*
-ifdef-wrapped kw-ifdef defines [root-type]* [delse-wrapped]? kw-endif
-ifndef-wrapped kw-ifndef defines [root-type]* [delse-wrapped]? kw-endif
+delse-wrapped kw-delse [root-type, paragraph]*
+ifdef-wrapped kw-ifdef defines [root-type, paragraph]* [delse-wrapped]? kw-endif
+ifndef-wrapped kw-ifndef defines [root-type, paragraph]* [delse-wrapped]? kw-endif
 define kw-define raw-name
```

In short, `#ifdef` blocks can now contain `paragraph`s, and `paragraph`s can contain `#ifdef` constructs.

To test this: [service/tests/compiler/b2style/self-test/ifdef-sentence.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/ifdef-sentence.txt):

```cpp
#include <b2style.h>
#include <testing.h>

void main() {
#ifdef NO
  b2style::testing::assert_true(false);
#else
  b2style::testing::assert_true(true);
#endif

#ifndef NO
  b2style::testing::assert_true(true);
#else
  b2style::testing::assert_true(false);
#endif

#ifdef NO {
  b2style::testing::assert_true(false);
} #else {
  b2style::testing::assert_true(true);
} #endif

#ifndef NO {
  b2style::testing::assert_true(true);
} #else {
  b2style::testing::assert_true(false);
} #endif

#ifdef B2STYLE || B3STYLE
  b2style::testing::finished();
#endif
}
```

With this change, code duplication in the `b2style` library for supporting `b3style` can be drastically reduced: [commit/7e5e3e730aaa23af1af7a830119c36c29d5597c2](https://github.com/Hzj-jie/osi.net/commit/7e5e3e730aaa23af1af7a830119c36c29d5597c2).
