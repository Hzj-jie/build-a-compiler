# Building a Programming Language from 0.1 - read_scoped and Handling value Nodes

Without a concrete use case, explaining the concept of `read_scoped` directly would be very difficult to understand. So let's start with handling `value` nodes first.

The `value` node is one of the more complex parts of the entire bstyle compiler. We should first look at where `value` is used:

- `condition`: `if (some_value) { ... }`
- `value-definition`: `int i = some_value;`
- `value-clause`: `i = some_value;`
- `while`: `while (some_value) { ... }`
- `for-loop`: `for (...; less(i, 100); ...) { ... }`
- `function-call`: `func(value_1, value_2 ...)`
- `return-clause`: `return some_value;`

Now let's look at the sources of a `value`:

- `function-call`
- `variable-name`
- `integer`
- `float`
- `bool`
- `string`

Simply put, they fall into two categories: one requires a temporary variable to store an intermediate value, and the other directly uses an existing variable. To simplify the implementation, even `value-clause` uses a temporary variable to receive the result of `value`. However, this temporary variable can be moved to the destination address using the `move` operation in the `logic` language. Therefore, `value` only needs the `with_temp_target` function to generate a temporary variable. [service/compiler/bstyle/logic/nodes/value.target.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value.target.vb)

Note that evaluations of `value` can be nested, which leads to the concept of `read_scoped`. [service/compiler/logic/logic_gen/scoped/read_scoped.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/logic_gen/scoped/read_scoped.vb)

The mechanism behind `read_scoped` is very simple: it is a stack that automatically pops whenever it is read. Its usage in `value` becomes straightforward: in the source of a `value`, `with_temp_target` directly pushes the temporary variable name onto the `read_scoped` stack. Then, depending on the use case, an ancestor node of `value` reads it from the stack.

- Alternatively, we could push onto the stack in the ancestor node of `value` and read the temporary variable name at the source of `value`. There is no fundamental difference between the two approaches; writing first and reading later simply aligns more closely with the natural order of thought: `i = 100; while i ...`.
- There is also a counterpart concept called `write_scoped`, which we will cover later.

When pushing onto the stack:

```text
Public Function with_temp_target(ByVal type As String, ByVal n As typed_node, ByVal o As writer) As String
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    Dim value_name As String = Nothing
    value_name = strcat("raw_value_@", n.word_start(), "-", n.word_end())
    If defined_temp_targets.find(value_name) = defined_temp_targets.end() Then
        builders.of_define(value_name, ta(type)).to(o)
        defined_temp_targets.emplace(value_name)
    End If
    read_targets.push(value_name)
    Return value_name
End Function

```

There are a few implementation details worth noting here:

- Since variable names in the `logic` language support any characters other than whitespace, we construct temporary variable names using `"@"`, `"-"`, and the position of the `typed_node` to avoid naming collisions.
- When implementing statements like `for-loop` or `while`, the `value` serving as the loop condition is evaluated twice—once before `while_then` in the `logic` language, and once at the end of the body `paragraph`—and both evaluations must use the same temporary variable. Therefore, a `defined_temp_targets` set is used to prevent duplicate definitions.

When popping from the stack, taking `condition` as an example:

```text
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() >= 5)
    If Not l.of(n.child(2)).build(o) Then
        Return False
    End If
    Using read_target As read_scoped(Of String).ref = code_gen_of(Of value)().read_target()
        Dim satisfied_paragraph As Func(Of Boolean) = Nothing
        satisfied_paragraph = Function() As Boolean
                                  Return l.[of](n.child(4)).build(o)
                              End Function
        If n.child_count() = 5 Then
            Return builders.of_if(+read_target, satisfied_paragraph).to(o)
        End If
        Dim unsatisfied_paragraph As Func(Of Boolean) = Nothing
        unsatisfied_paragraph = Function() As Boolean
                                    Return l.[of](n.child(5)).build(o)
                                End Function
        Return builders.of_if(+read_target, satisfied_paragraph, unsatisfied_paragraph).to(o)
    End Using
End Function

```

The third child node of `condition` is `value`. Calling `l.of(n.child(2)).build(o)` pushes a temporary variable name onto `value`'s `read_scoped` stack, after which we simply read that variable name to generate the subsequent code.

This also explains what was mentioned earlier about needing a reference to a specific type of `code_gen`: `code_gen_of(Of value)()` returns the `code_gen` implementation for the `value` type in the current `code_gens` collection, allowing `condition` to call its `read_target()` function to retrieve the current temporary variable name.
