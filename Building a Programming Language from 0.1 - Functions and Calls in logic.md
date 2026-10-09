# Building a Programming Language from 0.1 - Functions and Calls in logic

An important feature supported by the `logic` language is functions and function calls. Put simply, functions rely on `anchors` to mark their locations, while calls jump to the locations marked by `anchors`.

A function can always be represented as the following sequence of instructions:

```text
jmp {end-of-function}
## Entry point of the function
## A sequence of instructions
rest
## end-of-function

```

The entry point of a function is the second instruction of its instruction sequence, its exit is always the `rest` instruction, and its first instruction jumps to the instruction immediately following `rest`. When a function is called, execution begins at the entry point; otherwise, the function body is skipped over. This transforms a tree-structured function call graph into a flat, tape-like sequence of instructions.

A function call can always be represented as:

```text
push  ## The first push allocates space for the return value of the function implementation.
## For simplicity, even functions with a void / type0 return type still have a return value slot.
push
cp rel0 {parameter1}
## Combinations of push and cp to set up parameters; note that pass-by-value is used here rather than pass-by-reference.
## All data is copied to a temporary location via cp, and is not moved back to the original location via mov afterward.
## Pass-by-reference could be implemented using that approach, but as an experimental language, this feature was not added.
stst  ## Save current state
jmp {begin-of-function}  ## Jump to the entry point / second instruction of the function
mov rel{number of parameters} {address-of-return-value}  ## Move the return value to the defined destination
pop  ## Multiple pop instructions to deallocate the memory space of parameters and the return value

```

This brings up a classic calling convention question: who allocates the parameters and return value, who cleans them up, and in what order are they laid out? Much has been written on this topic; in the `logic` language, I adopted the most common and reliable convention: the caller both allocates and cleans up the parameters and the return value. The exact ordering of parameters and the return value is not critical; the order used here pushes the return value first, followed by the parameters from left to right.

Theory is always simpler than implementation. In practice, we must consider how definitions are tracked in `scope`, because all `logic` code refers to variables by name rather than stack offset.

As mentioned earlier, the standard operation in `scope` is to define a variable and record its stack position. During a function call, however, the caller and the function body do not share the same `scope`—the two pieces of code are compiled separately.

Consider a function definition:

```text
callee copy-one-byte1 type* ( source type* target type* pos uint len uint ) { ... }

```

What it needs to care about is locating the parameter list and the return value on the current stack. This is straightforward as long as it follows the caller's push order described above. At compile time, these slots need to be registered in `scope`, but without emitting `push` instructions to reserve space at runtime. Thus, while the runtime stack top remains unchanged, the offsets tracked in `scope` advance downward. In this way, that region of stack memory is shared between the caller and the callee. Here, we directly invoke `scope`'s `define` method rather than the `define` statement implementation (which performs `scope.define` + `push`).

There is one minor detail: when a function begins compiling, it does not yet know its own instruction length. Therefore, the initial `jmp` instruction must first reserve a placeholder and backpatch the target address later, though this is just a minor implementation detail.

Code for `callee`: [service/compiler/logic/callee.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/callee.vb)

On the caller side, there are more details to consider:

```text
caller result ( param1 param2 param3 )

```

All the information recorded in `anchors` comes into play when compiling this statement:

```text
begin - The entry point address of the function, used to generate the final jmp instruction.
return_type - The return type of the function, used to check whether the provided result variable can accept the return value of the function call.
parameters - The parameter types of the function, used to check whether the number of provided arguments and their types match.

```

As mentioned previously, `variable` implements width-based type checking. To simplify the implementation, two special placeholder variable naming patterns are used:

```text
@parameter_{x}_of_{function_name}_place_holder
@return_value_of_{function_name}_place_holder

```

The former represents parameters, and the latter represents the return value. Both have their types determined by the type information looked up from `anchors`, allowing the caller implementation to simply call `variable.move_from` or `variable.copy_from` to perform type checking.

Code for `caller`: [service/compiler/logic/caller.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/caller.vb)

Speaking of functions, we must also cover the `return` statement. It needs to do two things: transfer the return value into the stack slot allocated by the caller, and return execution to the caller. The first step only requires knowing the variable name of the return value defined for the current function so its position can be looked up in `scope`. Although not strictly necessary, for consistency `callee` also names the return value `@return_value_of_{function_name}_place_holder`. The simplest way to exit the function is to emit the `rest` instruction.

This actually violates the common "single-entry, single-exit" principle of assembly code, but since this is an experimental language and users are not expected to hand-write `primitive` instructions or `logic` code, it does not really matter.

Code for `return`: [service/compiler/logic/return.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/return.vb)

Additionally, the `stst` and `rest` instructions are included in the `primitive` processor purely to simplify the implementation of the `logic` language, rather than out of strict necessity. With a suitable memory layout—for example, splitting memory into two stacks—processor state could be saved onto the second stack using a sequence of other instructions. However, just as the `or` instruction is equivalent to a combination of `add` and `not`, providing a few extra instructions does not complicate the `primitive` processor while significantly simplifying the `logic` language implementation. x86 has a similar pair of instructions in `call` and `ret`.
