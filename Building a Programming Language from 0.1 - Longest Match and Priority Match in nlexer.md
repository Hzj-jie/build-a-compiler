# Building a Programming Language from 0.1 - Longest Match and Priority Match in nlexer

Consider the following rules:

```text
kw-return return
blank [\b]+
name [\w,_][\w,\d,_]*

```

When matching the following input string:

```text
return return_value

```

we expect to get three `typed_word` tokens: `kw-return`, `blank`, and `name`. But why don't we get `name`, `blank`, and `name`—or even `name`, `blank`, `kw-return`, and `name`? Here, `nlexer` relies on two simple disambiguation rules: longest (greedy) match and priority match.

Longest (greedy) match, as the name implies, means that at any input position, the lexer always selects the rule that produces the longest match, regardless of the order in which rules are defined. For example, `return_value` always matches `name` rather than splitting into `kw-return` followed by `name`, because the former matches all 12 characters at once whereas the latter matches only 6 characters (leaving the remaining 6 characters to match another rule).

Priority match means that when two rules match substrings of the exact same length, the rule defined earlier in the rule file wins. Thus, even though both `kw-return` and `name` can match `return`, `kw-return` is defined before `name` in the rule file, so `return` always matches `kw-return`.

The exact same principles will also be used later in syntax analysis, which we will cover when we get there.
