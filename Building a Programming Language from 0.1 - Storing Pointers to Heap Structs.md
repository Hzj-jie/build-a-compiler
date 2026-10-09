# Building a Programming Language from 0.1 - Storing Pointers to Heap Structs

An earlier article discussed [Zijie He: Building a Programming Language - Storing Heap Pointers](https://zhuanlan.zhihu.com/p/476306636), though the implementation at the time could only store primitive types. Recall how structs are implemented in bstyle: because every piece of data in the primitive processor has a variable length—in order to allow arbitrary-precision arithmetic ([Calculating the Value of π - Arbitrary-Precision Arithmetic](https://www.zhihu.com/column/c_1216677226609082368))—the downside is that traditional structs represented by a pointer to a contiguous block of memory cannot be handled by the primitive processor. Therefore, a struct is expanded into several primitive types, and `.` is merely a way to name each member variable of the struct without carrying much syntactic meaning on its own. This is a simple approach, but it presents a similar issue when dealing with heap data. For example:

```cpp
struct S { int x; string y; }
void main() {
  S s[10];
}
```

is actually expanded into:

```cpp
int s.x[10];
string s.y[10];
```

Since these two heap allocations share the same lifetime, storing a pointer to heap space also requires taking structs into account.

Notice that when executing the statement `S s[10]`, bstyle automatically expands the definition of `S`. By the same token, when executing `logic "dealloc_heap x";` and `logic "undefine x";` mentioned earlier, the struct definition must also be expanded at the same time.

There are two approaches here. The first is to provide compiler instructions disguised as functions—similar to `reinterpret_cast`—naturally named `dealloc` and `undefine`. The second is that, since these two features are only used when manipulating heap space, we could generate a specific `T alloc_heap(int size);` function for each struct.

The second approach has obvious drawbacks in the bstyle language: first, we would still need a way to mark whether `S s[10];` allocates heap memory that is not automatically freed; second, either `int x[10];` and `S s[10];` would have to be handled in different ways, or we would have to add an `int alloc_heap(int size);` function for every primitive type—yet bstyle does not know all the primitive types, as the logic language can add primitive types arbitrarily.

At the same time, similar to `S s[10];`, providing two function-like compiler instructions forms a natural pair. Therefore, the syntax we need to implement looks like this:

```cpp
struct S { int x; string y; }

void main() {
  S s[10];
  S ptr = s;
  undefine(s);
  dealloc(ptr);
}
```

### 1. Syntax and Lexis

Although it is rather mundane and has been covered many times before, it is still worth explaining briefly.

Lexically, we add two keywords: `kw-undefine` and `kw-dealloc`.

```text
  kw-delegate delegate
+ kw-undefine undefine
+ kw-dealloc dealloc

  single-line-comment //[*|\n]*
```

Syntactically, we add two statements: `undefine` and `dealloc`.

```text
+ # Function-like instructions.
  reinterpret-cast kw-reinterpret-cast start-bracket variable-name comma type-name end-bracket
+ undefine kw-undefine start-bracket variable-name end-bracket
+ dealloc kw-dealloc start-bracket variable-name end-bracket

  root-type-with-semi-colon [value-definition, heap-declaration, value-declaration, logic, typedef, delegate]? semi-colon
  base-root-type [root-type-with-semi-colon, function, include, define, ifndef-wrapped, struct]
- base-sentence-with-semi-colon [..., delegate]? semi-colon
+ base-sentence-with-semi-colon [..., delegate, undefine, dealloc]? semi-colon 
```

Like `reinterpret_cast`, the input to these two statements can only be a `variable-name`.

### 2. logic/nodes in bstyle and rewriter/nodes in b2style

In b2style, nothing special needs to be done; we simply pass them through to bstyle ([service/compiler/b2style/b2style.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/b2style.vb)).

```vb.net
  with_of_leaf_nodes(
     ' ...
+    "kw-undefine",
+    "kw-dealloc",

  with_of_all_childrens(
     ' ...
-    "struct-body").
+    "struct-body",
+    "undefine",
+    "dealloc").
```

In bstyle, we have two implementations similar to `reinterpret_cast`: `undefine` and `dealloc` ([service/compiler/bstyle/bstyle.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/bstyle.vb)).

```vb.net
   Return New code_gens_registrar(Of logic_writer)().
                 ' ...
+                with(undefine.instance).
+                with(dealloc.instance).
```

[service/compiler/bstyle/logic/nodes/undefine.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/undefine.vb)

```vb.net
Option Explicit On
Option Infer Off
Option Strict On

Imports osi.root.connector
Imports osi.service.automata
Imports osi.service.compiler.logic

Partial Public NotInheritable Class bstyle
    Private NotInheritable Class undefine
        Implements code_gen(Of logic_writer)

        Public Shared ReadOnly instance As New undefine()

        Private Sub New()
        End Sub

        Public Function build(ByVal n As typed_node,
                              ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            assert(n.child_count() = 4)
            Dim name As String = n.child(2).input_without_ignored()
            Return struct.undefine(name, o) OrElse
                   builders.of_undefine(name).to(o)
        End Function
    End Class
End Class
```

[service/compiler/bstyle/logic/nodes/dealloc.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/dealloc.vb)

```vb.net
Option Explicit On
Option Infer Off
Option Strict On

Imports osi.root.connector
Imports osi.service.automata
Imports osi.service.compiler.logic

Partial Public NotInheritable Class bstyle
    Private NotInheritable Class dealloc
        Implements code_gen(Of logic_writer)

        Public Shared ReadOnly instance As New dealloc()

        Private Sub New()
        End Sub

        Public Function build(ByVal n As typed_node,
                              ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
            assert(Not n Is Nothing)
            assert(Not o Is Nothing)
            assert(n.child_count() = 4)
            Dim name As String = n.child(2).input_without_ignored()
            Return struct.dealloc_from_heap(name, o) OrElse
                   builders.of_dealloc_heap(name).to(o)
        End Function
    End Class
End Class
```

The two pieces of code are almost identical: they first try handling the variable as a struct, and if that fails, they fall back to handling it as a primitive type. Thus, the core logic resides in `struct`.

### 3. struct.undefine and struct.dealloc

In fact, these two functions are not complicated at all. They simply enumerate all primitive fields of the struct and invoke the logic language's `dealloc_heap` or `undefine` instruction on each of them in turn.

```vb.net
Public Shared Function dealloc_from_heap(ByVal name As String, ByVal o As logic_writer) As Boolean
    assert(Not o Is Nothing)
    Dim v As struct_def = Nothing
    If Not scope.current().structs().resolve(name, v) Then
        Return False
    End If
    assert(Not v Is Nothing)
    Return v.for_each_primitive(Function(ByVal m As builders.parameter) As Boolean
                                    assert(Not m Is Nothing)
                                    Return builders.of_dealloc_heap(m.name).to(o)
                                End Function)
End Function

Public Shared Function undefine(ByVal name As String, ByVal o As logic_writer) As Boolean
    assert(Not o Is Nothing)
    Dim v As struct_def = Nothing
    If Not scope.current().structs().resolve(name, v) Then
        Return False
    End If
    assert(Not v Is Nothing)
    Return v.for_each_primitive(Function(ByVal m As builders.parameter) As Boolean
                                    assert(Not m Is Nothing)
                                    Return builders.of_undefine(m.name).to(o)
                                End Function)
End Function 
```

### 4. Some Tests

[service/tests/compiler/b2style/self-test/heap.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/heap.txt) tests these two instructions:

```cpp
#include <b2style.h>
#include <testing.h>
#include <b2style/stdio.h>

struct S {
  int x;
  string y;
};

void main() {
  S s;
  {
    S x[10];
    s = x;
    undefine(x);
  }
  for (int i = 0; i < 10; i++) {
    s[i].x = i + 1;
    s[i].y = b2style::int_to_str(i + 2);
  }
  for (int i = 0; i < 10; i++) {
    b2style::testing::assert_equal<int>(s[i].x, i + 1);
    b2style::testing::assert_equal<string>(s[i].y, b2style::int_to_str(i + 2));
  }
  dealloc(s);
  b2style::testing::finished();
}
```

This is also very close to how `vector` was implemented earlier: using an ordinary variable to point to heap memory.

### 5. b2style::heap_ptr<T>

Notice that `undefine` and `dealloc` are quite dangerous to use directly—it is easy to either double-free or forget to free memory—so encapsulating this functionality into a class is much safer. That is [service/compiler/b2style/lib/b2style/heap_ptr.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/heap_ptr.h).

```cpp
#ifndef B2STYLE_LIB_B2STYLE_HEAP_PTR_H
#define B2STYLE_LIB_B2STYLE_HEAP_PTR_H

#include <b2style/types.h>
#include <assert.h>

namespace b2style {

// Type T is required to use the dealloc instruction.
template <T>
class heap_ptr {
  T _a;
  int _s;

  int size() {
    return this._s;
  }

  bool empty() {
    return this.size() == 0;
  }

  void construct() {
    this._s = 0;
  }

  void destruct() {
    if (this.empty()) return;
    dealloc(this._a);
    this.construct();
  }

  void alloc(int size) {
    this.destruct();
    ::assert(size > 0);
    T x[size];
    this._a = x;
    this._s = size;
    undefine(x);
  }

  void construct(int size) {
    this.construct();
    this.alloc(size);
  }

  T get(int index) {
    ::assert(index >= 0);
    ::assert(index < this.size());
    return this._a[index];
  }

  void set(int index, T v) {
    ::assert(index >= 0);
    ::assert(index < this.size());
    this._a[index] = v;
  }

  void release() {
    this.construct();
  }
};

}  // namespace b2style

#endif  // B2STYLE_LIB_B2STYLE_HEAP_PTR_H
```

These are mostly implementation details with nothing particularly unusual.

### 6. Both std/vector and b2style/ref.h Can Use b2style::heap_ptr<T>

[service/compiler/b2style/lib/b2style/ref.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/ref.h) is a unique pointer to a single variable on the heap:

```cpp
#ifndef B2STYLE_LIB_B2STYLE_REF_H
#define B2STYLE_LIB_B2STYLE_REF_H

#include <b2style.h>
#include <b2style/heap_ptr.h>
#include <assert.h>

namespace b2style {

template <T>
class ref {
  heap_ptr<T> _a;

  void destruct() {
    this._a.destruct();
  }

  bool empty() {
    return this._a.empty();
  }

  void set(T v) {
    if (this.empty()) this._a.alloc(1);
    this._a.set(0, v);
  }

  void alloc() {
    T v;
    this.set(v);
  }

  T get() {
    ::assert(!this.empty());
    return this._a.get(0);
  }

  T release() {
    T r = this.get();
	this.destruct();
    return r;
  }

  void construct() {}

  void construct(T v) {
    this.construct();
    this.set(v);
  }

  void construct(ref<T>& other) {
    this.construct(other.release());
  }
};

}  // namespace b2style

#endif  // B2STYLE_LIB_B2STYLE_REF_H
```

Meanwhile, `vector` has also been updated so that it no longer needs to manage heap allocation and deallocation on its own ([service/compiler/b2style/lib/std/vector](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/std/vector)):

```cpp
#ifndef B2STYLE_LIB_STD_VECTOR
#define B2STYLE_LIB_STD_VECTOR

#include <b2style.h>
#include <assert.h>
#include <b2style/heap_ptr.h>
#include <std/types.h>

namespace std {

template <T>
class vector {
  ::b2style::heap_ptr<T> _a;
  int _s;

  int size() {
    return this._s;
  }

  int capacity() {
    return this._a.size();
  }

  T get(int index) {
    ::assert(index >= 0, "get");
    ::assert(index < this.size(), "get");
    return this._a.get(index);
  }

  void set(int index, T v) {
    ::assert(index >= 0, "set");
    ::assert(index < this.size(), "set");
    this._a.set(index, v);
  }

  void destruct() {
    this._a.destruct();
  }

  void reserve(int size) {
    ::b2style::heap_ptr<T> x(size);
    for (int i = 0; i < this.size(); i++) {
      x.set(i, this.get(i));
    }
	this.destruct();
    this._a = x;
	x.release();
  }

  void construct(int size) {
    this.reserve(size);
  }

  void construct() {
    this.construct(4);
  }

  void push_back(T v) {
    if (this.size() == this.capacity()) {
      this.reserve(this.capacity() << 1);
    }
    this._s++;
    this.set(this.size() - 1, v);
  }

  void pop_back() {
    ::assert(this.size() > 0, "pop_back");
    this._s--;
  }

  void clear() {
    this._s = 0;
  }
};

}  // namespace std

#endif  // B2STYLE_LIB_STD_VECTOR
```

These changes are mainly in [osi.net/commit/b783d352501b4d40154819ba0249441749c2e1db](https://github.com/Hzj-jie/osi.net/commit/b783d352501b4d40154819ba0249441749c2e1db).
