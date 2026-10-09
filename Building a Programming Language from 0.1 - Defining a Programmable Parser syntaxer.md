# Building a Programming Language from 0.1 - Defining a Programmable Parser syntaxer

Once you understand the `nlexer` lexical analyzer, the `syntaxer` syntax analyzer becomes much easier to grasp—at the very least, both share the same notation and symbol conventions. Let's start with an example:

```text
function name name start-bracket [paramlist]? end-bracket multi-sentence-paragraph
paramlist [param-with-comma]* param
param-with-comma param comma
param name name

value-definition-with-semi-colon value-definition semi-colon

paragraph [sentence, multi-sentence-paragraph]
sentence [sentence-with-semi-colon, sentence-without-semi-colon]
sentence-with-semi-colon [value-definition, value-declaration, value-clause, return-clause, ignore-result-function-call, kw-break, logic]? semi-colon
sentence-without-semi-colon [condition, while, for-loop]

multi-sentence-paragraph start-paragraph [sentence]* end-paragraph

```

This is an excerpt from the `syntaxer` rule file for the `bstyle` language: [service/compiler/bstyle/codegen/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/codegen/syntaxer_rule.txt).

A `syntaxer` rule file consists of:

Rules: Each rule matches a subtree and is composed of a sequence of matchers. A rule matches the current sub-sequence only if all of its matchers match from left to right. Although the "substring" here is a sequence of `typed_word` tokens rather than raw characters, the `syntaxer` implementation does not care about the string content of each `typed_word`—it only inspects its numeric type ID. The mapping between numeric type IDs and their string names is managed by `syntax_collection`, a bidirectional dictionary that needs no further explanation.

Matchers: A matcher can be either a single token/rule type or a group of types, and a group can also be marked as optional (`?`), repeating (`+`), or optional-and-repeating (`*`). The example above illustrates how these matchers are used: for instance, a statement might be an assignment or a variable definition, which is represented as a group of alternative types; a function body can contain any number of statements, which is an optional-and-repeating group; and a function declaration may or may not have a parameter list, which is an optional group.

This is very similar to `nlexer` rules, except that there is no concept of a rejected (negative) set, which reduces a group to a specific kind of matcher.
