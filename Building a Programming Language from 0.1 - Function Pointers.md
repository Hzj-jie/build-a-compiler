# Building a Programming Language from 0.1 - Function Pointers

Function pointers are also an essential part of class inheritance, used to implement function inheritance, which we will discuss when the time comes. Since this is a relatively broad concept in itself, let's work through it layer by layer.

## The primitive Processor

First, let's recall the `jump` instruction. This instruction jumps to a specific instruction position—which is itself a constant—and can be used for function calls because a function's entry point is fixed in the generated instruction sequence. Function pointers, however, are different: the target position to jump to is a value stored at a memory address. Therefore, we first need a new instruction, similar to `jmp eax` in the x86 instruction set.

We define this instruction as:

```text
jmpr	data_ref	#jump to instruction @(*A int64)
jump	data_ref	#jump to instruction @(A int64) 

```

Compared to the `jump` instruction, which uses `data_ref` itself, `jmpr` uses the value in the memory pointed to by `data_ref`. A similar comparison is `cp` versus `cpc`: the former copies the value at one address to another address, while the latter copies a constant to an address.

When adding instructions in `primitive`, the vast majority of the code is auto-generated; the crucial part is how to execute the `jmpr` instruction itself.

```vb.net
Partial Public NotInheritable Class [jmpr]
    Implements instruction

    Public Sub execute(ByVal imi As imitation) Implements instruction.execute
        Dim s As Int64 = imi.access_as_int64(d0)
        imi.instruction_ref(s)
        imi.do_not_advance_instruction_ref()
    End Sub
End Class

```

These changes are mainly in [commit/612d61a6174f72b7ba5802b028be1bdeb12d3d71](https://github.com/Hzj-jie/osi.net/commit/612d61a6174f72b7ba5802b028be1bdeb12d3d71#diff-cc1575e4ee760253cf84c598c76a7e7a17ee08b4b62b62428fa4b7e9b58e1503R529), which also includes the changes to the `logic` language.

## The logic Language

`jmpr` only provides the ability to jump at the processor level. For truly usable function pointers, several more questions need to be addressed.

- How to define a function pointer

When defining and calling a function, we need to record the function signature to ensure that the parameter and return types match expectations at the call site; function pointers have a similar requirement. Of course, one could simply use a `uint` to store the function address and accept whatever arguments the caller provides, but doing so would generally leave the locations of parameters and return values meaningless inside the function and make it impossible to detect errors, so that approach is not viable.

Simply put, we need to define a function pointer type to specify the parameter list and return type. Using this type, we can define multiple pointers, while the pointers themselves are simply of type `uint`.

These operations should be implemented via a `callee_ref` statement. Similar to `callee`, `callee_ref` also has a name, a return type, and a parameter list, except that the parameter list only requires types and not variable names.

```text
# define a callee / function reference with name, parameters:
#  callee_ref f int ( a uint )
#  a type suffixed by "&" indicates that it's a "reference".
callee_ref	string	string	parameters

```

There is a slight mistake in the comment here: it should just be `( uint )`.

Compare this with the definition of `callee`, which includes a `paragraph`:

```text
# define a callee / function with name, parameters and body:
#  callee f int (a uint) { return f a; }
#  a type suffixed by "&" indicates that it's a "reference".
callee	string	string	typed_parameters	paragraph

```

- How to bind a function pointer

As mentioned above, a function pointer stores the entry position of a function. At the `primitive` level, this storage process corresponds to just a `cpc` instruction, but in the `logic` language we need to: 1) check whether the signatures of the function pointer and the function itself match, 2) find the entry position of the function, and 3) generate the `cpc` instruction.

The operations above can be implemented by a `logic` statement called `address_of`:

```text
# copy the position of a function:
#  address_of target source
address_of	string	string

```

- How to call a function pointer

Similarly to the `caller` statements, there are two new `caller_ref` statements:

```text
# execute a function based on its address with parameters and store its return value:
#  caller_ref f return-value (p1, p2)
caller_ref	string	string	parameters

# execute a function based on its address with parameters and discard the return value:
#  caller_ref f (p1, p2)
caller_ref	string	parameters

```

The first retains the return value, while the second discards it.

Just like `primitive` instructions, most of the code for these statements is auto-generated; all that needs to be implemented is the functionality of these few statements themselves.

The main implementations are in [osi.net/commit/612d61a6174f72b7ba5802b028be1bdeb12d3d71](https://github.com/Hzj-jie/osi.net/commit/612d61a6174f72b7ba5802b028be1bdeb12d3d71#), [osi.net/commit/e862c6dc3534cb356a770ab3a0d637d8f88ef825](https://github.com/Hzj-jie/osi.net/commit/e862c6dc3534cb356a770ab3a0d637d8f88ef825), and [osi.net/commit/3694ff299679cc0a1b824f345bd4c0e04075f5d2](https://github.com/Hzj-jie/osi.net/commit/3694ff299679cc0a1b824f345bd4c0e04075f5d2), along with some miscellaneous changes.

### logic/scope.anchor_ref

Like `anchor` in `logic`, `callee_ref` also needs to record its definition. A slight difference is that in `anchor`, each function has a single unique entry point, whereas in `anchor_ref`, a single `callee_ref` can be bound to multiple variables.

- The implementation in [Implement interrupter/primitive.jmpr and logic.address_of/callee_ref/… · Hzj-jie/osi.net@612d61a](https://github.com/Hzj-jie/osi.net/commit/612d61a6174f72b7ba5802b028be1bdeb12d3d71#) was similar to C function pointers, where each `callee_ref` was a single unique pointer, but that approach would become rather cumbersome later on—we will return to this when discussing the `b2style` implementation.

Therefore, `anchor_ref` naturally has two dictionaries: one storing definitions and one storing variables. Variable storage is similar to that in `scope.variable`: `anchor_ref` stores a variable's function pointer type, while `scope.variable` always stores the `uint` type.

Let's first look at the definition of `anchor_ref` itself. [osi.net/scope.anchor_ref.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope.anchor_ref.vb)

```vb.net
Public NotInheritable Class anchor_ref
    Inherits function_signature

    Public Sub New(ByVal name As String,
                   ByVal return_type As String,
                   ByVal parameters() As builders.parameter_type)
        MyBase.New(name, return_type, parameters)
    End Sub

    Public Sub New(ByVal name As String,
                   ByVal return_type As String,
                   ByVal parameters As const_array(Of builders.parameter_type))
        MyBase.New(name, return_type, parameters)
    End Sub

    Public Function with_begin(ByVal begin As Func(Of data_ref)) As anchor
        Return New anchor(name, lazier.of(begin), return_type, parameters)
    End Function
End Class

```

`anchor_ref` is a `function_signature` with a return type, a name, and a parameter list. However, it can be converted into a callable `anchor` via the `with_begin` function.

`scope.anchor_refs()` holds all function pointer definitions and pointer variables.

```vb.net
Private NotInheritable Class anchor_ref_t
    ' type -> anchor_ref
    Private ReadOnly decls As New unordered_map(Of String, anchor_ref)()
    ' variable name -> anchor_ref
    Private ReadOnly defs As New unordered_map(Of String, anchor_ref)()

```

The former is used when defining a `callee_ref`, while the latter is used in `define`, `address_of`, and `caller_ref`.

### logic/callee_ref

This is quite simple: it just forwards the definition to `scope.current().anchor_refs()`. [service/compiler/logic/callee_ref.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/callee_ref.vb)

```vb.net
Public Function build(ByVal o As vector(Of String)) As Boolean Implements instruction_gen.build
    Return scope.current().anchor_refs().decl(name, return_type, parameters)
End Function

```

### logic/address_of

As mentioned above, this statement needs to do three things: [service/compiler/logic/address_of.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/address_of.vb)

```text
' TODO: Check the consistency of both signatures.
Public Function build(ByVal o As vector(Of String)) As Boolean Implements instruction_gen.build
    Dim t As variable = Nothing
    If Not variable.of(target, o, t) OrElse
       Not t.is_assignable_from_uint32() Then
        Return False
    End If
    Dim a As scope.anchor = Nothing
    If Not scope.current().anchors().of(name, a) Then
        errors.anchor_undefined(name)
        Return False
    End If
    assert((+a.begin).on_stack())
    assert((+a.begin).absolute())
    o.emplace_back(instruction_builder.str(command.cpc, t, New data_block((+a.begin).offset())))
    Return True
End Function

```

First, look up the target variable, which must be at least `uint32`.

Next, look up the target `anchor`.

Finally, generate the `cpc` instruction to write the `anchor` address into the target variable.

Of course, there could also be a fourth step: checking whether the definitions of `anchor` and `anchor_ref` match. However, that is not critical and has not been implemented yet.

### logic/caller_ref

`caller_ref` invokes `jmpr` to perform the jump. It is essentially the same as `caller` and shares a large amount of code, with only two differences: 1) the instruction (`jump` vs. `jmpr`), and 2) how the `anchor` is obtained (`caller_ref` uses `anchor_ref.with_begin`, whereas `caller` looks up `scope.anchor` directly).

[osi.net/caller_ref.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/caller_ref.vb)

```vb.net
Public Sub New(ByVal name As String,
               ByVal result As String,
               ParamArray ByVal parameters() As String)
    MyBase.New(command.jmpr, result, parameters)
    assert(Not name.null_or_whitespace())
    Me.name = name
End Sub

Protected Overrides Function retrieve_anchor(ByVal o As vector(Of String),
                                             ByRef anchor As scope.anchor) As Boolean
    Dim f As Func(Of scope.variable_t.exported_ref) =
        Function() As scope.variable_t.exported_ref
            Dim d As scope.variable_t.exported_ref = Nothing
            If Not scope.current().variables().export(name, d) Then
                Return Nothing
            End If
            Return d
        End Function
    If f() Is Nothing Then
        errors.anchor_ref_undefined(name)
        Return False
    End If
    Dim r As scope.anchor_ref = Nothing
    If Not scope.current().anchor_refs().of(name, r) Then
        Return False
    End If
    anchor = r.with_begin(Function() As data_ref
                              Return f().data_ref
                          End Function)
    Return True
End Function

```

### logic/define

If a `define` statement defines a function pointer variable, its behavior differs slightly: first, it defines a `uint32` variable, and second, it forwards the definition to `scope.anchor_ref`.

```vb.net
 Private Function define_callee_ref(ByVal o As vector(Of String)) As Boolean
    Return define_variable(scope.type_t.ptr_type, o) AndAlso
           scope.current().anchor_refs().define(type, name)
End Function

Public Function build(ByVal o As vector(Of String)) As Boolean Implements instruction_gen.build
    If define_variable(type, o) OrElse define_callee_ref(o) Then
        Return True
    End If
    errors.type_undefined(type, name)
    Return False
End Function

```

### Testing

With this implementation in place, function pointers now work in the `logic` language.

[service/tests/compiler/logic/codegen/address_of.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/logic/codegen/address_of.txt)

[service/tests/compiler/logic/codegen/address_of_address_of_param.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/logic/codegen/address_of_address_of_param.txt)

[service/tests/compiler/logic/codegen/address_of_param.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/logic/codegen/address_of_param.txt)

[service/tests/compiler/logic/codegen/address_of_ref.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/logic/codegen/address_of_ref.txt)

These are all test cases, which look roughly like this:

```text
type uint 4
define 1 uint
copy_const 1 i1

callee m uint ( n uint ) {
  add n n 1
  return m n
}

callee m2 uint ( n uint ) {
  subtract n n 1
  return m2 n
}

callee_ref delegate uint ( uint )
define f delegate
define n uint
define temp_string type*

address_of f m
caller_ref f n ( 1 ) 
interrupt stdout n temp_string

address_of f m2
caller_ref f n ( n )
interrupt stdout n temp_string

```
