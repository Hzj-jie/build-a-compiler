# Building a Programming Language from 0.1 - Implementing primitive Instructions

An instruction consists of two functionally orthogonal interfaces:

1. Input/Output (`exportable`)

Identical to the I/O interface of the `primitive` processor itself, supporting import and export of instructions in either text or binary format.

```text
Public Interface exportable
    Function export(ByRef b() As Byte) As Boolean
    Function import(ByVal i() As Byte, ByRef p As UInt32) As Boolean

    Function export(ByRef s As String) As Boolean
    Function import(ByVal s As vector(Of String), ByRef p As UInt32) As Boolean

    Function bytes_size() As UInt32
End Interface

```

2. Execution (`instruction`)

Operates on the virtual layer (`imitation`) to carry out the instruction's behavior.

```text
Public Interface instruction
    Inherits exportable

    Sub execute(ByVal imi As imitation)
End Interface

```

The I/O implementations can be generated directly from the instruction definitions rather than written by hand over and over. We implement a code generator in [commands-parser.cs](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/codegen/commands-parser.cs) that takes the instruction set specification [commands.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/commands.txt) as input and generates the following files:

`commands.enum`

An enum of binary instruction opcodes, ensuring each instruction has a unique numeric identifier.

`instructions`

The I/O (`exportable`) implementations for all instructions.

`instructions-exportable-test`

Unit tests for the instruction I/O implementations.

Beyond the generated code, we only need to implement the execution behavior of each instruction in [instructions.execute.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/interpreter/primitive/instructions.execute.vb).

With that, the entire `primitive` processor is complete and ready to execute all supported operations ([service/tests/interpreter/primitive/cases](https://github.com/Hzj-jie/osi.net/tree/master/service/tests/interpreter/primitive/cases)).
