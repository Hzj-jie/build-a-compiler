# Building a Programming Language from 0.1 - Storing Heap Pointers

In [Zijie He: Building a Programming Language from 0.1 - Heap Memory and Arrays](https://zhuanlan.zhihu.com/p/437868330), we covered the most basic implementation of heap memory. At the time, heap allocations were managed automatically: once declared, a heap block would be automatically deallocated when leaving its `scope`. That mechanism is very rudimentary and cannot support slightly more complex data structures—such as implementing a `vector`.

The most direct approach would be to support heap allocations inside a `struct`, but that has a drawback: since heap sizes are dynamic, every such struct would be forced to have a constructor, letting runtime functionality dictate syntax constraints.

An alternative approach is to allow storing heap pointers—that is, allowing a heap allocation under certain conditions not to be freed automatically when leaving its scope, but instead to be freed explicitly via a specific `logic` instruction.

The second approach is simpler and easier to reason about. From the description above, we need to support three things in the `logic` language:

1) Compile-time type reinterpretation (similar to `reinterpret_cast`)

As we will see shortly, the same type name is used to denote both stack variables and heap arrays of that type, but a heap pointer must be at least 8 bytes wide. Therefore, assigning a heap pointer of a type smaller than 8 bytes requires temporarily reinterpreting the variable's type.

2) Releasing ownership of a heap pointer

This prevents a heap allocation from being automatically freed when its `scope` exits.

3) Explicitly freeing a heap allocation

If a heap allocation is not freed automatically, there must be a way to free it manually.

These three requirements correspond directly to the three new commands added in [service/compiler/logic/definition.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/definition.txt#L142-L154):

```text
# redefine a variable into a different type:
#  redefine variable uint
# this command does not generate runtime instructions and works for both stack and heap variables.
redefine	string	string

# undefine a variable, it will not be accessible after this statement:
#  undefine variable
# this command does not generate runtime instructions and works for both stack and heap variables.
undefine	string

# deallocate the heap memory block.
#  dealloc_heap my_heap_ptr
dealloc_heap	string

```

The first two are implemented in [osi.net/scope.variable.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope.variable.vb):

```vb.net
' Removing items from the stack will cause the calculation of location of the references to be off, since
' variable_proxy.export function uses variable_t.size to decide the depth of stacks. So besides removing
' the items from the map, the count of removed items also needs to be tracked.
Public Function undefine(ByVal name As String) As Boolean
    undefined += uint32_1
    Return stack.erase(name)
End Function

Public Function size() As UInt32
    Return stack.size() + undefined
End Function

Public Function redefine(ByVal name As String, ByVal type As String) As Boolean
    Dim it As unordered_map(Of String, ref).iterator = stack.find(name)
    If it = stack.end() Then
        Return False
    End If
    stack(name) = (+it).second.with(type)
    Return True
End Function

```

The only subtle detail here is that we must keep track of the number of undefined variables, because each `scope` relies on the total count to calculate stack depth.

There is also a corresponding change in [osi.net/define_heap.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/define_heap.vb#L42): if the heap pointer variable is no longer defined when the `scope` ends, the heap memory is not automatically deallocated:

```vb.net
scope.current().when_end_scope(Sub()
                                   If scope.current().variables().defined_in_current_scope(name) Then
                                       ' The heap ptr has not been removed.
                                       o.emplace_back(instruction_builder.str(command.dealloc, "rel0"))
                                   End If
                                   o.emplace_back(command_str(command.pop))
                               End Sub)

```

[osi.net/dealloc_heap.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/dealloc_heap.vb) is even simpler—it just emits a `dealloc` instruction:

```vb.net
Protected Overrides Function process(ByVal ptr As variable, ByVal o As vector(Of String)) As Boolean
    assert(Not ptr Is Nothing)
    assert(Not o Is Nothing)
    o.emplace_back(instruction_builder.str(command.dealloc, ptr))
    Return True
End Function 

```

Additionally, arrays no longer use a separate `type_ptr` type: both `int i;` and `int i[100];` simply record that `i` (or what it points to) is of type `int`. Previously, using `type_ptr` for the latter was redundant since arrays are always pointers anyway. At the same time, code like:

```cpp
int i;
int j[100];
i = j;

```

is now permitted—though if the element type is smaller than 8 bytes, an extra `reinterpret_cast` is needed around the assignment. See [Add bool_array test · Hzj-jie/osi.net@465ef33](https://github.com/Hzj-jie/osi.net/commit/465ef33938645d00efd0f0cf0ef491c763da84f5) for details:

```cpp
#include <b2style.h>
#include <testing.h>

void main() {
  bool a[100];
  bool b;
  for (int i = 0; i < 100; i++) {
    // TODO: Why a[i] = ((i % 2) == 1) does not work?
    int j = i % 2;
    a[i] = (j == 1);
  }
  reinterpret_cast(b, int);
  b = a;
  reinterpret_cast(b, bool);
  for (int i = 0; i < 100; i++) {
    int j = i % 2;
    b2style::testing::assert_equal<bool>(b[i], (j == 1));
  }
  b2style::testing::finished();
}

```

This significantly reduces the amount of compiler code required, even if it is slightly counterintuitive at first glance since after `i = j`, the value stored in `i` is actually a heap address.

These changes are mainly in [service/compiler/logic/scope.variable.vb](https://github.com/Hzj-jie/osi.net/commit/80ef7b29292181b004979753ce9e2d62d0a5e4cc#diff-6130d4291023ba68e6dfbe10f6b613389f4fc224efcc3bbf7b586103b0121330) and [service/compiler/bstyle/scope.struct.vb](https://github.com/Hzj-jie/osi.net/commit/80ef7b29292181b004979753ce9e2d62d0a5e4cc#diff-e74cc8ef7d74211a46b68a4b1e20f27c983ef0bb65435cc471578746141b903d).

All of the changes above can be found in [commit/80ef7b29292181b004979753ce9e2d62d0a5e4cc](https://github.com/Hzj-jie/osi.net/commit/80ef7b29292181b004979753ce9e2d62d0a5e4cc).
