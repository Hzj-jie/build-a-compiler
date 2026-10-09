# Building a Programming Language from 0.1 - Variables in the logic Language

In the `logic` language, `variable` itself is simply a wrapper around `scope` functionality, serving as a reference to a variable. When generating `primitive` instructions, `variable` makes it much more convenient to retrieve a variable's location on the memory stack. Therefore, its most fundamental operation is:

```text
Public Function ref() As String
    Dim r As String = Nothing
    assert(scope.export(name, r))
    Return r
End Function

```

Because the instruction generation logic might call `define` again to define another variable after creating a `variable` instance, its referenced stack offset may change. Querying it on every access is the safe approach. Since the number of variables in any single `scope` is always limited, and this only affects compile-time performance, the performance impact is negligible.

Of course, I also gave `variable` a few additional responsibilities, such as checking whether a variable can be copied or moved into another variable, or whether it can represent a specific type like an integer or a `bool` value. Given that the `primitive` processor always uses arbitrary-precision integers (bigints) when performing arithmetic operations, restrictions are slightly relaxed in certain places to avoid forcing everything to use `bigint`. Unless the `primitive` processor overflows on integer operations like conventional processors do, variable width during arithmetic operations does not matter much. Therefore, we can simply use the `cut` instruction after an arithmetic operation to truncate the variable's width back to its defined width.

[service/compiler/logic/variable.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/variable.vb)
