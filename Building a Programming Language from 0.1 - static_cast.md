# Building a Programming Language from 0.1 - static_cast

Although the previous article mentioned building a compiler that translates a C++-like language directly into the logic language, as an important language feature, `static_cast` is still worth discussing.

`static_cast` in b2style implements a very small subset of C++'s `static_cast` functionality—specifically, conversions analogous to casting between `void*` and `A_Struct*` ([http://cpp.sh/76ynu](http://cpp.sh/76ynu)):

```cpp
// Example program
#include <iostream>
using namespace std;

struct S {
    int x;
    double y;
};

int main() {
    S* s = new S{1, 2.0};
    void* y = static_cast<void*>(s);
    S* z = static_cast<S*>(y);
    
    cout << s->x << " " << s->y << endl;
    cout << z->x << " " << z->y << endl;
    cout << y << endl;
}
```

Of course, b2style does not have pointer syntax like `S*`, so `void*` corresponds to `type_ptr`, which is a 64-bit integer. At the same time, because heap allocations are performed in units of primitive types, `static_cast` actually generates code to compute the heap locations of each member variable of a class or struct.

It is more direct to look at an example, which is itself a test case ([service/tests/compiler/b2style/self-test/static_cast_ptr_type.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type.txt)):

```cpp
#include <b2style.h>
#include <testing.h>
#include <b2style/to_str.h>

struct S {
  int x;
  string y;
};

void f(S s) {
  for (int i = 0; i < 10; i++) {
    b2style::testing::assert_equal<int>(s[i].x, i);
    b2style::testing::assert_equal<string>(s[i].y, b2style::int_to_str(i));
  }
}

void f(type_ptr p) {
  static_cast(p, S);
  f(p);
}

void main() {
  S s[10];
  for (int i = 0; i < 10; i++) {
    s[i].x = i;
    s[i].y = b2style::to_str(i);
  }
  static_cast(s, type_ptr);
  f(s);
  b2style::testing::finished();
}
```

This example is quite simple: it allocates a 10-element array on the heap where each element is a `struct S`, assigns initial values to the 10 elements, casts `s` to `type_ptr`, casts it back to `struct S`, and finally verifies that the element values match the initial values.

Yet this is sufficient, because we can store only `type_ptr` (i.e., an integer) and later restore its type information whenever needed.

As for the implementation, it is quite similar to `reinterpret_cast`, except that in addition to updating the compiler's internal state, `static_cast` also generates instructions to calculate the heap locations. Like `reinterpret_cast`, this syntax exists in both bstyle and b2style:

```text
--- service/compiler/shared/nlexer_rule.txt
  kw-statement __STATEMENT__
+ kw-static-cast static_cast

  single-line-comment //[*|\n]*

--- service/compiler/shared/syntaxer_rule.txt
  dealloc kw-dealloc start-bracket variable-name end-bracket
+ static-cast kw-static-cast start-bracket variable-name comma type-name end-bracket

- base-sentence-with-semi-colon [..., undefine, dealloc]? semi-colon
+ base-sentence-with-semi-colon [..., undefine, dealloc, static-cast]? semi-colon
```

Next is the implementation of `static_cast` itself ([service/compiler/bstyle/logic/nodes/static_cast.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/static_cast.vb)):

```vb.net
Private Function build(ByVal n As typed_node,
                       ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(n.child_count() = 6)
    Dim name As String = n.child(2).input_without_ignored()
    Dim type As String = scope.current().type_alias()(n.child(4).input_without_ignored())
    If scope.current().structs().types().defined(type) AndAlso
       scope.current().structs().variables().defined(name) Then
        ' Convert from struct ptr to struct ptr.
        raise_error(error_type.user,
                    "Cannot static_cast a struct instance of ",
                    name,
                    " to another struct type ",
                    type,
                    ", convert it to ",
                    compiler.logic.scope.type_t.ptr_type,
                    " first. ",
                    "TODO: Implementing this ""shortcut"" is quite complicated since the struct fields need ",
                    "to be carefully considered to avoid conflicting with the target type.")
        Return False
    End If
    If scope.current().structs().types().defined(type) Then
        ' Convert from type_ptr to struct ptr.
        Dim sdef As struct_def = Nothing
        assert(scope.current().structs().resolve(type, name, sdef))
        assert(Not sdef Is Nothing)
        assert(sdef.primitive_count() > 0)
        Return sdef.for_each_primitive(Function(ByVal t As builders.parameter) As Boolean
                                           Return value_declaration.declare_single_data_slot(t.type, t.name, o)
                                       End Function) AndAlso
               builders.start_scope(o).of(
                   Function() As Boolean
                       Dim offset_name As String = "@offset"
                       ' TODO: Using builders.of_define is sufficient.
                       Return value_declaration.declare_single_data_slot(
                                  compiler.logic.scope.type_t.ptr_type,
                                  offset_name,
                                  o) AndAlso
                              sdef.for_each_primitive(
                                  Function(ByVal t As builders.parameter) As Boolean
                                      ' TODO: Include bstyle/lib/const.h automatically.
                                      Return builders.of_add(t.name, offset_name, name).to(o) AndAlso
                                             builders.of_add(offset_name,
                                                             offset_name,
                                                             "@@prefixes@constants@ptr_offset").to(o)
                                  End Function)
                   End Function) AndAlso
               builders.of_undefine(name).to(o) AndAlso
               scope.current().variables().redefine(type, name)
    End If
    If scope.current().structs().variables().defined(name) Then
        ' Convert from struct ptr to type_ptr.
        Dim sdef As struct_def = Nothing
        assert(scope.current().structs().variables().resolve(name, sdef))
        assert(Not sdef Is Nothing)
        Return builders.of_define(name, type).to(o) AndAlso
               builders.of_copy(name, sdef.primitives().
                                      find_first().
                                      map(Function(ByVal x As builders.parameter) As String
                                              assert(Not x Is Nothing)
                                              Return x.name
                                          End Function).
                                      or_assert()).to(o) AndAlso
               sdef.for_each_primitive(Function(ByVal t As builders.parameter) As Boolean
                                           Return builders.of_undefine(t.name).to(o) AndAlso
                                                  scope.current().variables().undefine(t.name)
                                       End Function) AndAlso
               scope.current().variables().redefine(type, name)
    End If
    raise_error(error_type.user, "Unsupported static_cast ", name, " to ", type)
    Return False
End Function 
```

This function can be broken down into three `If` branches:

First, casting directly from one struct type to another struct type is disallowed because handling it is quite cumbersome—imagine the trouble if the source and target struct types have member variables with identical names. However, this is not a major restriction, as it can easily be achieved by chaining two `static_cast` statements.

Second, converting from `type_ptr` to a heap pointer of `struct S`. Naturally, we first declare the variables for the struct's fields—meaning variable names understood by the logic language, such as `s.x` rather than just `s`. Then we perform additions sequentially, accumulating the heap offset starting from the input pointer position.

Third, converting from a heap pointer of `struct S` to `type_ptr`. Similarly, we first declare the variable name in the logic language and then copy the value of the first member variable of `s`.

Note that because bstyle's syntax has no explicit pointer type like `S*`, heap variables and stack variables differ only in how they are accessed. The compiler does not care whether a variable actually points to heap memory, so the third branch can be misused. This is somewhat analogous to the following C++ code:

```cpp
void main() {
  int i = 100;
  int* j = i;
  cout << *j << endl;
} 
```

All that remains is adding a few more tests, mainly these four:

[service/tests/compiler/b2style/self-test/static_cast_ptr_type.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type.txt)

[service/tests/compiler/b2style/self-test/static_cast_ptr_type2.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type2.txt)

[service/tests/compiler/b2style/self-test/static_cast_ptr_type3.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type3.txt)

[service/tests/compiler/b2style/self-test/static_cast_ptr_type_bool.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/static_cast_ptr_type_bool.txt)

The main changes are in [commit/5be8086060145747a93350b51922b64e7aa28175](https://github.com/Hzj-jie/osi.net/commit/5be8086060145747a93350b51922b64e7aa28175).
