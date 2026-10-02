# Building a Programming Language from 0.1 - b3style_b2style_self_test

A PR that took over seven months to land: [Implement b3style_b2style_self_test with skipped tests](https://github.com/Hzj-jie/osi.net/pull/103/files).

Although implementation details have evolved, `self_test` is a testing paradigm already established in `b2style`: [Building a Programming Language from 0.1 - b2style_self_test](https://zhuanlan.zhihu.com/p/462594228). In short, test files are bundled together into a single tar archive, which are then compiled and executed sequentially: [service/tests/compiler/compiler_self_test_runner.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/compiler_self_test_runner.vb).

There are two primary categories of tests:
1. Tests that mimic comments such as `// ERROR: ` and `// ERROR PATTERN: ` to verify compile-time errors: [service/tests/compiler/compiler_self_compile_error_test_runner.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/compiler_self_compile_error_test_runner.vb)

```cpp
// ERROR PATTERN: *Type *S of s does not match the rvalue *Integer*
// b3style supports namespace, the output would be "::S of s" and "Integer" instead.

#include <b2style.h>

struct S {
  int x;
};

void main() {
  int x = 100;
  S s = x;
}
```

2. Tests that verify execution output against stdout assertions: [service/tests/compiler/compiler_self_assertion_test_runner.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/compiler_self_assertion_test_runner.vb)

```cpp
#include <b2style.h>
#include <testing.h>

class C {
  int x;

  void add(C that) {
    this.x += that.x;
  }
};

void main() {
  C c;
  c.x = 10;
  C d;
  d.x = 20;
  c.add(d);

  b2style::testing::assert_equal<int>(c.x, 30);
  b2style::testing::assert_equal<int>(d.x, 20);
  b2style::testing::finished();
}
```

The latter relies on libraries like [service/compiler/b2style/lib/testing/assert.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/testing/assert.h):

```cpp
#ifndef B2STYLE_LIB_TESTING_ASSERT_H
#define B2STYLE_LIB_TESTING_ASSERT_H

#include <b2style/stdio.h>
#include <b2style/to_str.h>
#include <bstyle.h>
#include <bstyle/str.h>
#include <testing/types.h>

namespace b2style {
namespace testing {

int _assertion_count = 0;

// TODO: Find a better way to test the __FILE__ in included files.
string assert__FILE__() {
  return __FILE__;
}

void assert_true(bool v, string msg) {
  _assertion_count++;
  string prefix;
  if (v) {
    prefix = "Success: ";
  } else {
    prefix = "Failure: ";
  }
  ::b2style::std_out(prefix);
  ::b2style::std_out(msg);
  ::b2style::std_out("\n");
}

void assert_true(bool v) {
  assert_true(v, "no extra information.");
}

void assert_false(bool v, string msg) {
  assert_true(!v, msg);
}

void assert_false(bool v) {
  assert_true(!v);
}

template <T, T2>
void assert_equal(T t, T2 t2) {
  assert_true(t == t2, ::str_concat(::b2style::to_str(t), " != ", ::b2style::to_str(t2)));
}

template <T>
void assert_equal(T t, T t2) {
  assert_equal<T, T>(t, t2);
}

}  // namespace testing
}  // namespace b2style

#endif  // B2STYLE_LIB_TESTING_ASSERT_H
```

The main benefit of this setup is reducing boilerplate and co-locating the test logic with its expected output in the same file, which is far more intuitive than traditional UTT runner cases:

```vb.net
    <test>
    Private Sub case1()
        Dim io As New console_io.test_wrapper()
        Dim e As executor = Nothing
        assertion.is_true(parse(io, _b2style_test_data.case1.as_text(), e))
        assertion.is_not_null(e)
        e.assert_execute_without_errors()
        assertion.equal(io.output(), "Hello World")
    End Sub
```

To enable `b2style_self_test` under `b3style` (i.e. `b3style_b2style_self_test`), implementing template functions was mandatory. Rather than copying and pasting code, let us examine the differences.

`b3style` is a combination of `bstyle` and `b2style`: it adopts `b2style` syntax, but compiles down to `logic` code like `bstyle`. In `logic` code, a callee always specifies a return value—callers may choose to discard it, but the `logic` runtime allocates stack slots accordingly. Thus, code generation requires distinguishing between `function_call.build` and `function_call.without_return`:

[service/compiler/logic/callee.vb#L52](https://github.com/Hzj-jie/osi.net/blob/858973ad033229a4b75bbef13fad213455e231df/service/compiler/logic/callee.vb#L52):

```text
while_then not-finished {
    subtract i i 1
    caller copy-one-byte1 ( source i 1 )
    more not-finished i 0
}

copy i length
more not-finished i 0
while_then not-finished {
    subtract i i 1
    define result type*
    caller copy-one-byte2 result ( source target2 i 1 )
    move target2 result
    more not-finished i 0
}
```

In `b2style`, this distinction does not exist because whether a return value is received is defined syntactically when translated into `bstyle`. Furthermore, `b3style/function_call_with_template` was adapted via `sed` from `b2style/function_call_with_template`. This prompted what might be the most crucial engineering improvement of the PR: adding `without_return` support to `b3style/function_call_with_template` without breaking `b2style/function_call_with_template`.

```diff
@@ -12,24 +12,16 @@ Partial Public NotInheritable Class b2style
     Partial Private NotInheritable Class function_call_with_template
         Implements code_gen(Of typed_node_writer)

-        Private Function build(ByVal n As typed_node, ByVal o As typed_node_writer) As Boolean _
+        Private Function build(ByVal n As typed_node, ByVal o As typed_node_writer) As Boolean _
                 Implements code_gen(Of typed_node_writer).build
-            assert(Not n Is Nothing)
+            Return build(n, o, AddressOf function_call.build)
+        End Function
+
+        Private Shared Function build(
+                ByVal n As typed_node,
+                ByVal o As typed_node_writer,
+                ByVal function_call_build As Func(Of String, typed_node, typed_node_writer, Boolean)) As Boolean
+            assert(Not n Is Nothing)
+            assert(Not function_call_build Is Nothing)
             Dim t As tuple(Of String, String) = Nothing
             If Not b2style.function_call.split_struct_function(scope.function_name.of(n.child(0).child(0)), t) Then
                 t = Nothing
             End If
             Dim extended_type As String = Nothing
             Return scope.template_t.resolve(n.child(0), extended_type) AndAlso
-                   function_call.build(scope.namespace_t.fully_qualified_name(
+                   function_call_build(scope.namespace_t.fully_qualified_name(
                                             If(t.is_null(),
                                                extended_type,
                                                b2style.function_call.build_struct_function(t.first(), extended_type))),
```

By leveraging partial classes, `b2style` and `b3style` share the bulk of template resolution logic, while `function_call_build` allows parameterizing whether the resulting `caller` instruction captures or discards the return value. `b3style` then easily generates template calls without returns: [service/compiler/b3style/logic/nodes/function_call_with_template_without_return.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/function_call_with_template_without_return.vb):

```vb.net
Partial Public NotInheritable Class b3style
    Partial Private NotInheritable Class function_call_with_template
        Public Shared Function without_return(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
            Return build(n, o, AddressOf function_call.with_parameters.without_return)
        End Function
    End Class
End Class
```

This enables registering `ignore-result-function-call-with-template` in `b3style`: [service/compiler/b3style/b3style.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/b3style.vb):

```diff
+                with_delegate("ignore-result-function-call-with-template",
+                              Function(ByVal n As typed_node, ByVal o As logic_writer) As Boolean
+                                  assert(Not n Is Nothing)
+                                  Return function_call_with_template.without_return(n.child(), o)
+                              End Function). 
```

The rest of the changes were largely refactoring optimizations, such as replacing `String` with `builders.parameter`.

Following these updates, `b3style_b2style_self_test` successfully ran 34 tests, with a small remaining set skipped:

```text
+    Protected Overrides Function ignore_case(ByVal name As String) As Boolean
+        Dim r As Boolean = False
+        ' Expect return value of ::type0@kw-return - should fix first.
+        r = r OrElse unordered_set.emplace_of("linked-list.txt").find(name).is_not_end()
+        ' Missing heap-struct-name - assertion failure
+        r = r OrElse unordered_set.of("static_cast_ptr_type3.txt",
+                                      "static_cast_ptr_type_bool.txt",
+                                      "static_cast_ptr_type.txt").find(name).is_not_end()
+        ' template class is not supported yet.
+        r = r OrElse unordered_set.emplace_of("bool-heap.txt",
+                                              "delegate-functions-with-same-name.txt",
+                                              "delegate_in_class_on_heap.txt",
+                                              "delegate_in_class.txt",
+                                              "heap_ptr_test.txt",
+                                              "heap.txt",
+                                              "include__FILE__.txt",
+                                              "more-closing-angle-brackets.txt",
+                                              "non-overridable-class-template-function.txt",
+                                              "ptr_offset.txt",
+                                              "ref_test.txt",
+                                              "static_cast_ptr_type2.txt",
+                                              "struct-and-primitive-type-with-same-name.txt",
+                                              "template_before_type.txt",
+                                              "template_class_with_ref.txt",
+                                              "template-in-namespace.txt",
+                                              "template-override.txt",
+                                              "two-closing-angle-brackets.txt",
+                                              "unmanaged_heap.txt",
+                                              "vector-test.txt").find(name).is_not_end()
+        ' expected compilation failure.
+        r = r OrElse unordered_set.emplace_of("define-class-constructor-for-non-class.txt").find(name).is_not_end()
+        Return r
+    End Function
```

The skipped tests fall into four buckets:
1. `linked-list.txt`: A minor return type bug (resolved in a subsequent update).
2. Missing `heap-struct-name`.
3. Template classes not yet supported.
4. Valid compile errors that `b3style` detects but `b2style` missed.

For example, [service/tests/compiler/b2style/self-test/define-class-constructor-for-non-class.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/define-class-constructor-for-non-class.txt):

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
  b2style::testing::assert_equal<int>(x, 101);
  b2style::testing::assert_equal<int>(y, 2);
  b2style::testing::finished();
}
```

Because `b2style` naively transformed constructor syntax into calls to `construct` and `destruct`, this test passed in `b2style`. In `b3style`, stricter validation ensures that constructors are only valid on declared classes, properly failing compilation:

[service/tests/compiler/b3style/b3style_compile_error_test.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b3style/b3style_compile_error_test.vb):

```vb.net
    <test>
    Private Sub define_class_constructor_for_non_class()
        run(_b3style_test_data.errors_define_class_constructor_for_non_class,
            b3style.class_initializer.failed_to_build_constructor_message("int", "x"))
    End Sub
```
