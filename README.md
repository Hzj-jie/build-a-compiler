# Building a Compiler from 0.1 (`build-a-compiler`)

English translations of the article series **"从0.1开始实现一个编程语言"** (*Building a Programming Language from 0.1*) by Zijie He, originally published on Zhihu ([Column c_1191347534474588160](https://www.zhihu.com/column/c_1191347534474588160)).

This series documents the incremental design, architectural decisions, grammar engineering, and code-generation backends for the compilers and interpreters in the `osi` project (spanning the `logic`, `bstyle`, `b2style`, and `b3style` languages).

---

## Translated Articles (Chronological Order)

1. [Building a Programming Language from 0.1 - Implementing Operators in b3style](<Building a Programming Language from 0.1 - Implementing Operators in b3style.md>) (2023-09-24)
   - *Design and compilation of unary, binary, and self-assignment operators down to `logic` statements via `function_call` expansions; namespace handling differences in `logic` statements.*

2. [Building a Programming Language from 0.1 - Classes in b3style](<Building a Programming Language from 0.1 - Classes in b3style.md>) (2024-09-02)
   - *Lowering classes into structs and functions; instance method dispatch; constructor and destructor invocation mechanisms.*

3. [Building a Programming Language from 0.1 - Class Destructors in b3style](<Building a Programming Language from 0.1 - Class Destructors in b3style.md>) (2024-09-09)
   - *Handling `value-declaration` vs. `value-definition` vs. `class-initializer`; registering and triggering scope-exit destructor calls via `IDisposable` and `when_end_scope`.*

4. [Building a Programming Language from 0.1 - Linking and Recursion in the logic Language](<Building a Programming Language from 0.1 - Linking and Recursion in the logic Language.md>) (2025-07-30)
   - *Why single-pass compilation cannot resolve arbitrary forward-declared mutual calls without linking, but can cleanly support direct recursion via pre-defining entry points in `anchors`.*

5. [Building a Programming Language from 0.1 - Syntax Analysis of Templates in b3style](<Building a Programming Language from 0.1 - Syntax Analysis of Templates in b3style.md>) (2025-08-04)
   - *Decoupling shared template parsing logic from backend-specific writers (`typed_node_writer` vs. `logic_writer`) using partial classes and automated translation scripts.*

6. [Building a Programming Language from 0.1 - b3style_b2style_compile_error_test](<Building a Programming Language from 0.1 - b3style_b2style_compile_error_test.md>) (2025-08-07)
   - *Ensuring destructor generation failures on invalid types or typecasts cleanly raise compiler errors rather than crashing during scope disposal.*

7. [Building a Programming Language from 0.1 - b3style_b2style_self_test](<Building a Programming Language from 0.1 - b3style_b2style_self_test.md>) (2025-08-19)
   - *Running self-contained tar-archive compilation suites; bridging template function calls with and without return values (`caller` vs. `caller_ref`).*

8. [Building a Programming Language from 0.1 - linked-list.txt](<Building a Programming Language from 0.1 - linked-list.txt.md>) (2025-08-21)
   - *Fixing dynamic `type0` calculation when namespaces are disabled so functions correctly identify whether return values are expected.*

9. [Building a Programming Language from 0.1 - Using #ifdef Inside Functions](<Building a Programming Language from 0.1 - Using #ifdef Inside Functions.md>) (2025-08-21)
   - *Extending the syntaxer grammar so `#ifdef` and `#ifndef` constructs can enclose individual statements and blocks within function bodies.*

10. [Building a Programming Language from 0.1 - heap_struct_name in b3style](<Building a Programming Language from 0.1 - heap_struct_name in b3style.md>) (2025-10-15)
    - *Supporting heap-allocated structs (`a[i].b` -> `a.b[i]`) for variable read/write operations and member method dispatch.*

---

## Repository Status

- **Total Articles in Zhihu Column**: 87
- **Translated and Published**: 10
- **Pending Extraction**: 77
