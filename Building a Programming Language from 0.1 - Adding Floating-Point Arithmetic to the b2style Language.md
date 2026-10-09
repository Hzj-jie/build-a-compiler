# Building a Programming Language from 0.1 - Adding Floating-Point Arithmetic to the b2style Language

To simplify calculating the value of π ([zhuanlan/c_1216677226609082368](https://zhuanlan.zhihu.com/c_1216677226609082368)), adding floating-point support to the `b2style` language is a very worthwhile endeavor.

Lexically and syntactically, `b2style` already supports a 16-byte `float` type, but that is not useful for arbitrary-precision computation.

Supporting arbitrary-precision floating-point numbers requires at least the following components:

1. Lexing and parsing: The language must be able to store a floating-point literal as a constant at compile time.

2. Arithmetic operations: At minimum, basic floating-point arithmetic must be supported; conversion between floating-point numbers and integers is less critical.

3. Output: Simply keeping values in memory is of limited use, so we also need a way to convert floating-point numbers into strings.

These features were broken down into the following changes:

1. Modify the `nlexer` and `syntaxer` rule files to support `ufloat`.

[commit/68b038bc3e4f4949fc347dc8bf44ccca72d19ea9](https://github.com/Hzj-jie/osi.net/commit/68b038bc3e4f4949fc347dc8bf44ccca72d19ea9)

This change is straightforward: aside from updating the rule files and their corresponding `codegen`, all that remains is using `big_udec` to parse a floating-point string and convert it into a byte sequence.

2. Add floating-point arithmetic instructions to the `primitive` processor.

[commit/d0a56cab67b31fdb140540a5d5b1efab9b6ea3b6](https://github.com/Hzj-jie/osi.net/commit/d0a56cab67b31fdb140540a5d5b1efab9b6ea3b6)

This is a set of instructions prefixed with `f`, such as addition, subtraction, multiplication, division, roots/exponents, and comparisons.

The implementation of these instructions simply delegates to `big_udec`.

3. Support multiple variable-length types in the `logic` language.

Previously, variable-length types used `max_uint32` to mark their length. However, that collides with function overloading, which mangles function names using type lengths.

Since fixed types with a length exceeding `max_int32` have no practical meaning, we define any type length greater than `max_int32` as variable-length. This gives us `string:max_uint32-3`, `ufloat:max_uint32-2`, and `biguint:max_uint32-1`, ensuring that function overloads for these types no longer collide with one another.

[commit/593b32b3f8453457d38c17231f92ebe1e260b93b](https://github.com/Hzj-jie/osi.net/commit/593b32b3f8453457d38c17231f92ebe1e260b93b)

4. Allow upward-compatible constants in the `logic` language.

Previously, upward-compatible (widening) assignments were allowed between variables, though such assignments are quite unsafe because they are raw memory copies. For example:

```text
int i = 1;
ufloat j = i;

```

does not produce the expected result and instead causes a runtime error, because the byte sequence for integer `1` cannot be recognized by `big_udec`.

Even so, this compatibility can produce meaningful results in cases like:

```text
int i = 1;
long j = i;

```

which behaves as expected.

Therefore, we can extend this upward-compatibility behavior to constants as well, making it possible to write directly:

```text
long i = 1;

```

Of course, the proper way to solve this problem is to introduce an `operator=` assignment operator, though that is not urgent and can be left for later.

[commit/901ca4280f3baa0fba0df254ee3b83bec8ffe899](https://github.com/Hzj-jie/osi.net/commit/901ca4280f3baa0fba0df254ee3b83bec8ffe899)

5. With these foundations in place, we can finally add floating-point functionality to the `logic` language.

Because the `logic` language itself does not attach semantic meaning to variable types, this simply introduces a set of new statements that map one-to-one to the new instructions in the `primitive` processor.

[commit/3cd48c63ac439bde4bc9f61c0cfb879baf850979](https://github.com/Hzj-jie/osi.net/commit/3cd48c63ac439bde4bc9f61c0cfb879baf850979)

Here we only need to modify the statement definitions in `definition.txt`; the vast majority of the remaining changes are generated code.

6. Add `load_method` and `execute_loaded_method` interrupts to the `primitive` processor.

This is a pragmatic compromise. For `biguint`, there is a `biguint_to_str` function implemented in `b2style` ([service/compiler/b2style/rewriter/codegen/prefix.txt#L354](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/codegen/prefix.txt#L354)), even though `big_uint` itself already has a `str` method. The same issue arises with `ufloat`, as `big_udec` also has a `str` method.

We could implement `ufloat_to_str` at several different layers, such as:

- Using a dedicated interrupt on the `primitive` processor.
- Using a statement in the `logic` language, such as `float_to_str`.
- Reimplementing it from scratch in `b2style`.

None of these options is particularly appealing. The first two bake `big_udec` implementation details into the processor and intermediate language, which should be agnostic to them. Note that `stdin`, `stdout`, and similar facilities are interrupts on the `primitive` processor because they are "external" features—and `to_str` is likewise an "external" feature. The floating-point instructions of the `primitive` processor only define arithmetic operations rather than representation details, so whether `to_str` is needed is orthogonal to the `primitive` processor's instruction set definition.

Reimplementing it in `b2style` is also impractical: unlike `biguint`, formatting a `ufloat` supports numerous parameters and involves much more complex logic.

Therefore, we add two general-purpose interrupts to the `primitive` processor: `load_method` and `execute_loaded_method`. The first loads a method from the host environment with the signature:

```text
public class C {
    public static byte[] operation(byte[] input) { ... }
}

```

and stores a reference to it; the second invokes that stored reference.

Beyond allowing `ufloat` to call the existing `to_str` method, this mechanism also allows executing host-specific functions without modifying any compiler logic.

[commit/cb10711c1dea44ba15d7c4d6f0f5db6f256937f9](https://github.com/Hzj-jie/osi.net/commit/cb10711c1dea44ba15d7c4d6f0f5db6f256937f9)

For example, `current_ms` can now be invoked in an alternative way:

```text
long loaded_current_ms() {
  string method_name = "osi.service.interpreter.primitive.loaded_methods, osi.service.interpreter:current_ms";
  logic "interrupt load_method method_name @@prefixes@temps@string";
  long result;
  logic "interrupt execute_loaded_method @@prefixes@temps@string result";
  return result;
}

```

7. Add `ufloat` support to the `b2style` language.

With all the changes above in place, the `b2style` language can now support `ufloat`. All that remains is implementing the basic `ufloat` functions for `b2style` in `bstyle`—such as calling the `logic` language's `float_equal` statement in:

```text
bool b2style__equal(ufloat i, ufloat j) { ... }

```

as well as using the `load_method` and `execute_loaded_method` interrupts in:

```text
string b2style__ufloat_to_str(ufloat i) { ... }

```

[commit/cab46cc0497edd62bc6d67c4c23aab0837e402c1](https://github.com/Hzj-jie/osi.net/commit/cab46cc0497edd62bc6d67c4c23aab0837e402c1)
