# Building a Programming Language from 0.1 - Function Parameter References in logic

Earlier articles mentioned that neither `bstyle` nor `b2style` supported function parameter references—all function parameters were passed by value. If we want to support classes, this is obviously insufficient: class methods cannot modify the state of a class instance under pass-by-value, which severely limits what classes can do. Therefore, to implement meaningful class functionality, function parameter references are essential. At the same time, as mentioned earlier, `i++` and `++i` could not modify the value of `i` itself, which can likewise be implemented via parameter references.

First, let's recall how functions are compiled. At function entry, the function assumes that the current top of the stack and several stack slots beneath it hold the function's parameters and return value. Specific implementation details may vary slightly—for example, whether a `void` function also reserves a slot for a return value, or the order in which parameters are pushed onto the stack. Those are just details; the fundamental principle is that a function itself cannot possibly know the stack locations of the arguments used by the caller at the call site, so the caller pushes all arguments onto the stack for the function to use.

For the exact same reason, a function cannot directly access the caller's stack frame, even though it is in fact the same stack, just at a different offset.

However, a parameter reference—or an in/out parameter—requires that the argument value can serve as input to the function while any modifications made to it during function execution are reflected back to the caller's variable. For example:

```c
void f(int x) { x++; }
void f2(int& x) { x++; }
void main() {
  int y = 100;
  f(y);
  ASSERT_EQ(y, 100);
  f2(y);
  ASSERT_EQ(y, 101);:
}

```

Here, modifications to `x` inside `f2` are reflected back to the passed-in argument, `y` in `main`, whereas modifications to `x` inside `f` do not have this effect.

Whether in `f` or `f2`, `x` is located at the top of the stack (assuming a `void` return value does not occupy a stack slot), and neither function can directly manipulate `main`'s variable `y`, because neither knows where `main`'s variable `y` is actually located on the stack.

Simply put, there are two ways to implement parameter references:

1. Pass the location of the argument rather than the argument value itself. In this case, a reference is just a specific use of a pointer.
2. Consider how function return values work: reference parameters can be implemented in the exact same way. After the function call finishes, before popping the stack, the caller copies the stack contents back into the variable used as the argument.

First of all, as mentioned earlier, `bstyle` and `b2style` cannot implement certain features that are not supported by the underlying `logic` language. Parameter references fall into this category, so they need to be implemented in the `logic` language.

Now let's compare these two implementation approaches. The first approach requires implementing a special pointer type, while the second approach is relatively simple and does not require adding a dedicated type. Furthermore, if we ever wanted to support properties—where the caller needs to invoke the property getter and setter functions of the argument—only the second approach would work:

```cpp
// Suppose property is implemented in C++.
class C {
  public:
    void property set_x(int x) {
      this->x = x;
    }
    int property get_x() {
      return this->x;
    }
  private:
    int x;
};

void f(int& x) { x++; }

int main() {
  C c;
  c.x() = 1;
  f(c.x());
  /*
    Equivalent to:
    int x = c.get_x();
    f(x);
    c.set_x(x);
  */
  ASSERT_EQ(c.x(), 2);
}

```

Although there are currently no plans to implement properties, I still chose the second approach.

As stated above, this feature is implemented in the `logic` language and only requires modifying the caller, which can be seen in [osi.net/commit/4d12dc2fba69e206b2ce39afdcf6481de3b4277f](https://github.com/Hzj-jie/osi.net/commit/4d12dc2fba69e206b2ce39afdcf6481de3b4277f).

Two main files were modified: [service/compiler/logic/builder/builders.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/builder/builders.vb) and [service/compiler/logic/caller.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/caller.vb).

(There were some other largely unrelated changes later; the links to these two files point to the latest version.)

In `builders.vb`, we need to know whether a parameter in a function definition is a reference. Here, the same `&` notation as in C++ is used.

```vb.net
Public NotInheritable Class parameter
' ...
    Public ReadOnly ref As Boolean

    Public Sub New(ByVal type As String, ByVal name As String)
        assert(Not type Is Nothing)
        assert(Not name Is Nothing)
        type = type.Trim()
        name = name.Trim()
        assert(Not type.null_or_whitespace())
        assert(Not name.null_or_whitespace())
        'TODO: Try to avoid allowing only "&" as parameter type.
        'assert(Not type.Equals(type_ref_suffix))
'----------------------------
        Me.ref = type.EndsWith(type_ref_suffix) AndAlso Not type.Equals(type_ref_suffix)
        If Me.ref Then
            Me.type = type.Remove(type.Length() - 1)
        Else
            Me.type = type
        End If
'-----------------------------
        Me.name = name
    End Sub

```

The key change is the section enclosed by the separator lines: a reference does not constitute a distinct type by itself, so the reference symbol is stripped from the type name while recording whether this parameter is a reference.

The more involved part is in `caller.vb`:

```vb.net
Public NotInheritable Class caller
' ...
    ' Forward return-value from scope_wrapper.
    Private Function forward_to_result(ByVal anchor As anchor, ByVal o As vector(Of String)) As Boolean
        assert(Not anchor Is Nothing)
        assert(Not o Is Nothing)
        Dim callee_params As const_array(Of builders.parameter) = Nothing
        assert(anchor.parameters(callee_params))
        assert(array_size(parameters) = callee_params.size())
'----------------------------
        For i As Int32 = 0 To parameters.array_size_i() - 1
            If Not callee_params(CUInt(i)).ref Then
                Continue For
            End If
            Dim parameter_place_holder As variable = Nothing
            assert(variable.of(types, parameter_place_holder_of(i), o, parameter_place_holder))
            Dim parameter As variable = Nothing
            assert(variable.of(types, parameters(i), o, parameter))
            If Not move.export(parameter, parameter_place_holder, o) Then
                Return False
            End If
        Next
'----------------------------
        If Not result Then
            Return True
        End If
        Dim result_var As variable = Nothing
        If Not variable.of(types, +result, o, result_var) Then
            errors.variable_undefined(+result)
            Return False
        End If
        Dim return_value_var As variable = Nothing
        assert(return_value.retrieve(types, name, o, return_value_var))
        Return move.export(result_var, return_value_var, o)
    End Function

```

Again, the key part is enclosed by separator lines. The operation is essentially identical to handling the return value: if a parameter is a reference, copy the result back to the variable used as the argument.

In addition to this, there is a small optimization. Because the `primitive` processor is much more efficient at `mov` operations than `cp`, if a parameter is a reference, its original value is moved to the top of the stack when passed in rather than copied as before. Of course, this is purely an optimization and does not affect the result.

```vb.net
Public NotInheritable Class caller
' ...
    ' Forward parameters to scope_wrapper.
    Private Function define_parameters(ByVal anchor As anchor,
                                       ByVal o As vector(Of String)) As Boolean
    ' ...
            If callee_params(CUInt(i)).ref Then
                If Not move.export(target, vars(i), o) Then
                    Return False
                End If
            Else
                If Not copy.export(target, vars(i), o) Then
                    Return False
                End If
            End If
        Next
        Return True
    End Function

```

When implementing `return` in `logic` previously, we assumed the return value was always a temporary variable and therefore used `mov` as well. However, this causes a bug when returning a reference variable:

```text
callee self_increment&Integer Integer ( x Integer& ) {
    define temp_value_@1@34417-34418 Integer copy_const temp_value_@1@34417-34418 a01000000
    define temp_value_@1@34398-34420 Integer
    caller b2style__add&Integer&Integer temp_value_@1@34398-34420 ( x temp_value_@1@34417-34418 )
    move x temp_value_@1@34398-34420
    return self_increment&Integer x
}

```

Here, the final `return` moves the value of `x` into the stack slot for the return value, which clears out the reference parameter `x`. Therefore, a small modification to `return` was made to avoid this issue: [osi.net/commit/ec823c35e1cf50aa45efc4050c308785d16a20c2](https://github.com/Hzj-jie/osi.net/commit/ec823c35e1cf50aa45efc4050c308785d16a20c2)

```vb.net
Public NotInheritable Class [return]
' ...
    Public Function export(ByVal o As vector(Of String)) As Boolean Implements exportable.export
        assert(Not o Is Nothing)
        Dim r As variable = Nothing
        If Not logic.return_value.retrieve(types, name, r) Then
            Return False
        End If
        assert(r.size)
        If return_value Then
            Dim var As variable = Nothing
            If Not variable.of_stack(types, +return_value, var) Then
                Return False
            End If
-           If Not move.export(r, var, o) Then
+           ' TODO: Check if +return_value is a temporary variable or a reference to decide whether move can be
+           ' used.
+           If Not copy.export(r, var, o) Then
                Return False
            End If
        Else

```

Finally, let's add a `logic` test:

```text
callee m type0 ( n type*& i type* ) {
  append n i
}

define n type*
define i type*
copy_const i sd

caller m ( n i )
caller m ( n i )
caller m ( n i )
caller m ( n i )

define temp_string type*
interrupt stdout n temp_string

```

This test uses a function named `m`, where the first parameter `n` is a reference, and uses the `append` instruction to concatenate the second parameter `i` onto the end of `n`. Initially, the value of `i` is the character `d`. After calling `m` four times, the resulting value of `n` should be four `d` characters, which is output via the `stdout` interrupt.

```vb.net
Public NotInheritable Class import_executor_callee_ref
    Inherits import_executor_case

    Private Shared ReadOnly io As New console_io.test_wrapper()

    Public Sub New()
        MyBase.New(_import_executor_cases.callee_ref.as_text(), New interrupts(+io))
    End Sub

    Protected Overrides Sub check_result(ByVal e As not_null(Of simulator))
        MyBase.check_result(e)
        assertion.equal(io.output(), New String(Convert.ToChar(100), 4))
    End Sub
End Class

```

With this, function parameter references in `logic` are implemented. However, using them in `bstyle` or `b2style` requires additional changes and involves the distinction between lvalues and rvalues. Although there aren't many changes, we will explain them in detail later.
