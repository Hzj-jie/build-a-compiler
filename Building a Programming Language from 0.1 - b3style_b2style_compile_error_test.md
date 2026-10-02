# Building a Programming Language from 0.1 - b3style_b2style_compile_error_test

After completing template syntax analysis in `b3style`, detecting compilation errors should at least be straightforward. Indeed, it was simple; the only noteworthy section of code is in [service/compiler/b3style/logic/nodes/class_initializer.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b3style/logic/nodes/class_initializer.vb):

```diff
 Partial Public NotInheritable Class b3style
     Private NotInheritable Class class_initializer
         Implements code_gen(Of logic_writer)
...
         Public Shared Sub destruct(ByVal name As String, ByVal o As logic_writer)
             ' Functions are always in global namespace.
             ' Use of this function should only be in the !_disable_namespace.
             scope.current().when_end_scope(
                 Sub()
                     value_list.with_empty()
-                    assert(function_call.ignore_parameters.without_return(b2style.function_call.build_struct_function(
+                    ' when_end_scope uses IDisposable, even the build failed in the middle, it will still be executed
+                    ' and may cause the crash if reinterpret_cast is used. See
+                    ' b2style_compile_error_test.reinterpret_cast_without_type_id.
+                    ' Likely the destructor shouldn't be executed in the case.
+                    ' TODO: Avoid running this when_end_scope callback if the build failed in the middle.
+                    If Not function_call.ignore_parameters.without_return(b2style.function_call.build_struct_function(
                                                                                name,
                                                                                scope.class_def.destruct),
-                                                                        o))
+                                                                          o) Then
+                        raise_error(error_type.user,
+                                    "Failed to build the destructor of the variable ",
+                                    name,
+                                    ", this shouldn't happen unless the reinterpret_cast is used ",
+                                    "which changed the definition of the variable.")
+                    End If
```

There is an extensive comment here explaining the issue. Previously, it was assumed that destructors could always be successfully generated, hence the use of an assertion. However, in the implementation details, this code executes inside `scope.current().when_end_scope`, which relies on `IDisposable`. Even if compilation fails halfway through, this cleanup code still runs. If compilation failed because a type is not actually a class, `b2style.function_call.build_struct_function` will also fail.

In theory, two cases can trigger this. The first and simpler case is when the type is not a class at all, as illustrated in [b2style_compile_error_test.reinterpret_cast_without_type_id](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/errors/reinterpret_cast_without_type_id.txt):

```cpp
#include <b2style.h>
#include <b2style/stdio.h>

class S {
  int x;
  void print() {
    b2style::std_out(this.x);
  }
};

struct S2 {
  int x;
};

// This function should not be used.
void f(S& s) {
  s.x = 0;
}

void f(S2& s) {
  s.x = 100;
}

void main() {
  S s;
  s.x = -100;
  s.print();
  reinterpret_cast(s, S2);
  f(s);  // s.S2__struct__type__id should be missing here.
  reinterpret_cast(s, S);
  s.print();
}
```

The issue here is that upon the first `reinterpret_cast(s, S2)`, type `S` cannot naturally convert to type `S2` because `s.S2__struct__type__id` does not exist—only `s.S__struct__type__id` is present. See [reinterpret_cast.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/reinterpret_cast.txt) for proper usage:

```cpp
void main() {
  S s;
  s.x = -100;
  s.print();
  S2__struct__type__id__type s.S2__struct__type__id;
  // Note: this is not a typical way of using reinterpret_cast, it's pretty much useless without the support of // inheritance.
  reinterpret_cast(s, S2);
  f(s);
  reinterpret_cast(s, S);
  s.print();
}
```

Even when compilation errors occur, in `b3style`, the destructor is still queued for generation; since `s.S2__struct__type__id` does not exist, `function_call.ignore_parameters.without_return` fails.

A simpler example, [reinterpret_cast_to_a_different_class_type](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/reinterpret_cast_to_a_different_class_type.txt), uses `reinterpret_cast` and `undefine` to produce code that `b2style` could compile, while `b3style`, which triggers destructors, correctly detects the compilation error:

```cpp
#include <b2style.h>

class S {
  int x;
};

struct S2 {
  int x;
};

void main() {
  S s;
  S2__struct__type__id__type s.S2__struct__type__id; 
  reinterpret_cast(s, S2);
  undefine(s.S__struct__type__id);
  // On b3style, it will also complain the destruction of s as S2 cannot be generated.
  // b2style would only complain the S2__struct__type__id hasn't been defined.
}
```

`b2style_test`:

```vb.net
    <test>
    Private Shared Sub reinterpret_cast_to_a_different_class_type()
        Dim e As executor = Nothing
        assertion.is_true(parse(New console_io.test_wrapper(),
                                _b2style_test_data.reinterpret_cast_to_a_different_class_type.as_text(),
                                e))
        assertion.is_not_null(e)
        e.assert_execute_without_errors()
    End Sub
```

`b3style_b2style_compile_error_test`:

```vb.net
    <test>
    Private Shared Sub reinterpret_cast_to_a_different_class_type()
        ' b3style can detect the error correctly.
        run(_b2style_test_data.reinterpret_cast_to_a_different_class_type,
            "s.::S2__struct__type__id",
            b3style.class_initializer.failed_to_build_destructor_message("s"))
    End Sub
```
