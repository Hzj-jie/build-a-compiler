# Building a Programming Language from 0.1 - i++++ and set<set<T>>, i>>1

Although the design of syntaxer is very close to that of nlexer as mentioned earlier, the rule structures of syntax analyzers and lexical analyzers are different. Rules in a lexical analyzer are linear, whereas rules in syntax analysis form a graph.

Take a simple example:

```text
value [value-with-bracket, value-without-bracket]
value-with-bracket start-bracket value end-bracket
value-without-bracket [function-call, variable-name, integer, float, bool, string]

function-call name start-bracket [value-list]? end-bracket
value-list [value-with-comma]* value
value-with-comma value comma

```

`value` represents an rvalue, which may be a function call. However, the foundation of a function call is its parameter list, where each parameter may also be an rvalue. Therefore, syntax analysis rules contain cycles.

However, if the position in the substring does not advance, it results in an infinite loop. Fortunately, this is a two-dimensional infinite loop that can be detected by tracking the rule type and the position in the substring. Simply put, a rule is not allowed to execute twice at the exact same position; the second attempt should immediately return an error.

[service/automata/syntax/matching.vb#L57](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/syntax/matching.vb#L57)

Consequently, syntaxer cannot directly support cases like `i++++`. A simplified rule file ignoring irrelevant rules looks like this:

```text
value [function-call, variable-name, value-with-operation, integer, float, bool, string]
value-with-operation [unary-operation-value, binary-operation-value]
unary-operation-value [pre-operation-value, post-operation-value]
post-operation-value value [self-inc, self-dec]

```

This creates the following infinite loop that does not require advancing the position in the substring:

```text
value -> value-with-operation -> unary-operation-value -> post-operation-value -> value

```

To handle this, `self-inc` and `self-dec` can be defined as a specific value type:

```text
value-with-post-operation value [self-inc, self-dec]+
value-with-operation [pre-operation-value, binary-operation-value]

```

The same issue also occurs in the rule:

```text
binary-operation-value value operations value

```

As an experimental language, I chose a simpler approach: disallowing the first group of `post-operation-value` from being `value`, and instead using the following rules to simplify the implementation.

```text
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

This is the syntaxer rule file for b2style, a language slightly more advanced than bstyle: [service/compiler/b2style/codegen/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/codegen/syntaxer_rule.txt)

Depending on how syntaxer rules are defined, `set<set<T>>` and `i>>1` also present some interesting situations. According to nlexer's longest/greedy matching principle, `>>` as a right-shift operator will always be matched in preference to `>`. However, in `set<set<T>>`, that is not the intention—the trailing `>>` is actually a combination of two `>` tokens. This was an issue that existed since C++98 and was not resolved until C++11.

For syntaxer, although neither bstyle nor b2style currently supports templates or bitwise shift operations, it is still an interesting problem. Of course, there is a way to handle it: simply remove `>>` from the nlexer rules and embed it into the syntaxer rules instead.

```text
# nlexer
greater-than >

```

and

```text
# syntaxer
right-shift value greater-than greater-than value
template kw-template less-than [template, name] greater-than

```
