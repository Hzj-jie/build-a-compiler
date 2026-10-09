# Building a Programming Language from 0.1 - Delegates and Their Templates

I mentioned earlier that I am not a fan of C's function pointer syntax:

```cpp
void (*fun_ptr)(int) = &fun;

void (*fun_ptr2)(int) = NULL;
fun_ptr = &fun;

```

There are two issues with it: 1) the function pointer type definition and the variable declaration are combined, leading to a lot of repetition—for example, `fun_ptr` and `fun_ptr2` above share the exact same type definition; and 2) the syntax is awkward—the parentheses around the name and the leading asterisk feel somewhat arbitrary and out of place alongside the rest of the grammar. While the second issue is a matter of personal taste, the first is a serious concern: more repeated definitions mean more code to compile.

Of course, I could imitate `std::function`, but that requires class inheritance. Yet function pointers are meant to implement class inheritance in the first place, turning it into a chicken-and-egg problem—so that won't work either.

As a compromise, we can implement a feature similar to function pointers that separates type definitions from variable declarations without relying on class inheritance. Borrowing the `delegate` keyword from C#, we implement the following syntax:

```cpp
#include <bstyle.h>

delegate void func();

void f() { bstyle__putchar(102); }
void g() { bstyle__putchar(103); }

void main() {
  func x;
  x = f;
  x();
  x = g;
  x();
} 

```

Notice that compared to the `logic` language, the `bstyle` implementation introduces no new syntax other than `delegate` itself. Consequently, `bstyle` has to handle more context: for example, checking whether a variable `x` is a function pointer variable, and when assigning to it, determining whether the right-hand side of `=` is a function, another function pointer, or an unrelated integer so that it can emit the appropriate `logic` statement. The same applies when calling a function pointer: since the call syntax looks identical to a regular function call, `bstyle` must check the variable's definition to decide whether to emit a `caller_ref` statement.

Let's start with the new grammar rules, which, as before, are straightforward:

```text
# nlexer_rule
kw-delegate delegate

# syntaxer_rule
- root-type-with-semi-colon [..., typedef]? semi-colon
+ root-type-with-semi-colon [..., typedef, delegate]? semi-colon
- base-sentence-with-semi-colon [..., reinterpret-cast]? semi-colon
+ base-sentence-with-semi-colon [..., reinterpret-cast, delegate]? semi-colon

+ delegate kw-delegate type-name name start-bracket [paramtypelist]? end-bracket
+ paramtype type-name [reference]?
+ paramtypelist [paramtype-with-comma]* paramtype
+ paramtype-with-comma paramtype comma

```

Similar to `logic/anchor_ref`, `bstyle` has `scope.delegate`: [osi.net/scope.delegate.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.delegate.vb). One difference is that `bstyle` only needs to record the delegate types, because `scope.variable` can record the corresponding delegate type for a pointer variable directly rather than using `uint`—that is a lower-level detail for the `logic` language to care about.

```vb.net
Private NotInheritable Class delegate_t
    Private ReadOnly s As New unordered_map(Of String, function_signature)()

    Public Function define(ByVal name As String, ByVal signature As function_signature) As Boolean
        assert(Not signature Is Nothing)
        assert(Not name.null_or_whitespace())
        If s.emplace(name, signature).second() Then
            Return True
        End If
        raise_error(error_type.user, "Delegate type ", name, " has been defined already as ", s(name))
        Return False
    End Function

    Public Function retrieve(ByVal name As String, ByRef o As function_signature) As Boolean
        assert(Not name.null_or_whitespace())
        Return s.find(name, o)
    End Function
End Class

```

When processing a `delegate` node, `define` is called to record the function pointer signature. [osi.net/delegate.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/delegate.vb)

When looking up a variable, an optional output parameter can be provided to indicate whether the variable is a function pointer, populated via `retrieve`. [service/compiler/bstyle/scope.variable.vb#L122](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb#L122)

```vb.net
Public Function try_resolve(ByVal name As String,
                            ByRef type As String,
                            ByVal signature As ref(Of function_signature)) As Boolean
    ' logic_name.of_function_call requires type of the parameter to set function name.
    If variable.is_heap_name(name) Then
        name = heap_name_of(name.Substring(0, name.IndexOf(character.left_mid_bracket)))
    End If

    Dim s As scope = Me.s
    While Not s Is Nothing
        If Not s.v.resolve(name, type) Then
            s = s.parent
            Continue While
        End If
        If Not signature Is Nothing Then
            Dim f As function_signature = Nothing
            If s.delegates().retrieve(type, f) Then
                assert(Not f Is Nothing)
                signature.set(f)
            End If
        End If
        Return True
    End While
    Return False
End Function

```

This overload taking `ref(Of function_signature)` is used in `function_call` and `value_clause`: the former decides between `caller` and `caller_ref`, while the latter decides between `address_of` and `copy`.

[service/compiler/bstyle/logic/nodes/function_call.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function_call.vb)

```vb.net
Private Function build(ByVal n As typed_node,
                       ByVal o As writer,
                       ByVal build_caller As Func(Of String, vector(Of String), Boolean),
                       ByVal build_caller_ref As Func(Of String, vector(Of String), Boolean)) As Boolean
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    assert(Not build_caller Is Nothing)
    assert(n.child_count() >= 3)
    Dim bc As Func(Of vector(Of String), Boolean) =
        Function(ByVal parameters As vector(Of String)) As Boolean
            If scope.current().variables().try_resolve(n.child(0).input_without_ignored(), Nothing, Nothing) Then
                Return build_caller_ref(n.child(0).input_without_ignored(), parameters)
            End If

            Dim name As String = Nothing
            If Not logic_name.of_function_call(n.child(0).input_without_ignored(), parameters, name) Then
                Return False
            End If
            scope.current().call_hierarchy().to(name)
            Return build_caller(name, parameters)
        End Function

```

[service/compiler/bstyle/logic/nodes/value_clause.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_clause.vb)

```vb.net
Private Function build(ByVal name As typed_node,
                       ByVal value As typed_node,
                       ByVal struct_copy As Func(Of vector(Of String), Boolean),
                       ByVal single_data_slot_copy As Func(Of String, Boolean),
                       ByVal o As writer) As Boolean
    assert(Not name Is Nothing)
    assert(Not value Is Nothing)
    assert(Not struct_copy Is Nothing)
    assert(Not single_data_slot_copy Is Nothing)
    assert(Not o Is Nothing)
    Dim type As String = Nothing
    Dim delegate_definition As New ref(Of function_signature)()
    If Not scope.current().variables().resolve(name.input_without_ignored(), type, delegate_definition) Then
        ' Emmmm, scope.variable should log the error already.
        Return False
    End If
    If delegate_definition Then
        ' TODO: Avoid copying.
        Dim target_function_name As String = logic_name.of_function(value.input_without_ignored(),
                                                                    +delegate_definition.get().parameters)
        If scope.current().functions().is_defined(target_function_name) Then
            ' Use address-of to copy a function address to the target.
            ' TODO: Need to use logic_name here.
            scope.current().call_hierarchy().to(target_function_name)
            Return builders.of_address_of(name.input_without_ignored(), target_function_name).to(o)
        End If
        Return builders.of_copy(name.input_without_ignored(), value.input_without_ignored()).to(o)
    End If
    If Not l.of(value).build(o) Then
        Return False
    End If

```

With this in place, the example above can now be executed: [service/tests/compiler/bstyle/codegen/delegate.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/codegen/delegate.txt).

`b2style` requires almost no changes—it simply transpiles directly into `bstyle` code.

The main changes are in [Implement delegate in bstyle and b2style (#4) · Hzj-jie/osi.net@86dbb98](https://github.com/Hzj-jie/osi.net/commit/86dbb9881a823cda32c71afa3fe5b70fb99cab8c).

## Delegate Templates

Extending the previously implemented template mechanism to `delegate` gives us something close to `std::function`. Let's start with the grammar:

```text
# syntaxer_rule
- template kw-template less-than type-param-list greater-than class
+ delegate-with-semi-colon delegate semi-colon
+ template-body [class, delegate-with-semi-colon]
+ template kw-template less-than type-param-list greater-than template-body

```

Here we add an intermediate `template-body` rule, which can be either a `class` or a `delegate` + `semi-colon`.

The template implementation in `b2style` largely doesn't care what kind of node it is operating on; it simply substitutes `raw-type-name` nodes. The only thing to account for is how to extract the template's name: in `class`, the name is at the second child node, whereas in `delegate`, it happens to be at the third child node. Beyond that, nothing special is required.

The main changes are in [osi.net/commit/04ba3cb53d65ea5fa3a24ded053540a868389653](https://github.com/Hzj-jie/osi.net/commit/04ba3cb53d65ea5fa3a24ded053540a868389653).

## --remove-unused-function, Huh?

Let's look at that simplest example again:

```cpp
#include <bstyle.h>

delegate void func();

void f() { bstyle__putchar(102); }
void g() { bstyle__putchar(103); }

void main() {
  func x;
  x = f;
  x();
  x = g;
  x();
} 

```

Neither `f` nor `g` is called directly, yet the compiler still needs to know that both functions might be called so they don't get stripped by `--remove-unused-function`. Unlike direct function calls, calling through a function pointer means a function is "potentially called": because a pointer variable can be reassigned at any time to point to a different function, as soon as a pointer variable is ever assigned a function, that function must be treated as "potentially called". That is why `value_clause` contains this crucial line of code:

```vb.net
scope.current().call_hierarchy().to(target_function_name) 

```

Here, `target_function_name` is the name of the `callee` in the `logic` language.

The same functionality also needs to be implemented in `b2style`, though it can be simpler there: whenever we see

```cpp
some_type a = b;

```

where `b` is not a defined variable, we can assume it is a function. Although this heuristic is imprecise, `bstyle` will perform an exact pass to strip unused functions later anyway, so there is no real downside.

[service/compiler/b2style/scope.call_hierarchy.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/scope.call_hierarchy.vb)

```vb.net
Public Shared Function from_value_clause() As Func(Of typed_node, Boolean)
    Return Function(ByVal n As typed_node) As Boolean
               assert(Not n Is Nothing)
               ' From value-clause or value-definition
               assert(n.child_count() = 3 OrElse n.child_count() = 4)
               ' TODO: This is a very simple solution with noticable amount of false-positive cases. But it
               ' won't impact the result, cause bstyle will do the function cleanup again.
               If n.last_child().only_descendant(n) AndAlso n.type_name.Equals("raw-name") Then
                   scope.current().call_hierarchy().to(n.word().str())
               End If
               Return True
           End Function
End Function

```

## Testing

In addition to the examples discussed above, there are also:

[service/tests/compiler/b2style/codegen/delegate.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/delegate.txt)

[service/tests/compiler/b2style/codegen/delegate2.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/delegate2.txt)

[service/tests/compiler/b2style/codegen/delegate_ref.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/delegate_ref.txt)

[service/tests/compiler/b2style/codegen/delegate_template.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/delegate_template.txt)

For example:

```cpp
#include <b2style.h>

template<RT, T>
delegate RT function(T);

int inc(int i) {
  return ++i;
}

void write(string s) {
  b2style::std_out(s);
}

void self_inc(int& i) {
  i++;
}

void main() {
  function<int, int> f = inc;
  b2style::std_out(f(1));

  function<void, string> f2 = write;
  f2("abc");

  function<void, int&> f3 = self_inc;
  int i = 0;
  f3(i);
  b2style::std_out(i);
}

```
