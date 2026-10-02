# Building a Programming Language from 0.1 - Implementing Operators in b3style

[Building a Programming Language from 0.1 - Purpose and Basic Architecture of the b2style Language](https://zhuanlan.zhihu.com/p/105230021) noted two significant differences between `b2style` and `bstyle`: namespaces and operators. While `b2style` also includes features such as classes and templates, operators—much like namespaces—deserve priority implementation.

## Support for function_call

As described in [Building a Programming Language from 0.1 - Supporting Operators](https://zhuanlan.zhihu.com/p/105513770), operators in `b3style` can also be implemented straightforwardly using functions. The difference is that `b3style` compiles code directly into `logic` statements rather than `bstyle` statements. This process closely mirrors the implementation of `function_call`. The distinction is that `function_call` obtains the function name from the first node and the parameter list in `value_list` from the third node (`paramlist`); operators, on the other hand, translate the operator itself into a function name, and the arguments are supplied by the expressions on either or both sides of the operator.

```vb.net
' Handling regular functions in function_call
        Private Shared Function build(ByVal n As typed_node,
                                      ByVal o As logic_writer,
                                      ByVal build_caller As Func(Of String, vector(Of String), Boolean),
                                      ByVal build_caller_ref As Func(Of String, vector(Of String), Boolean)) As Boolean
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            assert(n.child_count() >= 3)
            If n.child_count() = 3 Then
                value_list.with_empty()
            ElseIf Not code_gen_of(n.child(2)).build(o) Then
                Return False
            End If
            Return build(n.child(0).input_without_ignored(), build_caller, build_caller_ref)
        End Function
```

> As shown above, when a `function_call` has only three nodes (e.g. `func()`), `value_list` is empty (no arguments). Otherwise, `paramlist` populates `value_list`.

Notice that the core logic of `function_call` does not particularly care where `value_list` originates, as long as a `value_list` is provided. This allows call nesting to be handled cleanly via a stack. Therefore, a variant can be written as:

```vb.net
        Public Shared Function build(ByVal function_name As String, ByVal o As logic_writer) As Boolean
            Return build(function_name, caller_builder(o), caller_ref_builder(o))
        End Function 
```

Here, `value_list` is expected to be prepared by the caller. The primary modification to `function_call` was adding this overload, along with a `without_return` overload; the remaining changes were minor engineering details. The modified `function_call` is in [service/compiler/b3style/logic/nodes/function_call.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/function_call.vb), with the diff visible in [commit/fffb736ec9b8ee94cfb1fb2d0ecb3aad17dc97fa](https://github.com/Hzj-jie/osi.net/commit/fffb736ec9b8ee94cfb1fb2d0ecb3aad17dc97fa#diff-88d4ffb9e3b8bbafa67a4e104e0bbc1789931d9629266978c756e3936d07e6b3).

## Calling Functions Corresponding to Operators

As in `b2style`, operator operations fall into two categories: unary and binary. The former includes expressions like `!boolean_value`, while the latter includes `number1 + number2`, corresponding to `unary_operation_value` and `binary_operation_value`.

[service/compiler/b3style/logic/nodes/binary_operation_value.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/binary_operation_value.vb):

```vb.net
    Private NotInheritable Class binary_operation_value
        Implements code_gen(Of logic_writer)

        Public Shared Function operation_function_name(ByVal op As String) As String
            assert(Not op.null_or_whitespace())
            Return scope.current_namespace_t.in_global_namespace(
                           scope.current_namespace_t.with_namespace("b2style", op.Replace("-"c, "_"c)))
        End Function

        Public Shared Function build(ByVal left As typed_node,
                                     ByVal op As String,
                                     ByVal right As typed_node,
                                     ByVal o As logic_writer) As Boolean
            assert(Not left Is Nothing)
            assert(Not op.null_or_whitespace())
            assert(Not right Is Nothing)
            assert(Not o Is Nothing)
            value_list.build(Sub(ByVal a As Action(Of typed_node))
                                 assert(Not a Is Nothing)
                                 a(left)
                                 a(right)
                             End Sub,
                             o)
            Return function_call.build(operation_function_name(op), o)
        End Function

        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(n.child_count() = 3)
            Return build(n.child(0), n.child(1).type_name, n.child(2), o)
        End Function
```

This reflects an early implementation, but the invocation pattern is clear.

As in `b2style`, `operation_function_name` maps operator tokens to corresponding function names; for example, `&&` / `and` maps to `bool and(bool i, bool j)`:

[service/compiler/b2style/lib/b2style/operators.h#L11](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/operators.h#L11):

```cpp
#ifndef B2STYLE_LIB_B2STYLE_OPERATORS_H
#define B2STYLE_LIB_B2STYLE_OPERATORS_H

#include <b2style/types.h>
#include <bstyle/int.h>
#include <bstyle/str.h>

namespace b2style {

bool and(bool i, bool j) {
  if (i) return j;
  return false;
}

// ...
```

The `build` function invokes `value_list` to construct the argument list, then calls `function_call.build` to generate the `logic` instructions.

Similarly, `unary_operation_value` follows the exact same pattern, processing only one argument: [service/compiler/b3style/logic/nodes/unary_operation_value.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/unary_operation_value.vb):

```vb.net
        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            assert(n.child_count() = 2)
            Dim function_name As String =
                    binary_operation_value.operation_function_name(n.child(operator_index).type_name + suffix)
            value_list.build(Sub(ByVal a As Action(Of typed_node))
                                 assert(Not a Is Nothing)
                                 a(n.child(uint32_1 - operator_index))
                             End Sub,
                             o)
            Return function_call.build(function_name, o)
        End Function
```

## A Minor Difference in self_value_clause

Strictly speaking, `self_value_clause` (e.g. `i += 1;`) is an assignment statement rather than a value expression. However, it can still be processed similarly to binary operators by utilizing specialized functions that take references. This same approach was already used for prefix/postfix increment and decrement operators:

[service/compiler/b2style/lib/b2style/operators.h#L299](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/operators.h#L299):

```cpp
biguint self_inc_post(biguint& x) {
  biguint r = x;
  logic "add b2style__x b2style__x @@prefixes@constants@int_1";
  return r;
}

// ...
#ifdef B3STYLE

void self_and(bool& i, bool j) {
  i = and(i, j);
}
```

Even inside the `b2style` library, this code is specific to `b3style`, because the corresponding logic was originally handled directly within `b2style/self_value_clause`: [service/compiler/b2style/rewriter/nodes/self_value_clause.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/self_value_clause.vb):

```vb.net
    Private NotInheritable Class self_value_clause
        Implements code_gen(Of typed_node_writer)

        Private Const self_prefix As String = "self-"

        Private Function build(ByVal n As typed_node,
                               ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
            assert(n.child_count() = 3)
            If Not code_gen_of(n.child(0)).build(o) Then
                Return False
            End If
            Dim function_name As String = n.child(1).type_name
            assert(function_name.StartsWith(self_prefix))
            function_name = _namespace.bstyle_format.operator_function_name(
                                function_name.Substring(self_prefix.Length()))
            scope.current().call_hierarchy().to_bstyle_function(function_name)
            o.append("=")
            o.append(function_name)
            o.append("(")
            If Not code_gen_of(n.child(0)).build(o) Then
                Return False
            End If
            o.append(",")
            If Not code_gen_of(n.child(2)).build(o) Then
                Return False
            End If
            o.append(")")
            Return True
        End Function
```

Of course, `b2style` could also be transitioned to the exact same implementation.

Note that `self_and` returns `void`, because syntax like `i > (j += 1)` is disallowed. This necessitates `function_call.without_return` and its matching `binary_operation_value.without_return`:

```vb.net
    Private NotInheritable Class function_call
    ' ...
        Public Shared Function without_return(ByVal function_name As String, ByVal o As logic_writer) As Boolean
            Return build(function_name, without_return_caller_builder(o), without_return_caller_ref_builder(o))
        End Function

    Private NotInheritable Class binary_operation_value
    ' ...
        Public Shared Function without_return(ByVal n As typed_node,
                                              ByVal o As logic_writer) As Boolean
            Return build(n, AddressOf function_call.without_return, o)
        End Function

    Private NotInheritable Class self_value_clause
    ' ...
        Private Function build(ByVal n As typed_node,
                               ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            Return binary_operation_value.without_return(n, o)
        End Function
```

## Differences in logic Statements

[Building a Programming Language from 0.1 - A Better #ifndef](https://zhuanlan.zhihu.com/p/651984853) discussed the purpose of `#ifdef B3STYLE`. In short, `b2style` handles namespaces somewhat naively:

```cpp
namespace b2style {
void f(int i) {}
}  // namespace b2style
```

is simply translated into the `bstyle` statement:

```cpp
void b2style__f(b2style__int b2style__i) {}
```

In `b2style`, this is not an issue, but when embedding inline `logic` statements, variable names within those statements had to reference the `bstyle`-mangled names.

`b3style` handles namespaces much more robustly without this limitation, so variable names in `logic` statements remain their original identifier names. Consequently, functions in `b2style/operators.h` that use `logic` statements must supply specialized implementations for `b3style`:

[service/compiler/b2style/lib/b2style/operators.h#L73C1-L85C7](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/operators.h#L73C1-L85C7):

```cpp
#ifdef B2STYLE
bool greater_than(biguint i, biguint j) {
  bool result;
  logic "more b2style__result b2style__i b2style__j";
  return result;
}
#else
bool greater_than(biguint i, biguint j) {
  bool result;
  logic "more result i j";
  return result;
}
#endif
```

The vast majority of functions that do not embed `logic` statements require no such divergence. Additionally, unlike standard C++, `#ifdef` is not a preprocessor stage, so its branches can only enclose `root-type` constructs (such as a full function) rather than arbitrary statements.

## Testing

With these changes in place, `b3style` can run a significant portion of `b2style_test` (though `b2style_self_test` still requires template support). Supported tests are grouped under `b2style_test_b3style_supported`: [service/tests/compiler/b2style/b2style_test.vb#L18](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/b2style_test.vb#L18).

For example, shift operators:

```cpp
#include <b2style.h>
#include <b2style/stdio.h>

void main() {
  int i = 100;
  i <<= 2;
  b2style::std_out(i);
  int j = 0;
  j = i >> 5;
  b2style::std_out(i);
  b2style::std_out(j);
}
```

## Performance Improvements

Since `b3style` can directly execute parts of the `b2style` test suite, eliminating the intermediate pass of compiling `b2style` into `bstyle` yields substantial performance improvements—running roughly twice as fast:

```console
hzj-jie@hzj-jie-x1 - Sun Sep 24-23:37:53:~/git/osi.net/root/utt/bin/Release$ time ./osi.root.utt.exe b2style_test.* > /dev/null 2>&1 ; time ./osi.root.utt.exe b3style_b2style_test.* > /dev/null 2>&1

real    1m19.487s
user    0m0.001s
sys     0m0.013s

real    0m46.509s
user    0m0.000s
sys     0m0.015s
hzj-jie@hzj-jie-x1 - Sun Sep 24-23:40:30:~/git/osi.net/root/utt/bin/Release$ git diff
diff --git a/service/tests/compiler/b2style/b2style_test.vb b/service/tests/compiler/b2style/b2style_test.vb
index 81257f80..5e705e5d 100755
--- a/service/tests/compiler/b2style/b2style_test.vb
+++ b/service/tests/compiler/b2style/b2style_test.vb
@@ -749,7 +749,7 @@ End Class

 <test>
 Public NotInheritable Class b2style_test
-    Inherits b2style_test(Of _parse)
+    Inherits b2style_test_b3style_supported(Of _parse)

     Public NotInheritable Class _parse
         Inherits __do(Of console_io.test_wrapper, String, executor, Boolean)
```

The principal pull requests for these changes are:
- [commit/fffb736ec9b8ee94cfb1fb2d0ecb3aad17dc97fa](https://github.com/Hzj-jie/osi.net/commit/fffb736ec9b8ee94cfb1fb2d0ecb3aad17dc97fa)
- [commit/7da6f8040dc17ec73a183dd51763ec0c421841cd](https://github.com/Hzj-jie/osi.net/commit/7da6f8040dc17ec73a183dd51763ec0c421841cd)
