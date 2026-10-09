# Building a Programming Language from 0.1 - An Example of nlexer

A rule in `nlexer` consists of a sequence of groups matched from left to right.

Each group consists of an accepted set and a rejected set, and may be suffixed with an optional, repeating, or optional-and-repeating quantifier.

An accepted or rejected set consists of one or more matchers.

Everything else is just concrete syntax and implementation details, such as which symbols denote which constructs. In the current `nlexer` implementation, groups are enclosed in `[]`; the accepted set and rejected set are separated by `|`; the optional suffix is `?`, the repeating suffix is `+`, and the optional-or-repeating suffix is `*`; individual matchers within a set are separated by `,`. Any of these special characters can be escaped with `\`.

Explaining this in the abstract is a bit tedious—looking at a concrete example makes it much clearer. I designed a C-like language called `bstyle` (unrelated to the famous B language; I just borrowed the name). It is mostly identical to a subset of C, except for a `logic` keyword used to embed inline `logic` statements (such as interrupt calls). The `nlexer` rule file for `bstyle` looks like this:

[service/compiler/bstyle/codegen/nlexer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/codegen/nlexer_rule.txt)

```text
kw-if if
kw-else else
kw-for for
kw-while while
kw-do do
kw-loop loop
kw-return return
kw-break break
kw-logic logic

blank [\b]+

bool [true,false]
integer [+,-]?[\d]+
float [+,-]?[\d]*.[\d]+
string "[\",*|"]*"

comma \,
colon :
question-mark ?
start-paragraph {
end-paragraph }
start-bracket (
end-bracket )
start-square-bracket \[
end-square-bracket \]
semi-colon ;

assignment =

name [\w,_][\w,\d,_]*

```

Most of these are straightforward rules for keywords and punctuation symbols, while `integer`, `float`, and `string` were already covered in the previous article.

`name` is a particularly interesting case: this rule can also match any of the keywords defined above it, which raises the question of how rule ambiguity is resolved. We will discuss that in the next article.
