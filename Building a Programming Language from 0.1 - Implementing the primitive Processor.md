# Building a Programming Language from 0.1 - Implementing the primitive Processor

The `primitive` processor consists of three functionally independent interfaces:

1. Input/Output (`exportable`)

Imports and exports instruction sequences in either text or binary format.

```text
Public Interface exportable
    Function export(ByRef b() As Byte) As Boolean
    Function import(ByVal i() As Byte, ByRef p As UInt32) As Boolean

    Function export(ByRef s As String) As Boolean
    Function import(ByVal s As vector(Of String), ByRef p As UInt32) As Boolean

    Function bytes_size() As UInt32
End Interface

```

2. Executor (`executor`)

Exposes user-facing operations: execution control, register inspection, and execution errors.

```text
Public Interface executor
    Inherits exportable

    Enum error_type
        instruction_pointer_overflow
        stack_access_out_of_boundary
        undefined_interrupt
        unsupported_feature
        invalid_buffer_size
    End Enum

    Structure state
        Public Shared ReadOnly empty As state
        Public ReadOnly instruction_pointer As UInt64
        Public ReadOnly stack_size As UInt64

        Public Sub New(ByVal instruction_pointer As UInt64, ByVal stack_size As UInt64)
            Me.instruction_pointer = instruction_pointer
            Me.stack_size = stack_size
        End Sub
    End Structure

    Function access_stack(ByVal p As data_ref) As pointer(Of Byte())
    Function stack_size() As UInt64
    Function instruction_pointer() As UInt64
    Function carry_over() As Boolean
    Function divided_by_zero() As Boolean
    Function imaginary_number() As Boolean
    Function access_states(ByVal p As UInt64) As state
    Function states_size() As UInt64

    Function halt() As Boolean
    Function errors() As vector(Of error_type)

    Sub execute()
End Interface

```

3. Virtual Layer (`imitation`)

Exposes processor primitives manipulated by individual instructions: setting registers, invoking interrupts, and halting execution.

```text
Public Interface imitation
    Inherits executor
    Function interrupts() As interrupts
    Sub push_stack()
    Sub pop_stack()
    Sub store_state()
    Sub restore_state()
    Overloads Sub instruction_pointer(ByVal v As Int64)  ' data_ref.offset() returns int64
    Overloads Sub carry_over(ByVal v As Boolean)
    Overloads Sub divided_by_zero(ByVal v As Boolean)
    Overloads Sub imaginary_number(ByVal v As Boolean)
    Sub advance_instruction_pointer(ByVal v As Int64)
    Sub do_not_advance_instruction_pointer()
    Sub [stop]()
End Interface

```

A concrete processor implementation needs to implement all three interfaces above; the actual logic is clean and straightforward. The I/O part is slightly more involved because it deals with serializing and deserializing individual instructions, which we will cover next.

The concrete implementation can be found in [simulator.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/simulator.vb).
