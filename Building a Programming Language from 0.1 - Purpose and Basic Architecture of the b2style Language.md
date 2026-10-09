# Building a Programming Language from 0.1 - Purpose and Basic Architecture of the b2style Language

Compared to the `bstyle` language, the `b2style` language provides more "conventional" features—specifically `namespace` and operators.

The output of the `b2style` language is `bstyle` code, which is then compiled by the `bstyle` compiler. Therefore, unlike `bstyle`, `b2style` uses the `rewriter_rule_wrapper` mentioned earlier. The `b2style` language rewrites syntax that `bstyle` does not support while ignoring the vast majority of syntax that is identical to `bstyle`.

To support these new features, the `nlexer` rule file needs some additional rules: [service/compiler/b2style/codegen/nlexer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/codegen/nlexer_rule.txt)

```text
less-or-equal <=
greater-or-equal >=
equal ==
less-than <
greater-than >

add \+
minus -
multiply \*
divide /
mod %
power ^
bit-and &
bit-or \|
and &&
or \|\|
not !
self-inc ++
self-dec --

kw-namespace namespace

name [\w,_,::][\w,\d,_,::]*
```

These fall mainly into two categories: the first half consists of various operators, and the second half supports `namespace`.

Similarly, the `syntaxer` rules also need to change: [service/compiler/b2style/codegen/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/codegen/syntaxer_rule.txt)

```text
ROOT_TYPES function, value-definition-with-semi-colon, namespace

namespace kw-namespace name start-paragraph [function, value-definition-with-semi-colon]* end-paragraph

value [value-with-bracket, value-without-bracket]
value-with-bracket start-bracket value end-bracket
raw-value [function-call, variable-name, integer, float, bool, string]
value-without-bracket [raw-value, value-with-operation]
value-with-operation [unary-operation-value, binary-operation-value]
unary-operation-value [pre-operation-value, post-operation-value]
pre-operation-value [not, self-inc, self-dec] value
# Should support post-operation-value value [self-inc, self-dec]
post-operation-value raw-value [self-inc, self-dec]
# Should support binary-operation-value value [add, minus, multiply, divide, mod, power, bit-and, bit-or, and, or, less-than, greater-than, less-or-equal, greater-or-equal, equal] value
binary-operation-value raw-value [add, minus, multiply, divide, mod, power, bit-and, bit-or, and, or, less-than, greater-than, less-or-equal, greater-or-equal, equal] value
```

These also fall into two categories: the first part supports `namespace`, including allowing `namespace` as a root type; the second half modifies `value` to add operator expressions.

`rewriter_rule_wrapper` supports a special `rewriter`/`code_gen` called [rewriters.default](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/rewriter/default.vb), referenced in [rewriter_rule_wrapper.default_registerer](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/rewriter/rewriter_rule_wrapper.vb#L83). By default, `typed_node` types registered via `default_registerer` output the original source code directly without rewriting. [service/compiler/b2style/b2style.vb#L48](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/b2style.vb#L48)

`b2style` supports operators simply by converting them into function calls. For example:

```text
a + b
```

is converted into:

```cpp
b2style::add(a, b)  // b2style form with namespace support
b2style__add(a, b)  // bstyle form without namespace support

```

Consequently, `b2style` requires a set of built-in functions. At the same time, to make `b2style` more practical, we also add several other functions that need to be translated directly into `logic` language code—or rather, `bstyle` `logic` statements. [service/compiler/b2style/rewriter/codegen/prefix.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/codegen/prefix.txt)

Here are some excerpts:

```cpp
string b2style__true_str = "True";
string b2style__false_str = "False";

void b2style__std_out(string i) {
  logic "interrupt stdout i @@prefixes@temps@string";
}

void b2style__std_err(string i) {
  logic "interrupt stderr i @@prefixes@temps@string";
}

void b2style__bool_std_out(bool i) {
  if (i) {
    b2style__std_out(b2style__true_str);
  } else {
    b2style__std_out(b2style__false_str);
  }
}

void b2style__bool_std_err(bool i) {
  if (i) {
    b2style__std_err(b2style__true_str);
  } else {
    b2style__std_err(b2style__false_str);
  }
}

void b2style__std_out(bool i) {
  b2style__bool_std_out(i);
}

void b2style__std_err(bool i) {
  b2style__bool_std_err(i);
}

int b2style__current_ms() {
  int result;
  logic "interrupt current_ms @@prefixes@temps@string result";
  return result;
}

```

Ultimately, there will be quite a large number of these functions, since almost every function that needs to use `logic` statements directly should be included here.

`b2style` does not have any special `suffix`.

What remains are the few `rewriter`/`code_gen` implementations that do not use `rewriters.default`, which we will cover in detail later.
