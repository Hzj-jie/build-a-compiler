# Building a Programming Language from 0.1 - Supporting Operators

As mentioned earlier, operators in `b2style` are converted into function calls in `bstyle`. In the latest implementation, operators can be used in two contexts:

```cpp
i += 1;
i + 1

```

The first is a special kind of `value-clause` called `self-value-clause`; the second is a `value`. The corresponding `rewriter`/`code_gen` implementations for both are actually very simple:

```cpp
i = b2style::add(i, 1);
b2style::add(i, 1)

```

For example, here is the implementation of the second one: [service/compiler/b2style/rewriter/nodes/binary_operation_value.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/binary_operation_value.vb)

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(n.child_count() = 3)
    o.append(operations.function_name(n.child(1)))
    o.append("(")
    If Not l.of(n.child(0)).build(o) Then
        Return False
    End If
    o.append(",")
    If Not l.of(n.child(2)).build(o) Then
        Return False
    End If
    o.append(")")
    Return True
End Function

```

Other implementations include [service/compiler/b2style/rewriter/nodes/pre_operation_value.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/pre_operation_value.vb), [service/compiler/b2style/rewriter/nodes/post_operation_value.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/post_operation_value.vb), and [service/compiler/b2style/rewriter/nodes/self_value_clause.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/self_value_clause.vb). Because functions in `bstyle` do not support pass-by-reference, `self-inc` and `self-dec` operators like `++` and `--` do not actually modify the variable in place. This differs from C++, though for an experimental language it is not particularly important.

By comparison, what matters more is the set of functions referenced in `operations.function_name`. As noted earlier, `b2style` implements a large number of built-in functions and automatically prepends them to the final compiled `bstyle` code via `prefix`. Those functions are the key to implementing operators. For instance, for `bit_and`, the corresponding functions are:

```cpp
bigint b2style__bit_and(bigint i, bigint j) {
  logic "and i i j";
  return i;
}

long b2style__bit_and(long i, long j) {
  logic "and i i j";
  return i;
}

int b2style__bit_and(int i, int j) {
  logic "and i i j";
  return i;
} 

```

Leveraging function overloading in `bstyle`, `b2style` can call the function directly without worrying about operand types. Similarly, by varying function parameter types, `b2style` can easily support mixed-type operations—for example, `int + long` producing a `long` result. [service/compiler/b2style/rewriter/codegen/prefix.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/codegen/prefix.txt)

Note that the `primitive` processor does not understand types; for example, the `add` instruction operates on `big_uint`. When adding `int + int`, the result may exceed the range of `int`. A simple way to handle this is to truncate the result to the target width after the operation completes:

```cpp
int b2style__fit_in_int(int x) {
  logic "cut_slice x x @@prefixes@constants@int_0 @@prefixes@constants@size_of_int";
  return x;
}

int b2style__add(int i, int j) {
  logic "add i i j";
  return b2style__fit_in_int(i);
}

```
