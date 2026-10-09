# Building a Compiler from 0.1 (`build-a-compiler`)

English translations of the article series **"从0.1开始实现一个编程语言"** (*Building a Programming Language from 0.1*) by Zijie He, originally published on Zhihu ([Column c_1191347534474588160](https://www.zhihu.com/column/c_1191347534474588160)).

This series documents the incremental design, architectural decisions, grammar engineering, and code-generation backends for the compilers and interpreters in the `osi` project (spanning the `primitive`, `logic`, `bstyle`, `b2style`, and `b3style` languages).

---

## Translated Articles (Chronological Order)

1. [Building a Programming Language from 0.1 - Things Not Learned from Books](<Building a Programming Language from 0.1 - Things Not Learned from Books.md>) (2019-12-19)
   - *Introduces the motivation behind the series—stemming from web development frustrations and an early experimental language called `elpmis`—to document a practical workflow for building a programming language.*

2. [Building a Programming Language from 0.1 - The Absolute Basics](<Building a Programming Language from 0.1 - The Absolute Basics.md>) (2019-12-19)
   - *Explains why the series starts from "0.1" rather than raw binary and provides links to the VB.NET reference implementations (`automata`, `compiler`, and `primitive`).*

3. [Building a Programming Language from 0.1 - Interpreted vs. Compiled](<Building a Programming Language from 0.1 - Interpreted vs. Compiled.md>) (2019-12-19)
   - *Compares interpreted and compiled execution models, as well as native machine code versus virtual-machine bytecode, motivating the choice to compile down to a virtual intermediate representation called `primitive`.*

4. [Building a Programming Language from 0.1 - A Processor Called primitive](<Building a Programming Language from 0.1 - A Processor Called primitive.md>) (2019-12-19)
   - *Specifies the architecture of the virtual `primitive` processor, including its three memory regions (instructions, variable-width data stack, and state stack), five status registers (`IP`, `CO`, `DBZ`, `IN`, `DNAIP`), arbitrary-precision instruction set, and standard I/O interrupts.*

5. [Building a Programming Language from 0.1 - Input of the primitive Processor](<Building a Programming Language from 0.1 - Input of the primitive Processor.md>) (2019-12-21)
   - *Demonstrates how `primitive` instruction sequences are represented in human-readable text format (including type prefixes for constants and `rel`/`abs` stack addressing) alongside a `hello world` example using interrupts.*

6. [Building a Programming Language from 0.1 - Implementing the primitive Processor](<Building a Programming Language from 0.1 - Implementing the primitive Processor.md>) (2019-12-28)
   - *Presents the three core interfaces that make up the `primitive` processor implementation in `simulator.vb`: `exportable` (text/binary serialization), `executor` (execution and state inspection), and `imitation` (the virtual machine layer manipulated by instructions).*

7. [Building a Programming Language from 0.1 - Implementing primitive Instructions](<Building a Programming Language from 0.1 - Implementing primitive Instructions.md>) (2019-12-28)
   - *Explains how individual `primitive` instructions implement `exportable` and `instruction`, using `commands-parser.cs` to auto-generate opcode enums and serialization boilerplate from `commands.txt` while keeping execution logic in `instructions.execute.vb`.*

8. [Building a Programming Language from 0.1 - An Intermediate Language Called logic](<Building a Programming Language from 0.1 - An Intermediate Language Called logic.md>) (2019-12-28)
   - *Introduces `logic`, a whitespace-delimited intermediate language with no complex lexer or ambiguous syntax that adds named variables, data-length-based types, functions, and control flow (`if`, `while_then`, `do_while`, `do_until`) on top of `primitive`.*

9. [Building a Programming Language from 0.1 - Generating Code from logic Instruction Definitions](<Building a Programming Language from 0.1 - Generating Code from logic Instruction Definitions.md>) (2020-01-14)
   - *Describes how `importer_gen.cs` and `builder_gen.cs` generate the `logic` statement parser and strongly-typed builder APIs directly from the `logic` command specification.*

10. [Building a Programming Language from 0.1 - Anchors, Scope, and Types in the logic Language](<Building a Programming Language from 0.1 - Anchors, Scope, and Types in the logic Language.md>) (2020-01-15)
    - *Details how the `logic` compiler resolves functions via single-pass `anchors` tables and manages lexical variable lifetimes and pass-by-value stack slots via hierarchical `scope` and `scope_wrapper` structures.*

11. [Building a Programming Language from 0.1 - Variables in the logic Language](<Building a Programming Language from 0.1 - Variables in the logic Language.md>) (2020-01-15)
    - *Explains how `variable` in `logic` wraps `scope` lookups to dynamically resolve stack offsets and perform width-based type and assignment checks.*

12. [Building a Programming Language from 0.1 - Functions and Calls in logic](<Building a Programming Language from 0.1 - Functions and Calls in logic.md>) (2020-01-23)
    - *Details the calling convention, stack layout, `anchors` metadata, and compilation of `callee`, `caller`, and `return` statements in the `logic` language.*

13. [Building a Programming Language from 0.1 - Types and Macros in logic](<Building a Programming Language from 0.1 - Types and Macros in logic.md>) (2020-01-23)
    - *Describes `logic`'s width-based `types` registry (including `type0` and `type*`) and compile-time `macros` used to infer types and support function overloading without duplicating type tracking in high-level languages.*

14. [Building a Programming Language from 0.1 - An Example in the logic Language](<Building a Programming Language from 0.1 - An Example in the logic Language.md>) (2020-01-23)
    - *Presents a complete compiled `logic` program generated from `b2style` that reads the current timestamp 100 times, decrements even values, and prints `False` 100 times via `stdout`.*

15. [Building a Programming Language from 0.1 - Lexical Analysis and Syntax Analysis](<Building a Programming Language from 0.1 - Lexical Analysis and Syntax Analysis.md>) (2020-01-23)
    - *Introduces the role of lexical analysis (`lexer` producing `typed_word` tokens) and syntax analysis (`syntaxer` building a `typed_node` AST) as language-independent automata stages in a high-level compiler.*

16. [Building a Programming Language from 0.1 - Defining a Programmable Lexer nlexer](<Building a Programming Language from 0.1 - Defining a Programmable Lexer nlexer.md>) (2020-01-23)
    - *Defines the design and rule syntax of `nlexer` (natural lexical analysis), which tokenizes source code using groups of accepted and rejected character matchers with quantifiers.*

17. [Building a Programming Language from 0.1 - An Example of nlexer](<Building a Programming Language from 0.1 - An Example of nlexer.md>) (2020-01-24)
    - *Walks through the concrete `nlexer` rule file for the C-like `bstyle` language, covering keywords, literals, punctuation, and identifiers.*

18. [Building a Programming Language from 0.1 - Longest Match and Priority Match in nlexer](<Building a Programming Language from 0.1 - Longest Match and Priority Match in nlexer.md>) (2020-01-24)
    - *Explains how `nlexer` resolves ambiguity between competing token rules using longest (greedy) matching first and declaration-order priority matching for ties.*

19. [Building a Programming Language from 0.1 - Implementation Details of nlexer](<Building a Programming Language from 0.1 - Implementation Details of nlexer.md>) (2020-01-24)
    - *Outlines the ~1,100-line implementation of `nlexer` built around a unified `matcher` interface shared across `rule`, `group`, and `matcher` (including `or_matcher`).*

20. [Building a Programming Language from 0.1 - Defining a Programmable Parser syntaxer](<Building a Programming Language from 0.1 - Defining a Programmable Parser syntaxer.md>) (2020-01-24)
    - *Introduces the `syntaxer` programmable parser and its rule format for `bstyle`, showing how it mirrors `nlexer` notation to match sequences of token/rule type IDs into AST subtrees.*

21. [Building a Programming Language from 0.1 - int i; and int i=1;](<Building a Programming Language from 0.1 - int i; and int i=1;.md>) (2020-01-25)
    - *Explains how `syntaxer` uses longest/greedy matching and priority matching—just like `nlexer`—to unambiguously distinguish variable declarations (`value-declaration`) from variable definitions (`value-definition`) regardless of rule order.*

22. [Building a Programming Language from 0.1 - i++++ and set<set<T>>, i>>1](<Building a Programming Language from 0.1 - i++++ and set<set<T>>, i>>1.md>) (2020-01-25)
    - *Discusses how `syntaxer` detects non-advancing cycles in recursive grammar graphs by tracking `(rule_type, position)` pairs, how `b2style` avoids left-recursive cycles in unary/binary operations using `raw-value`, and how the `>>` vs. template `> >` ambiguity can be resolved by moving `>>` from lexical rules into syntax rules.*

23. [Building a Programming Language from 0.1 - Implementation Details of syntaxer](<Building a Programming Language from 0.1 - Implementation Details of syntaxer.md>) (2020-01-28)
    - *Describes how `syntax_collection` maps between numeric and string token types while storing rule definitions, enabling `matching_delegate` to resolve forward/cyclic rule references in `syntaxer`'s ~1,500-line implementation.*

24. [Building a Programming Language from 0.1 - Executing typed_node](<Building a Programming Language from 0.1 - Executing typed_node.md>) (2020-01-28)
    - *Introduces the layered ("Russian-doll") compiler architecture where `typed_node` trees generated by `syntaxer` are translated into the intermediate `logic` language, which is then compiled into `primitive` instructions for execution.*

25. [Building a Programming Language from 0.1 - Looking Up code_gen from typed_node](<Building a Programming Language from 0.1 - Looking Up code_gen from typed_node.md>) (2020-01-29)
    - *Details the `code_gen(Of WRITER)` interface, the `code_gens` registry mapped by node `type_name`, the `statements` mechanism (`prefixes` and `suffixes` such as `caller main ( )`), and `code_gen_rule_wrapper`, along with type-agnostic vs. type-aware node forwarding.*

26. [Building a Programming Language from 0.1 - Using code_gen](<Building a Programming Language from 0.1 - Using code_gen.md>) (2020-01-29)
    - *Contrasts the two main specializations of `code_gen_rule_wrapper`: `logic_rule_wrapper` (used by `bstyle` to emit `logic` code) and `rewriter_rule_wrapper` (used by `b2style` to selectively rewrite superset syntax nodes while leaving `bstyle`-compatible nodes untouched).*

27. [Building a Programming Language from 0.1 - Basic Architecture of the bstyle Implementation](<Building a Programming Language from 0.1 - Basic Architecture of the bstyle Implementation.md>) (2020-01-30)
    - *Presents the overall structure of `bstyle` as a minimal C-like language with function overloading, listing its full `nlexer` and `syntaxer` rule files, built-in type/constant/temp `prefixes`, and the `caller main ( )` `suffix`.*

28. [Building a Programming Language from 0.1 - Simple code_gen in the bstyle Language](<Building a Programming Language from 0.1 - Simple code_gen in the bstyle Language.md>) (2020-01-31)
    - *Walks through the straightforward `code_gen` implementations in `bstyle`—including `logic`, `multi-sentence-paragraph`, `paragraph`, `sentence`, `value-definition-with-semi-colon`, `value-with-bracket`, `value-with-comma`, and `value-without-bracket`—that primarily strip delimiters or forward child nodes.*

29. [Building a Programming Language from 0.1 - read_scoped and Handling value Nodes](<Building a Programming Language from 0.1 - read_scoped and Handling value Nodes.md>) (2020-01-31)
    - *Explains how `bstyle` evaluates nested `value` nodes by allocating position-indexed temporary variables via `with_temp_target` and pushing their names onto a `read_scoped` auto-popping stack for ancestor nodes (such as `condition`) to consume.*

30. [Building a Programming Language from 0.1 - Function Calls and Parameter Lists](<Building a Programming Language from 0.1 - Function Calls and Parameter Lists.md>) (2020-02-01)
    - *Shows how `value-list` and `function-call` use `read_scoped` to collect parameter temporary variable names for nested function calls and emit either `callee *` (via `without_return` for ignored return values) or `callee return_value` statements in `logic`.*

31. [Building a Programming Language from 0.1 - Function Definitions and return Statements](<Building a Programming Language from 0.1 - Function Definitions and return Statements.md>) (2020-02-03)
    - *Introduces `write_scoped` (using `IDisposable` for safe stack popping) to track the currently compiling function definition in `bstyle`, allowing `return` statements to omit the explicit function name required in `logic`.*

32. [Building a Programming Language from 0.1 - Function Overloading](<Building a Programming Language from 0.1 - Function Overloading.md>) (2020-02-03)
    - *Explains how `bstyle` implements function overloading via `function_name`, which mangles function names by appending `logic` language `size_of_type` macros for each parameter type.*

33. [Building a Programming Language from 0.1 - Conditionals and Loops](<Building a Programming Language from 0.1 - Conditionals and Loops.md>) (2020-02-05)
    - *Summarizes the `code_gen` implementations for `condition` (`if`/`else`), `for-loop` (using `child_named_map`), and `while` in `bstyle` by lowering them to `logic` language `if` and `while_then` builders.*

34. [Building a Programming Language from 0.1 - Purpose and Basic Architecture of the b2style Language](<Building a Programming Language from 0.1 - Purpose and Basic Architecture of the b2style Language.md>) (2020-02-05)
    - *Outlines the architecture of `b2style`, which uses `rewriter_rule_wrapper` and `rewriters.default` to rewrite `namespace` declarations and operator expressions into `bstyle` code and built-in `prefix` function calls.*

35. [Building a Programming Language from 0.1 - Supporting namespace](<Building a Programming Language from 0.1 - Supporting namespace.md>) (2020-02-05)
    - *Details how `b2style` supports nested namespaces using `write_scoped` in `kw_namespace` and rewrites `::`-qualified identifiers into `__`-separated `bstyle` names via `bstyle_format`.*

36. [Building a Programming Language from 0.1 - Supporting Operators](<Building a Programming Language from 0.1 - Supporting Operators.md>) (2020-02-06)
    - *Describes how `b2style` lowers unary, binary, and compound assignment operators into overloaded `b2style__*` function calls defined in `prefix.txt`, truncating results to fit fixed-width integer types when necessary.*

37. [Building a Programming Language from 0.1 - primitive, b2stylec, and b2style](<Building a Programming Language from 0.1 - primitive, b2stylec, and b2style.md>) (2020-02-07)
    - *Contrasts the `b2style` direct interpreter with separating compilation (`b2stylec`) and execution (`primitive`), demonstrating an order-of-magnitude speedup by precompiling `b2style` source into `primitive` instruction sequences.*

38. [Building a Programming Language from 0.1 - Adding Floating-Point Arithmetic to the b2style Language](<Building a Programming Language from 0.1 - Adding Floating-Point Arithmetic to the b2style Language.md>) (2020-03-13)
    - *Walks through the seven changes needed to support arbitrary-precision `ufloat` arithmetic (backed by `big_udec`) across `primitive`, `logic`, and `b2style`, including distinct variable-length type markers for overloading and `load_method`/`execute_loaded_method` interrupts for string conversion.*

39. [Building a Programming Language from 0.1 - Adding Integer Bitwise Shift Operations to the b2style Language](<Building a Programming Language from 0.1 - Adding Integer Bitwise Shift Operations to the b2style Language.md>) (2020-04-30)
    - *Demonstrates adding `<<`, `>>`, `<<=`, and `>>=` bitwise shift operations end-to-end across `primitive` (`lfs`/`rfs`), `logic` (`left_shift`/`right_shift`), and `b2style` rules and `prefix` functions in a single commit.*

40. [Building a Programming Language from 0.1 - Adding Comments](<Building a Programming Language from 0.1 - Adding Comments.md>) (2021-11-03)
    - *Shows how single-line (`//`) and multi-line (`/* ... */`) comments are supported in `bstyle` and `b2style` by defining token rules in `nlexer` and listing them under `IGNORE_TYPES` in `syntaxer`.*

41. [Building a Programming Language from 0.1 - #include \<stdio.h\>](<Building a Programming Language from 0.1 - #include <stdio.h>.md>) (2021-11-03)
    - *Explains how `#include`, `#ifndef`, and `#define` are implemented directly within the single-pass `code_gen` pipeline of `bstyle` and `b2style` by splitting `code_builder` from `builder` and embedding recursion depth into temporary variable names to prevent collisions.*

42. [Building a Programming Language from 0.1 - typedef and scope](<Building a Programming Language from 0.1 - typedef and scope.md>) (2021-11-07)
    - *Introduces a hierarchical lexical `scope` mechanism in `bstyle` and `b2style` to support scoped `typedef` resolution (`type_alias`), cycle detection, and canonical-type function overloading, replacing `logic` macros.*

43. [Building a Programming Language from 0.1 - struct in bstyle](<Building a Programming Language from 0.1 - struct in bstyle.md>) (2021-11-09)
    - *Details how `bstyle` and `b2style` implement `struct` purely at compile time by recursively flattening struct definitions and nested fields into dot-separated primitive variables in `logic` and packing/unpacking structs for function return values.*

44. [Building a Programming Language from 0.1 - Function Parameter References in logic](<Building a Programming Language from 0.1 - Function Parameter References in logic.md>) (2021-11-22)
    - *Implements pass-by-reference parameters (`&`) in the `logic` language by moving arguments to the callee's stack frame on call entry and moving modified stack slots back to the caller's variables before popping the stack.*

45. [Building a Programming Language from 0.1 - Function Parameter References in bstyle and b2style](<Building a Programming Language from 0.1 - Function Parameter References in bstyle and b2style.md>) (2021-11-25)
    - *Extends `bstyle` and `b2style` grammar and `value.target` handling to pass variables directly without intermediate temporaries, enabling true reference parameters and proper prefix/postfix increment and decrement operators (`++i` and `i++`).*

46. [Building a Programming Language from 0.1 - Implementing Classes with Structs](<Building a Programming Language from 0.1 - Implementing Classes with Structs.md>) (2021-11-25)
    - *Demonstrates how `b2style` implements `class` declarations by rewriting each class into a `bstyle` `struct` for member variables and overloaded free functions taking a `Class& this` reference as their first parameter.*

47. [Building a Programming Language from 0.1 - Heap Memory and Arrays](<Building a Programming Language from 0.1 - Heap Memory and Arrays.md>) (2021-11-25)
    - *Adds dynamic heap allocation (`alloc`/`dealloc`) and pointer dereferencing (`habs`/`href`) to `primitive` and `logic`, and extends `bstyle` and `b2style` to support dynamically sized arrays of primitives, structs, and classes on the heap.*

48. [Building a Programming Language from 0.1 - Optimizing by Removing Unused Functions from Compilation Output](<Building a Programming Language from 0.1 - Optimizing by Removing Unused Functions from Compilation Output.md>) (2021-12-10)
    - *Introduces a `call_hierarchy` reachability tracker rooted at `main` in `bstyle` and `b2style` to filter out unreferenced functions from the compiled output, drastically reducing compilation time and final `primitive` code size.*

49. [Building a Programming Language from 0.1 - Simple Pre-expanded Templates](<Building a Programming Language from 0.1 - Simple Pre-expanded Templates.md>) (2022-01-03)
    - *Contrasts type erasure with type expansion (monomorphization) and implements C++-style pre-expanded class templates (including nested templates) in `b2style` using AST placeholder substitution (`template_template`) and `reparser`.*

50. [Building a Programming Language from 0.1 - reinterpret_cast](<Building a Programming Language from 0.1 - reinterpret_cast.md>) (2022-01-13)
    - *Implements an in-place `reinterpret_cast(variable, Type)` construct in `bstyle` that redefines a variable's compile-time type in `scope.variable` without generating runtime instructions, laying the groundwork for class inheritance and polymorphism.*

51. [Building a Programming Language from 0.1 - Function Pointers](<Building a Programming Language from 0.1 - Function Pointers.md>) (2022-01-27)
    - *Introduces the `jmpr` register-indirect jump instruction in the `primitive` processor and adds `callee_ref`, `address_of`, and `caller_ref` statements to the `logic` language to support defining, binding, and calling function pointers.*

52. [Building a Programming Language from 0.1 - C2\<C2\<C2\<C2\<C2\<C\<void\>\>\>\>\>\> c;](<Building a Programming Language from 0.1 - C2<C2<C2<C2<C2<C<void>>>>>> c;.md>) (2022-01-27)
    - *Moves `>>` handling from the lexer to the syntaxer so that deeply nested template types can be parsed without requiring spaces between closing angle brackets.*

53. [Building a Programming Language from 0.1 - b2style_self_test](<Building a Programming Language from 0.1 - b2style_self_test.md>) (2022-01-27)
    - *Simplifies compiler feature testing by packing `b2style` libraries and self-contained test files with in-code assertions into in-memory `tar` archives executed by a generic VB.NET test runner.*

54. [Building a Programming Language from 0.1 - Delegates and Their Templates](<Building a Programming Language from 0.1 - Delegates and Their Templates.md>) (2022-01-27)
    - *Implements C#-style `delegate` declarations in `bstyle` and `b2style`—along with template delegates and call-hierarchy tracking for `--remove-unused-function`—to provide clean, reusable function pointer types.*

55. [Building a Programming Language from 0.1 - T...](<Building a Programming Language from 0.1 - T....md>) (2022-01-28)
    - *Supports overloading templates by type parameter count and evaluates variadic template parameters (`T...`), concluding that their limited applicability in `b2style` makes full variadic templates unnecessary for now.*

56. [Building a Programming Language from 0.1 - Template Functions](<Building a Programming Language from 0.1 - Template Functions.md>) (2022-02-01)
    - *Extends `b2style` templates to functions (`function-call-with-template`) and fixes namespace resolution so that templates are indexed and expanded within their defining namespaces.*

57. [Building a Programming Language from 0.1 - Class Constructors and Destructors](<Building a Programming Language from 0.1 - Class Constructors and Destructors.md>) (2022-02-02)
    - *Examines C++ initialization quirks and implements `construct` and `destruct` lifecycle functions in `b2style` via `class-initializer` syntax and scope-exit hooks, later extending them to non-class types as well.*

58. [Building a Programming Language from 0.1 - Inheriting Class Member Variables](<Building a Programming Language from 0.1 - Inheriting Class Member Variables.md>) (2022-02-11)
    - *Adds single and multiple class inheritance syntax to `b2style` and implements `class_def` and `scope.classes` to merge inherited base class member variables into derived structs.*

59. [Building a Programming Language from 0.1 - Inheriting Member Functions](<Building a Programming Language from 0.1 - Inheriting Member Functions.md>) (2022-03-05)
    - *Discusses why struct-flattening parameter passing prevents vtable-based polymorphism from accessing derived fields, demonstrates static polymorphism via templates, and implements `overridable`/`override` shadow-style method inheritance with `reinterpret_cast` forwarding.*

60. [Building a Programming Language from 0.1 - Storing Heap Pointers](<Building a Programming Language from 0.1 - Storing Heap Pointers.md>) (2022-03-08)
    - *Adds `redefine`, `undefine`, and `dealloc_heap` instructions to `logic` and unifies array/element type representations so heap pointers can outlive their lexical scope and be manually deallocated.*

61. [Building a Programming Language from 0.1 - std/vector](<Building a Programming Language from 0.1 - std-vector.md>) (2022-03-08)
    - *Introduces an initial `std::vector<T>` template class implementation in `b2style` using `get`/`set` methods and inline `logic` instructions for heap management.*

62. [Building a Programming Language from 0.1 - Storing Pointers to Heap Structs](<Building a Programming Language from 0.1 - Storing Pointers to Heap Structs.md>) (2022-03-21)
    - *Adds the `undefine` and `dealloc` compiler instructions to support heap lifetime management for structs as well as primitives, encapsulating them in `b2style::heap_ptr<T>` and `b2style::ref<T>` and refactoring `std::vector<T>` to use `heap_ptr<T>`.*

63. [Building a Programming Language from 0.1 - More Accurate Function Call Detection](<Building a Programming Language from 0.1 - More Accurate Function Call Detection.md>) (2022-03-21)
    - *Replaces the recursive DFS reachability check in `scope.call_hierarchy` with an iterative BFS—and subsequently a reverse BFS starting from `main` via a compiler suffix—to prevent stack overflows on cyclic call graphs caused by overloaded function names.*

64. [Building a Programming Language from 0.1 - Defining Types After Templates](<Building a Programming Language from 0.1 - Defining Types After Templates.md>) (2022-03-25)
    - *Introduces `root_type_injector` in `b2style` so that template expansions are injected immediately before the current `root-type` rather than at the template definition site, allowing types defined after a template (such as after `#include <std/vector>`) to be used as template arguments.*

65. [Building a Programming Language from 0.1 - Common Macros in C and C++](<Building a Programming Language from 0.1 - Common Macros in C and C++.md>) (2022-04-28)
    - *Implements predefined compile-time location tokens (`__FILE__`, `__func__`, `__LINE__`, and `__STATEMENT__`) in `bstyle` and `b2style` to provide source context for assertions and debugging without runtime overhead.*

66. [Building a Programming Language from 0.1 - If Linked Lists Are Needed](<Building a Programming Language from 0.1 - If Linked Lists Are Needed.md>) (2022-06-29)
    - *Examines the requirements for self-referential data structures like linked lists using untyped `type_ptr` pointers and `static_cast`, highlighting `b2style`'s limitations with member function templates and motivating a new `b3style` compiler that directly targets `logic`.*

67. [Building a Programming Language from 0.1 - static_cast](<Building a Programming Language from 0.1 - static_cast.md>) (2022-06-30)
    - *Implements the `static_cast` instruction in `bstyle` and `b2style` to convert between untyped `type_ptr` handles and typed struct heap pointers by generating runtime offset-calculation instructions across a struct's primitive fields.*

68. [Building a Programming Language - Basic Architecture of b3style](<Building a Programming Language - Basic Architecture of b3style.md>) (2022-07-18)
    - *Outlines the architecture and test-driven development roadmap for `b3style`, a unified compiler that accepts `b2style`/`bstyle` syntax and emits `logic` code directly without an intermediate `bstyle` rewriting pass.*

69. [Building a Programming Language from 0.1 - NLP in b3style](<Building a Programming Language from 0.1 - NLP in b3style.md>) (2022-08-14)
    - *Sets up the lexer and parser (`nlp`) for `b3style` by including `b2style`'s `nlexer` and `syntaxer` rule files and verifying syntax parsing against the existing `bstyle` and `b2style` test suites.*

70. [Building a Programming Language from 0.1 - bstyle/b2style -> scope](<Building a Programming Language from 0.1 - Moving bstyle and b2style into scope.md>) (2022-09-03)
    - *Explains how the `scope` implementations of `bstyle` and `b2style` are consolidated into a shared base `scope` class using an `accessor_t` stub for state access and a root `features_t` table for language-specific feature flags.*

71. [Building a Programming Language from 0.1 - scope/value_target](<Building a Programming Language from 0.1 - scope-value_target.md>) (2022-10-30)
    - *Describes moving `value.target`—the value stack used to evaluate nested expressions and function calls—from `bstyle` into the shared `scope` so `b3style` can also compile expressions directly into `logic` instructions.*

72. [Building a Programming Language from 0.1 - b3style/includes](<Building a Programming Language from 0.1 - b3style-includes.md>) (2022-10-30)
    - *Details how `b3style.includes` (and subsequently `b2style.includes`) is configured to search and include header files from `bstyle` and `b2style` temporary include directories.*

73. [Building a Programming Language from 0.1 - b3style/scope](<Building a Programming Language from 0.1 - b3style-scope.md>) (2023-06-26)
    - *Presents the implementation of `b3style.scope` as the union of `bstyle` and `b2style` scope components wired together through `accessor_t`.*

74. [Building a Programming Language from 0.1 - Supporting bstyle in b3style](<Building a Programming Language from 0.1 - Supporting bstyle in b3style.md>) (2023-07-12)
    - *Shows how `b3style` achieves `bstyle` compatibility and passes `bstyle_test` by reusing `bstyle` node code generators via a small `sed` codegen tool and wiring up the `b3style` compiler structure.*

75. [Building a Programming Language from 0.1 - b3style_b2style_b3style_self_test](<Building a Programming Language from 0.1 - b3style_b2style_b3style_self_test.md>) (2023-07-13)
    - *Introduces `b2style.parse_wrapper_b3style` and `b3style_b2style_b3style_self_test` to use `b3style` as the backend compiler for `b2style`'s generated `bstyle` code, expanding test coverage and adding remaining node handlers like `reinterpret_cast`, `dealloc`, and `undefine`.*

76. [Building a Programming Language from 0.1 - A Better #ifndef](<Building a Programming Language from 0.1 - A Better #ifndef.md>) (2023-08-23)
    - *Extends conditional compilation directives with `#ifdef`, `#else`, logical OR (`||`), and compiler-specific predefined macros (`BSTYLE`, `B2STYLE`, `B3STYLE`) so standard library headers can be shared across compilers.*

77. [Building a Programming Language from 0.1 - b3style::namespace](<Building a Programming Language from 0.1 - b3style::namespace.md>) (2023-09-12)
    - *Implements `namespace` support in `b3style`, directly emitting `::`-qualified names into `logic` code without needing `b2style`'s `__` prefix mangling.*

78. [Building a Programming Language from 0.1 - Implementing Operators in b3style](<Building a Programming Language from 0.1 - Implementing Operators in b3style.md>) (2023-09-24)
    - *Design and compilation of unary, binary, and self-assignment operators down to `logic` statements via `function_call` expansions; namespace handling differences in `logic` statements.*

79. [Building a Programming Language from 0.1 - Classes in b3style](<Building a Programming Language from 0.1 - Classes in b3style.md>) (2024-09-02)
    - *Lowering classes into structs and functions; instance method dispatch; constructor and destructor invocation mechanisms.*

80. [Building a Programming Language from 0.1 - Class Destructors in b3style](<Building a Programming Language from 0.1 - Class Destructors in b3style.md>) (2024-09-09)
    - *Handling `value-declaration` vs. `value-definition` vs. `class-initializer`; registering and triggering scope-exit destructor calls via `IDisposable` and `when_end_scope`.*

81. [Building a Programming Language from 0.1 - Linking and Recursion in the logic Language](<Building a Programming Language from 0.1 - Linking and Recursion in the logic Language.md>) (2025-07-30)
    - *Why single-pass compilation cannot resolve arbitrary forward-declared mutual calls without linking, but can cleanly support direct recursion via pre-defining entry points in `anchors`.*

82. [Building a Programming Language from 0.1 - Syntax Analysis of Templates in b3style](<Building a Programming Language from 0.1 - Syntax Analysis of Templates in b3style.md>) (2025-08-04)
    - *Decoupling shared template parsing logic from backend-specific writers (`typed_node_writer` vs. `logic_writer`) using partial classes and automated translation scripts.*

83. [Building a Programming Language from 0.1 - b3style_b2style_compile_error_test](<Building a Programming Language from 0.1 - b3style_b2style_compile_error_test.md>) (2025-08-07)
    - *Ensuring destructor generation failures on invalid types or typecasts cleanly raise compiler errors rather than crashing during scope disposal.*

84. [Building a Programming Language from 0.1 - b3style_b2style_self_test](<Building a Programming Language from 0.1 - b3style_b2style_self_test.md>) (2025-08-19)
    - *Running self-contained tar-archive compilation suites; bridging template function calls with and without return values (`caller` vs. `caller_ref`).*

85. [Building a Programming Language from 0.1 - linked-list.txt](<Building a Programming Language from 0.1 - linked-list.txt.md>) (2025-08-21)
    - *Fixing dynamic `type0` calculation when namespaces are disabled so functions correctly identify whether return values are expected.*

86. [Building a Programming Language from 0.1 - Using #ifdef Inside Functions](<Building a Programming Language from 0.1 - Using #ifdef Inside Functions.md>) (2025-08-21)
    - *Extending the syntaxer grammar so `#ifdef` and `#ifndef` constructs can enclose individual statements and blocks within function bodies.*

87. [Building a Programming Language from 0.1 - heap_struct_name in b3style](<Building a Programming Language from 0.1 - heap_struct_name in b3style.md>) (2025-10-15)
    - *Supporting heap-allocated structs (`a[i].b` -> `a.b[i]`) for variable read/write operations and member method dispatch.*

---

## Repository Status

- **Total Articles in Zhihu Column**: 87
- **Translated and Published**: 87
- **Pending Extraction**: 0
