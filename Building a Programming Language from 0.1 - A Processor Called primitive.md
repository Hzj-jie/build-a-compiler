# Building a Programming Language from 0.1 - A Processor Called primitive

As an intermediate representation, it needs to be easy to execute and extend. Of course, compared to existing languages, it should also have some distinctive characteristics of its own. With that in mind, I designed the following virtual "processor". The concept of this processor comprises instructions, registers, and memory.

An instruction consists of an operation and zero or more operands—for example, a copy operation that copies A to B. Operands fall into two categories: references and constants. A reference addresses a piece of data on the memory stack, whereas a constant is a way to embed literal values directly in the intermediate representation.

Registers track specific aspects of the processor's state, such as the currently executing instruction.

Memory is divided into three parts:

1. Instructions (read-only): simply an array of instructions.
2. Data stack: the memory space that instructions can manipulate. Each slot on the stack holds a variable-length byte array.
3. State stack: saves current register values for state switching.

A data reference (`data_ref`) identifies a position on the data stack in memory. There are two addressing modes: relative and absolute. Relative addressing specifies the distance from the top of the stack to the referenced data—for example, `rel1` refers to the slot right below the top of the stack. Absolute addressing specifies the index of the data from the bottom of the data stack—for example, `abs2` refers to the third slot on the stack.

A constant (`data_block`) is a simple byte array, which can represent values such as a string.

Registers can only be affected as a side effect of instruction execution; instructions cannot manipulate registers directly.

1. `IP` (Instruction Pointer, `uint64`): the index of the currently executing instruction in the instruction array. Unless the Do-Not-Advance-Instruction-Pointer (`DNAIP`) register is set to `true`, executing each instruction increments this register by 1.
2. `CO` (Carry Over, `boolean`): indicates whether the previous instruction resulted in a carry or borrow. Arithmetic instructions update this register.
3. `DBZ` (Divided By Zero, `boolean`): indicates whether the previous instruction caused a division-by-zero error. Division instructions update this register.
4. `IN` (Imaginary Number, `boolean`): indicates whether the previous instruction resulted in an imaginary number error. Root-extraction instructions update this register.
5. `DNAIP` (Do Not Advance Instruction Pointer, `boolean`, write-only): if set to `true`, the `IP` register will not be incremented after the current instruction finishes executing, and `DNAIP` is automatically reset to `false`.

Instruction Set:

```text
push	#push a data slot in to stack
pop	#pop a data slot from stack
jump	data_ref	#jump to instruction @A
cpc	data_ref	data_block	#(*A) = B
mov	data_ref	data_ref	#(*A) = (*B), clr B
cp	data_ref	data_ref	#(*A) = (*B)
add	data_ref	data_ref	data_ref	#(*A big_uint) = (*B big_uint) + (*C big_uint)
sub	data_ref	data_ref	data_ref	#(*A big_uint) = (*B big_uint) - (*C big_uint)
mul	data_ref	data_ref	data_ref	#(*A big_uint) = (*B big_uint) * (*C big_uint)
div	data_ref	data_ref	data_ref	data_ref	#(*A big_uint) = (*C big_uint) / (*D big_uint) (*B big_uint) = (*C big_uint) % (*D big_uint)
ext	data_ref	data_ref	data_ref	data_ref	#(*A big_uint) = extract((*C big_uint), (*D big_uint)) (*B big_uint) = remainder(extract((*C big_uint), (*D big_uint)))
pow	data_ref	data_ref	data_ref	#(*A big_uint) = pow((*B big_uint), (*C big_uint))
jumpif	data_ref	data_ref	#jump to instruction @A, if (*B bool) is true
cpco	data_ref	#(*A) = CO
cpdbz	data_ref	#(*A) = DBZ
cpin	data_ref	#(*A) = IN
stop	#finish execution, and leave the executor in a stop but not error state.
equal	data_ref	data_ref	data_ref	#(*A) = ((*B big_uint) == (*C big_uint))
less	data_ref	data_ref	data_ref	#(*A) = ((*B big_uint) < (*C big_uint))
app	data_ref	data_ref	#(*A) += (*B)
sapp	data_ref	data_ref	#(*A) += sizeof(*B) + (*B)
cut	data_ref	data_ref	data_ref	#(*A) = sub-array(src=(*B), from=(*C uint))
cutl	data_ref	data_ref	data_ref	data_ref	#(*A) = sub-array(src=(*B), from=(*C uint), len=(*D uint))
int	data_ref	data_ref	data_ref	#execute an interrupt function (*A uint), with parameter (*B), return value will be set to (*C)
clr	data_ref	#set (*A) to empty array
scut	data_ref	data_ref	data_ref	#(*A) = the (*C uint) chunk of (*B)
sizeof	data_ref	data_ref	#(*A) = array-size(*B)
empty	data_ref	data_ref	#(*A) = ((*B) == null)
and	data_ref	data_ref	data_ref	#(*A big_uint) = (*B big_uint) AND (*C big_uint)
or	data_ref	data_ref	data_ref	#(*A big_uint) = (*B big_uint) OR (*C big_uint)
not	data_ref	data_ref	#(*A big_uint) = NOT (*B big_uint)
stst	#push current state into states, such as IP_AFTER_THIS_INSTRUCTION, i.e. IP + 1, Stack Size, a typical usage is [ stst, jump ? ]
rest	#pop extra (StackSize - states.top.StackSize) slots from stack, and jump to instruction @(* states.top.IP uint64)

```

Besides the standard instructions, `int` provides a simple mechanism to invoke functionality that cannot be expressed using regular instructions alone, such as reading from or writing to standard I/O streams.

Interrupts are invoked using specific unsigned integers, so interrupt numbers cannot be changed arbitrarily and form part of the processor specification.

```text
0 stdout, write input to stdout and return null
1 stderr, write input to stderr and return null
2 stdin, read from stdin and return the byte-array
3 current_ms, return the current milliseconds as int64 byte-array

```

Unlike typical hardware processors, each slot on the `primitive` processor's memory stack has a variable size, and all arithmetic operations natively support arbitrary-precision integers.
