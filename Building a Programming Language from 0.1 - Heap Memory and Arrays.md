# Building a Programming Language from 0.1 - Heap Memory and Arrays

Beginners in C++ have probably all wondered: why is the following program invalid?

```cpp
#include<iostream>
int main() {
    int n;
    std::cin>>n;
    int a[n];
    return 0 ;
}

```

Yet this one is valid:

```cpp
int main() {
    const int n = 100;
    int a[n];
    return 0 ;
}

```

Early versions of Turbo C++ would report a strange error:

```text
Error main.cpp 4: Constant expression required in function main()

```

And even up to Visual Studio 2019, this still produces an error:

```text
C:\temp>cl b.cc
Microsoft (R) C/C++ Optimizing Compiler Version 19.28.29337 for x86
Copyright (C) Microsoft Corporation.  All rights reserved.

b.cc
b.cc(4): error C2131: expression did not evaluate to a constant
b.cc(4): note: failure was caused by a read of a variable outside its lifetime
b.cc(4): note: see usage of 'n'

```

Meanwhile, `g++` and `clang++` support this syntax. So what exactly is the difference between `int x[100]` and `int x[y]`? This comes down to static allocation versus dynamic allocation.

If you look closely at our previous examples, regardless of what features were supported, once translated into `primitive` code, all memory accesses were at fixed locations determined by their relative offsets from the top of the stack. What is the principle behind this? Notice that when any piece of code executes, the memory space it declares is predetermined and depends solely on the code itself. For example, `int x` declares a 4-byte memory block, and `double x` declares an 8-byte memory block—all of which can be determined at compile time. Therefore, allocating and accessing stack space can always be known at compile time. However, this has two problems: 1) there is no way to allocate a memory block whose length cannot be determined at compile time, and 2) the size of the stack is fixed and cannot be allocated on demand.

We can discuss the second problem in more detail later when compiling `logic` to a more x86-like instruction set; the one-sentence explanation is that you cannot allocate all available memory to the stack just because it "might need that much," so the stack size is fixed in advance at compile time or by the operating system.

The first problem is more obvious. Consider a realistic example: the user inputs a sequence of numbers of arbitrary length, and the program sorts and outputs them. That sounds like an ACM problem 1001, but the arbitrary length is awkward. Of course, any input has some upper bound, so a 1001 problem isn't that troublesome—say there are at most 1M numbers, so we just declare a fixed-length array of 1M elements.

Now let's make the problem slightly more complex: the user inputs 100 groups of numbers of arbitrary length; the total number of elements across all groups does not exceed 8M, but any single group may have up to 8M elements. We are required to sort and output each group individually, and finally sort and output all the numbers together. Okay, that's ACM problem 1002, with a memory limit of 32 MB. If we use fixed-length arrays, 100 integer arrays of 8M elements each would take 3,200 MB—unfortunately exceeding the 32 MB limit. Therefore, supporting only static allocation is not enough; we also need a way to dynamically allocate and access memory space.

In common programming languages, this dynamically allocated space has a strange name: heap memory. Unlike stack memory, heap memory seems to have nothing to do with the heap data structure. Of course, the name doesn't matter; a language needs to support this feature.

Under our architecture of `primitive`, `logic`, `bstyle`, and `b2style`, the following changes are needed:

- `primitive` needs to support allocating and accessing heap memory. Just like the `push` instruction for stack space, an instruction to allocate heap memory is not strictly mandatory in theory, but since `primitive` does not pre-allocate memory, an instruction is needed to mark the allocation. Accessing heap memory is mandatory, though what "accessing heap memory" actually means is an interesting concept that we will discuss when we get to the `primitive` changes.
- `logic` and `bstyle` need to support allocating and accessing heap memory.
- `b2style` is slightly more involved, as it needs to support heap memory and `struct` together. Although `bstyle` also supports `struct`, some syntactic details are only implemented in `b2style`.

## Supporting Heap Memory in `primitive`

Allocating and freeing heap memory is straightforward and requires just two instructions:

```text
alloc	data_ref	data_ref	#allocate a block of memory with size (*B big_uint) and store its reference into (*A), each one is a data slot
dealloc	data_ref	#de-allocate a block of memory identified by its reference (*A)

```

During allocation, the address of the allocated space is stored into the return value, and the parameter of the deallocation instruction is the address returned during allocation. Implementing these two instructions on the `primitive` processor involves nothing particularly special—a free-list or similar data structure suffices, so we won't dwell on it. ([osi.net/simulator.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/simulator.vb#L176-L200))

One difference from conventional systems, however, is that the `primitive` processor knows the size of each heap allocation, so `dealloc` can accurately reclaim all the memory previously allocated for that block.

Accessing heap memory is a more interesting question. First, we need to understand what "accessing heap memory" means. Unlike stack space, the location of the heap memory being accessed is not decided at compile time, but only becomes known at runtime. It is analogous to asking someone to pick two cards from the hand they are holding and sum their points: the numbers on the chosen cards are the final values, while the positions of the chosen cards in the hand are the inputs—in other words, pointers.

That being said, although the processor itself does not necessarily need to understand what "heap memory" is, it needs a way to access the data at the location pointed to by a pointer. This is what the square brackets commonly seen in x86 instructions mean: `mov eax, [ebx]`.

The standard x86 instruction set places many restrictions on such memory accesses; for instance, `mov [eax], [ebx]` is illegal, and many basic arithmetic instructions do not allow directly accessing two memory addresses at once. As a result, `mov` and its extended instructions account for roughly a third or even more of all instructions in binaries on x86 systems.

![image](https://picx.zhimg.com/v2-0ef3f5d11d40185b96333e56ea18fdb3_r.jpg?source=d16d100b)
This is a small snippet from `ls.exe` in Cygwin64.

The `primitive` processor itself has no concept of registers, so it does not have as many restrictions. Our simplified approach is that all instructions can accept heap pointers. Because `primitive` implements the heap and the stack separately, accessing stack space via memory pointers is not permitted.

The underlying mechanism is to provide two additional location types—`habs` and `href`—for [data_ref](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/data_ref.vb), and perform a dereference on these two location types in [simulator.access](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/simulator.vb#L160).

The implementation is mostly contained here: [Support heap in all primitive instructions · Hzj-jie/osi.net@6e8a8e4](https://github.com/Hzj-jie/osi.net/commit/6e8a8e42afa5aaa40f97af4c1efef25620669906)

## Supporting Heap Memory in `logic`

With the foundational design in `primitive`, supporting heap memory in `logic` becomes very direct. Again, it comes down to two questions: allocating and freeing heap memory, and how to access heap memory. For the first version of heap memory support, let's start with something simple.

- The [define_heap](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/definition.txt#L123) instruction

For now, the `logic` language only supports [allocating heap memory](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/define_heap.vb), while the allocated heap memory is automatically freed when leaving the `scope`. [osi.net/scope.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope.vb#L177-L191)

```vb.net
Protected Overrides Sub when_dispose()
    new_scope.heap().
              stream().
              foreach(Sub(ByVal name As String)
                          Dim v As String = Nothing
                          assert(new_scope.export(name).data_ref.export(v))
                          o.emplace_back(instruction_builder.str(command.dealloc, v))
                      End Sub)
    Dim i As UInt32 = 0
    While i < new_scope.size()
        o.emplace_back(instruction_builder.str(command.pop))
        i += uint32_1
    End While
    MyBase.when_dispose()
End Sub

```

- Accessing heap memory

In [logic/variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/variable.vb), `[` and `]` are likewise used to denote indexing, so an `optional<variable>` was added to `variable` as the index.

```vb.net
Public NotInheritable Class variable
    Public ReadOnly name As String
    Public ReadOnly index As [optional](Of variable)
    Public ReadOnly type As String
    Public ReadOnly size As [optional](Of UInt32)

```

Parsing the index is a matter of simple string manipulation and recursion, though various edge cases make the code somewhat long. The only notable detail is that the target address must be computed—since `x[1] = *(x + 1)`—so `logic` needs to allocate a temporary variable to hold the target address.

```vb.net
Dim ptr_name As String = scope.current().unique_name()
assert(define.export(ptr_name, types.heap_ptr_type, v))
Dim d As data_ref = scope.current().export(ptr_name).data_ref
Dim r As scope.exported_ref = Nothing
If Not scope.current().export(name.Substring(0, index_start), r) Then
    errors.variable_undefined(name.Substring(0, index_start))
    Return False
End If
v.emplace_back(instruction_builder.str(
    command.add,
    d,
    r.data_ref.ToString(),
    index.ToString()))

```

[`heap.txt` in `logic`](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/logic/codegen/heap.txt) tests the two features above:

```text
type uint 4
type bool 1

define true bool
copy_const true b1

## Always use a new scope to ensure the heap allocation can be freed.
if true {

    define uint_100 uint
    copy_const uint_100 i100
    define_heap h uint uint_100
    define uint_99 uint
    copy_const uint_99 i99
    copy h[uint_99] uint_99
    define out uint
    copy out h[uint_99]

    define temp_string type*
    interrupt putchar out temp_string

}

```

## `bstyle` and `b2style`

[heap-name](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/codegen/syntaxer_rule.txt#L40) is a core rule in `bstyle` and `b2style` that supports both heap memory allocation and access:

```text
heap-name name start-square-bracket value end-square-bracket
heap-declaration name heap-name
variable-name [heap-name, name]

```

There is nothing else particularly special; in [bstyle.heap_name](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/heap_name.vb), the `value` needs to be evaluated into a temporary variable for use in `logic`:

```vb.net
Return build(n.child(2),
             o,
             Function(ByVal indexstr As String) As Boolean
                 Return l.typed_code_gen(Of variable_name)().build(
                            n.child(0),
                            Function(ByVal type As String,
                                     ByVal ps As vector(Of single_data_slot_variable)) As Boolean
                                l.typed_code_gen(Of value)().with_target(
                                    type,
                                    ps.stream().
                                       map(Function(ByVal d As single_data_slot_variable) As _
                                               single_data_slot_variable
                                               assert(Not d Is Nothing)
                                               Return d.with_index(indexstr)
                                           End Function).
                                       collect(Of vector(Of single_data_slot_variable))())
                                Return True
                            End Function,
                            Function(ByVal type As String, ByVal source As String) As Boolean
                                l.typed_code_gen(Of value)().with_single_data_slot_target(
                                    type, variable.name_of(source, indexstr))
                                Return True
                            End Function,
                            o)
             End Function)

```

In `b2style`, things are slightly more complex because both `struct` and `class` are flattened into member variables, so heap access works differently from what one might conventionally expect:

```cpp
#include <b2style.h>
struct S { int x; }
void main() {
  S s[10];
  b2style::std_out(s[0].x);
  b2style::std_out(s.x[0]);
}

```

In the code above, `s.x[0]` is the access form supported by default in `logic` (and `bstyle`), whereas `s[0].x` requires [heap-struct-name](https://github.com/Hzj-jie/osi.net/commit/2c01384cf26e58315361bd4e739e1936b7b49b96). Similarly, calling a member function on a heap-allocated `class` instance also needs special handling via [heap-struct-function-call](https://github.com/Hzj-jie/osi.net/commit/d9ed1108cb4b6fdd5f8188516388b6df3888c278).

Skipping the remaining minor details, once the items above are implemented, `b2style` can support primitive types, `struct`s, and `class`es on the heap.

[service/tests/compiler/b2style/codegen/heap.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/heap.txt)

```cpp
#include <b2style.h>

struct S {
  string s;
};

void f(S s) {
  b2style::std_out(s.s);
}

S new(string s) {
  S r;
  r.s = s;
  return r;
}

void main() {
  int x = 1;
  int v[x++];
  S v2[1 + ++x++];
  v[0] = 100;
  v2[0] = new("abc");
  v2[1] = new("def");
  f(v2[0]);
  f(v2[1]);
  v2[2].s = "ghi";
  f(v2[2]);
  // TODO: Make v2[2].  s = "ghi" work.
  b2style::std_out(v[0]);
}

```

[service/tests/compiler/b2style/codegen/class_on_heap.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/class_on_heap.txt)

```cpp
#include <b2style.h>

class C {
  int x;
  string y;

  void print() {
    b2style::std_out(this.x);
	b2style::std_out(" ");
	b2style::std_out(this.y);
  }
};

class C2 {
  C x;
  int y;

  void print() {
    this.x.print();
	b2style::std_out(" ");
	b2style::std_out(this.y);
	b2style::std_out("\n");
  }
};

void main() {
  C2 cs[100];
  for (int i = 0; i < 100; i++) {
    cs[i].x.x = i;
    cs[i].x.y = b2style::int_to_str(i);
	cs[i].y = i;
  }

  for (int i = 0; i < 100; i++) {
    cs[i].print();
  }
}

```
