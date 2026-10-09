# Building a Programming Language from 0.1 - Template Functions

This feature is not strictly required for class inheritance, but since we have already implemented templates and extended them to function pointers, extending them further to functions is a good way to see if there are any rough edges in the template implementation. (And indeed there were—for example, previous versions had issues with namespace handling. Namespaces are quite tricky, with lots of subtle details that are easy to get wrong.)

The grammar rules are simple: just attach `less-than`, `paramtypelist`, and `greater-than` to `function`. However, a small structural optimization helps minimize code changes. [service/compiler/b2style/codegen/syntaxer_rule.txt#L21](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/codegen/syntaxer_rule.txt#L21)

```text
function-name-with-template name less-than paramtypelist greater-than
function-call-with-template function-name-with-template start-bracket [value-list]? end-bracket
ignore-result-function-call-with-template function-call-with-template
b2style-sentence-with-semi-colon [..., ignore-result-function-call-with-template, ...]? semi-colon

raw-value [heap-struct-function-call, function-call-with-template, base-value-without-bracket]

template-body [class, delegate-with-semi-colon, function]

```

Here we use `function-name-with-template` so that its child nodes have the exact same order as `template-type-name`, allowing them to share processing logic. During template instantiation, just like `class`, the second child node of `function` is its name:

```text
template <T>
class C {}

template <T>
void f() {}

```

[service/compiler/b2style/template_template.vb#L95](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/template_template.vb#L95)

```vb.net
Public Shared Function [of](ByVal l As code_gens(Of typed_node_writer),
                            ByVal n As typed_node,
                            ByRef o As template_template) As Boolean
    assert(Not n Is Nothing)
    assert(n.type_name.Equals("template"))
    assert(n.child_count() = 5)
    Dim name_node As typed_node = n.child(4).child()
    If name_node.type_name.Equals("class") OrElse name_node.type_name.Equals("function") Then
        name_node = name_node.child(1)
    ElseIf name_node.type_name.Equals("delegate-with-semi-colon") Then
        name_node = name_node.child(0).child(2)
    Else
        assert(False)
    End If

```

All that remains is `function_call_with_template` itself: [service/compiler/b2style/rewriter/nodes/function_call_with_template.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/function_call_with_template.vb).

The principle here is identical to class templates: expand the template at the point where it is referenced.

```cpp
template <T>
void f(T i) { }

void main() {
  f<int>(1);  // Expand f__int(int i) at the location where template f is defined
}

```

Consequently, the implementation is essentially the same as well: [osi.net/function_call_with_template.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/function_call_with_template.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As typed_node_writer) As Boolean _
        Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    Dim extended_type As String = Nothing
    If Not scope.current().template().resolve(l, n.child(0), extended_type) Then
        Return False
    End If
    Return l.typed(Of function_call).build(extended_type, n, o)
End Function

```

At this point, the core implementation is largely complete—except that namespaces are not handled properly. There are two issues here: 1) looking up a template name should include its namespace (for instance, in the example above, `f<int>` and `::f<int>` are the same); and 2) the expanded implementation itself must reside in the same namespace where the template was defined. Both of these had flaws in the initial implementation. While the code above is mostly in [Implement template function (#15) · Hzj-jie/osi.net@1fc23d0](https://github.com/Hzj-jie/osi.net/commit/1fc23d05b551145f34d2b786edfc88eb44373dd0), that commit also includes some rough-but-passable namespace workarounds and a test case: [service/tests/compiler/b2style/self-test/function-template.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/function-template.txt)

```cpp
#include <b2style.h>
#include <testing.h>

template <T>
T add(T& x, T y) {
  x += y;
  return x;
}

void main() {
  int x = 1;
  b2style::testing::assert_true(add<int>(x, 1) == 2);
  add<int>(x, 1);
  b2style::testing::assert_true(x == 3);
  b2style::testing::finished();
}

```

[templates are now working in namespace (#16) · Hzj-jie/osi.net@6d5c702](https://github.com/Hzj-jie/osi.net/commit/6d5c70268f760c60b6ef6fe20ddf53bf87b5c3c2) and [osi.net/commit/d19ea1c095b418ae59fe20a21f92041e349b0dd1](https://github.com/Hzj-jie/osi.net/commit/d19ea1c095b418ae59fe20a21f92041e349b0dd1) properly resolve both issues along those two axes: attaching the namespace when naming a template, and restoring the namespace when expanding and recompiling the template body. The second commit uses a `precompiler`-generated `podw` (plain-old-data-wrapper) to create a `namespace_with_name` backed by `tuple<String, String>`, making naming and comparisons much clearer. Accordingly, when `template_body_reparser` in `scope.template` runs `parse`, it uses `scope.current().current_namespace().define` to enter the template's namespace.

[service/compiler/b2style/scope.template.vb#L29](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/scope.template.vb#L29)

```vb.net
Protected Overrides Function wrapper(ByVal n As typed_node) As IDisposable
    ' The definition should be searched already in resolve.
    Dim name As name_with_namespace = template_name(n)
    Dim d As definition = +m.find_opt(name)
    assert(Not d Is Nothing)
    If name.namespace().empty_or_whitespace() Then
        Return MyBase.wrapper(n)
    End If
    Return scope.current().current_namespace().define(name.namespace())
End Function 

```

Full-featured template functions in standard languages include many additional capabilities—such as overload resolution, automatic template argument deduction, and so on—none of which are implemented here yet:

```text
template <T> void f(T i) {}
void f(int i) {}

template <T> void g(int i, T j) {}
template <T, T2> void g(T i, T2 j) {}

int main() {
  f(100);  // f(int i)
  f(1.1);  // f<double>(1.1)
  g(1, 1);  // g<int>(1, 1)
  g(1, 1.1);  // g<int, double>(1, 1.1)
}

```
