# Building a Programming Language from 0.1 - Function Parameter References in bstyle and b2style

Although in simple terms we only need to translate the function parameter reference symbol `&` into the `logic` language, there are actually several details to handle—such as lvalues and rvalues, which were mentioned earlier in [Building a Programming Language from 0.1 - Function Parameter References in logic](https://zhuanlan.zhihu.com/p/431336597).

First, let's consider what needs to be handled:

1. In the `nlexer` and `syntaxer` rules of `bstyle` and `b2style`, `param` needs to support the reference symbol.
2. `bstyle/logic/nodes/param` needs to support `logic` builders that use references.
3. When a function call argument is a variable, avoid copying it into a temporary variable; otherwise, the modified reference value would only be copied back to the temporary variable.
4. Modify the functions in `b2style/lib`—for example, distinguishing between `self_inc_post` and `self_inc_pre`, and making the input parameters of these functions references.
5. Add some tests.

Changes to lexical and syntactic rules

This part is straightforward. In `bstyle`, there was no `&` symbol at all, while `b2style` already had `bit-and`. Overall, it is very simple and needs no further elaboration.

```text
# bstyle/codegen/nlexer_rule.txt
  start-square-bracket \[
  end-square-bracket \]
  semi-colon ;
+ reference &

  assignment =

# bstyle/codegen/syntaxer_rule.txt
  function name name start-bracket [paramlist]? end-bracket multi-sentence-paragraph
  paramlist [param-with-comma]* param
  param-with-comma param comma
- param name name
+ param name [reference]? name

  paragraph [sentence, multi-sentence-paragraph]

# b2style/codegen/syntaxer_rule.txt
  function name name start-bracket [paramlist]? end-bracket multi-sentence-paragraph
  paramlist [param-with-comma]* param
  param-with-comma param comma
- param name name
+ param name [bit-and]? name

  namespace- kw-namespace name start-paragraph [root-type]* end-paragraph

```

Supporting the reference symbol in `bstyle/logic/nodes/param`

Following our consistent design, `bstyle` does not directly concatenate raw strings to generate `logic` code, because `logic` has no lexical analyzer and simple string concatenation is very error-prone. Therefore, we need to update the code in `param` and `logic/builders`.

[service/compiler/bstyle/logic/nodes/param.vb#L48-L54](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/param.vb#L48-L54)

```vb.net
Public NotInheritable Class param
' ...
    Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
        assert(Not n Is Nothing)
        assert(n.child_count() = 2)
        assert(Not o Is Nothing)
        assert(n.child_count() = 2 OrElse n.child_count() = 3)
        Dim params As vector(Of builders.parameter) = Nothing
        If Not scope.current().structs().resolve(n.child(0).word().str(), n.child(1).word().str(), params) Then
            params = vector.of(New builders.parameter(n.child(0).word().str(), n.child(1).word().str()))
        If Not scope.current().structs().resolve(n.child(0).word().str(), n.last_child().word().str(), params) Then
            params = vector.of(New builders.parameter(n.child(0).word().str(), n.last_child().word().str()))
        End If
' ------------------------------
        If n.child_count() = 3 Then
            assert(n.child(1).leaf())
            assert(n.child(1).type_name.Equals("reference"))
            params = params.stream().
                            map(AddressOf builders.parameter.to_ref).
                            collect(Of vector(Of builders.parameter))()
        End If
' ------------------------------
        rs.push(params)
        ' No parameter nesting expected, use read_scoped to reduce the cost of maintaining the state only.
        assert(rs.size() = 1)
        Return True
    End Function

```

The main newly added code is set off by separator lines. Essentially, when `param` contains a `reference`, the generated `builders.parameter` entries are converted into references. Here, `to_ref` is the corresponding change in `logic/builders`.

[service/compiler/logic/builder/builders.vb#L39-L58](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/builder/builders.vb#L39-L58)

```vb.net
Public NotInheritable Class builders
' ...
    Public Function logic_type() As String
        If ref Then
            Return strcat(type, type_ref_suffix)
        End If
        Return type
    End Function

    Public Shared Function of_ref(ByVal type As String, ByVal name As String) As parameter
        assert(Not type.null_or_whitespace())
        Return New parameter(strcat(type, type_ref_suffix), name)
    End Function

    Public Shared Function to_ref(ByVal p As parameter) As parameter
        assert(Not p Is Nothing)
        Return p.to_ref()
    End Function

    Public Function to_ref() As parameter
        Return of_ref(type, name)
    End Function

```

The changes above are mainly in [Partially implement function reference in bstyle and b2style · Hzj-jie/osi.net@43d3bfb](https://github.com/Hzj-jie/osi.net/commit/43d3bfbec2108b793ee237c9738d0211b25d3f1c#diff-8ec0866b4314468790be0a56b804fc52f8be256d05362253844fd8e33272debeL33-R52).

Avoiding copying variables into temporary variables

Note that the `logic` language only allows variables to be used when calling a function, whereas `bstyle` is much more flexible, allowing combinations of function return values, constants, and so on. Thus, in the early implementation, a simple approach was to copy every `value` into a temporary variable on the `read_scope` stack; when setting up call arguments ([bstyle/logic/ndoes/value_list.vb#L40](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_list.vb#L40)), the compiler could simply take the value at the top of this stack without caring how it was produced. Without parameter references, this was fine—just one extra copy, which was slightly wasteful. However, if the parameter is a reference, `logic` would always end up referencing the temporary variable rather than the original variable. For a reference parameter, the passed-in argument can be in one of two states: a variable or a temporary variable.

```cpp
void f(int& x) { ... }
void main() {
  int x = 0;
  f(x);  // Variable
  f(1);  // Temporary variable
}

```

A variable can be assigned to and can appear on the left-hand side of an `=` sign, so it is called an lvalue. A temporary variable cannot be assigned to and can only appear on the right-hand side of an `=` sign, so it is called an rvalue. Because modifying an rvalue makes no sense, calls like `f(1)` have been forbidden since as early as C++98—though Borland's Turbo C++ 3.0 from 1992 still allowed this statement.

Because temporary variables in `bstyle` [use specific characters](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/logic_name.vb#L17-L22) to guarantee they never collide with variables in `bstyle` source code, distinguishing between lvalues and rvalues is feasible. However, I wasn't particularly interested in this check, so I didn't implement it.

The general idea is to allow `value.target` to directly accept existing variables instead of only using `with_temp_target`.

```vb.net
Public Function with_temp_target(ByVal type As String,
                                 ByVal n As typed_node,
                                 ByVal o As writer) As vector(Of String)
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    Dim params As vector(Of builders.parameter) = Nothing
    assert(scope.current().structs().resolve(type, logic_name.temp_variable(n), params))
    assert(Not params Is Nothing)
    Dim vs As vector(Of String) = params.stream().
                                         map(Function(ByVal p As builders.parameter) As String
                                                 assert(Not p Is Nothing)
                                                 define_single_data_slot_temp_target(p.type, p.name, o)
                                                 Return p.name
                                             End Function).
                                         collect(Of vector(Of String))()

    read_targets.push(vs)
    params.stream().foreach(Sub(ByVal p As builders.parameter)
                                assert(Not p Is Nothing)
                                define_single_data_slot_temp_target(p.type, p.name, o)
                            End Sub)
    Return with_target(params)
End Function

Public Function with_target(ByVal ps As vector(Of builders.parameter)) As vector(Of String)
    assert(Not ps Is Nothing)
    Dim vs As vector(Of String) = ps.stream().
                                     map(Function(ByVal p As builders.parameter) As String
                                             assert(Not p Is Nothing)
                                             assert(Not p.name.null_or_whitespace())
                                             Return p.name
                                         End Function).
                                     collect(Of vector(Of String))()
    read_targets.push(vs)
    Return vs
End Function

Public Function with_single_data_slot_target(ByVal v As String) As String
    assert(Not v.null_or_whitespace())
    read_targets.push(vector.of(v))
    Return v
End Function

```

These changes are mainly in [avoid copying variable_name to temporary targets · Hzj-jie/osi.net@23d91f3](https://github.com/Hzj-jie/osi.net/commit/23d91f316614a52abb03d36fc656b4d580440658).

Making increment and decrement operators in `b2style` support references

Previously, without references, there was no difference between `i++` and `++i` in `b2style`, and the value of `i` itself was not modified. Now that references are supported, the behavior of these operators can be fixed.

There are two main parts to the change. First, we need to distinguish between prefix and postfix increment/decrement operations:

```vb.net
' service/compiler/b2style/rewriter/nodes/post_operation_value.vb 
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(n.child_count() = 2)
-   o.append(operations.function_name(n.child(1)))
+   o.append(strcat(operations.function_name(n.child(1)), "_post"))
    o.append("(")
    If Not l.of(n.child(0)).build(o) Then
        Return False

' service/compiler/b2style/rewriter/nodes/pre_operation_value.vb 
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(n.child_count() = 2)
-   o.append(operations.function_name(n.child(0)))
+   o.append(strcat(operations.function_name(n.child(0)), "_pre"))
    o.append("(")
    If Not l.of(n.child(1)).build(o) Then
        Return False

```

Second, we update `b2style/lib` to make the parameters of the functions corresponding to these operators references:

```cpp
biguint self_inc_post(biguint& x) {
  biguint r = x;
  logic "add b2style__x b2style__x @@prefixes@constants@int_1";
  return r;
}

```

These changes are mainly in [post and pre self-inc / self-dec · Hzj-jie/osi.net@10782c1](https://github.com/Hzj-jie/osi.net/commit/10782c1ae3ef016b0c3f9bbc00e0782c79ca0db9).

Adding some tests

Let's add a cool test: [service/tests/compiler/b2style/codegen/%2B%2Bi%2B%2B.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/%2B%2Bi%2B%2B.txt)

```cpp
#include <b2style.h>

void main() {
  int i = 1;
  i += i++;
  b2style::std_out(i);
  b2style::std_out("_");
  i += ++i;
  b2style::std_out(i);
  b2style::std_out("_");
  ++i;
  i += i++;
  b2style::std_out(i);
  b2style::std_out("_");
  ++i;
  b2style::std_out(i);
  b2style::std_out("_");
  i++;
  b2style::std_out(i);
  b2style::std_out("_");

  // This is not supported by c++ since the (++i) is a rvalue and (++i)++ does not take effect to the original i.
  // But b2style does not detect rvalue reference now, so the expected result is to increase i by 1 rather than 2.
  ++i++;
  b2style::std_out(i);
}

```

Because of the issue with binding a reference to an rvalue, this code triggers an rvalue error in C++98 or later:

```text
 In function 'int main()':
34:6: error: lvalue required as increment operand

```

However, `b2style` doesn't mind, and outputs the result `3_8_19_20_21_22`:

```vb.net
<test>
Private Shared Sub __i__()
    Dim io As New console_io.test_wrapper()
    Dim e As executor = Nothing
    assertion.is_true(b2style.with_functions(New interrupts(+io)).
                              parse(_b2style_test_data.__i__.as_text(), e))
    assertion.is_not_null(e)
    e.assert_execute_without_errors()
    assertion.equal(io.output(), "3_8_19_20_21_22")
End Sub

```

[osi.net/b2style_test.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/b2style_test.vb#L377-L386)
