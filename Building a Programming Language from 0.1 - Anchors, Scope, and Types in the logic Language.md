# Building a Programming Language from 0.1 - Anchors, Scope, and Types in the logic Language

Although `logic` differs drastically from mainstream programming languages in its lexical and syntactic structure, it implements the most fundamental building blocks of common languages: variables and functions. Variable resolution relies on `scope`, while function resolution relies on `anchors`.

The `logic` compiler emits a flat sequence of `primitive` instructions, where each instruction's position in the final sequence is fixed; likewise, the entry point of each function is at a fixed index. A function call simply sets up the arguments on the stack and executes a `jump` instruction to the function's entry point. All we need is a dictionary that maps each function name to its instruction index so that calls can jump to the right location. This dictionary is called `anchors`.

`anchors` records each function's signature—including its name, instruction index, parameter types, and return type—so that function calls can emit the corresponding setup instructions. For example:

```text
push  # prepare for the return-value
push
cp ref0 {parameter1}
push
cp ref0 {parameter2}
push
cp ref0 {parameter3}
jmp {entry of callee}

```

At the same time, this metadata allows the `logic` compiler to validate argument counts, argument types, and return types during instruction generation.

This raises a classic question: to link or not to link? In other words, can we always know the positions of all functions at compile time? If function A calling function B requires function B to be defined before function A, then we don't need a second linking pass—we can emit `primitive` instructions in a single pass. Otherwise, because jump target addresses depend on how and in what order instruction sequences are merged, an extra linking pass is required to patch jump targets.

Of course, for a simple experimental language, a separate linking phase isn't critical, so we simply omit it.

The implementation of `anchors` can be found in [service/compiler/logic/anchors.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/anchors.vb).

`scope`, on the other hand, tracks variable names, types, and stack positions.

Just like hardware processors, `primitive` references variables via relative or absolute stack positions. When a specific instruction is emitted, the memory slot it needs to access is also fixed.

Unlike `anchors`, `scope`—true to its name—is a specialized stack of lexical scopes. When looking up a variable, the search always starts from the current `scope` and walks up through parent scopes until it reaches the root `scope`. Consider the following code:

```text
define output type*

type uint 4
type bool 1

define 0 uint
copy_const 0 i0
define 1 uint
copy_const 1 i1
define 2 uint
copy_const 2 i2
define 3 uint
copy_const 3 i3

callee method type0 ( n type* ) {
  append_slice output n  // line 1
}
callee main type0 ( ) {
  define 100 uint
  copy_const 100 i100
  define i type*
  copy_const i i1
  define continue bool
  less_or_equal continue i 100
  while_then continue {
    caller method ( i )  // line 2
    add i i 1  // line 3
    less_or_equal continue i 100  // line 4
  }
}

caller main ( )

```

On line 1, `n` is defined inside `method`, while `output` is defined in the global root scope.

On line 2, `i` is defined inside `main`.

On line 3, `1` is defined in the global root scope.

On line 4, `continue` is defined inside `main`.

The implementation of `scope` itself is straightforward: its primary operations are looking up a variable's stack position and type by name, and defining a new variable (which only requires knowing its type and the current variable count on the stack). Its implementation is in [service/compiler/logic/scope.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope.vb).

A child `scope` must be created when defining a function parameter list, invoking a function, or—most commonly—entering a new block (`paragraph`).

```text
type int 4
callee method int ( p1 int p2 int ) {  // line 1
  define v int  // line 2
  return v
}

define 0 int
copy_const 0 i0
define 1 int
copy_const 1 i1
caller method ( 0 1 )  // line 3

```

Line 1: When compiling `callee method`, its parameter list does not belong to the root scope, so a new child `scope` is created. While it could share the same scope as the function body's `paragraph`, using a dedicated scope keeps the implementation simpler without any real downside.

Line 2: Compiling a `paragraph` always creates a new child `scope`.

Line 3: When compiling a function call (`caller`), the argument list does not belong to the enclosing scope, so it also gets a new child `scope`. While we could theoretically pass existing variables directly without allocating new ones, creating a new scope serves two important purposes:

1. It enforces pass-by-value rather than pass-by-reference semantics.
2. It reuses the exact same mechanism to verify that argument types match the function signature.

This works because arguments are always copied from their original references into newly allocated stack slots.

In summary, `scope` is used in two ways:

1. For `define` operations: each time a variable is defined in a `scope`, a `push` instruction is emitted to allocate a slot on the memory stack for subsequent instructions to use. When the `scope` exits, an equal number of `pop` instructions are emitted to restore the stack top.
2. Inside a function definition (`callee`), to map parameters onto the stack slots already allocated by the caller. In this case, no `push` instructions are emitted, and parameter names in the callee do not need to match the argument names at the call site because they reside in completely separate scopes—the callee simply resolves each parameter to its corresponding stack offset.

Since the first usage pattern is shared between function calls and block (`paragraph`) compilation, that logic is encapsulated in `scope_wrapper`: [service/compiler/logic/scope_wrapper.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope_wrapper.vb).
