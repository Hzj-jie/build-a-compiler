# Building a Programming Language from 0.1 - Using code_gen

`code_gen_rule_wrapper` simplifies the rules of nlexer and syntaxer, as well as the mappings between `statements` and `code_gen`, into a unified pattern, so that the generator for a given language only needs to care about the concrete implementations of `code_gen`, `statements`, and `WRITER`.

There are two common ways to use it: `logic_rule_wrapper` and `rewriter_rule_wrapper`.

As the name implies, `logic_rule_wrapper` converts a `typed_node` into `logic` language code. Its corresponding `WRITER` naturally needs to support all basic types in the `logic` language, including integers, strings, `data_block`, and various combinations of these types.

[service/compiler/logic/logic_gen/logic_rule_wrapper.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/logic_gen/logic_rule_wrapper.vb)

`rewriter_rule_wrapper` is more specialized: it is used to rewrite certain nodes while ignoring others. For example, when implementing b2style—which is a superset of bstyle—we only need to translate the syntax constructs not supported by bstyle, while leaving the already-supported syntax nodes untouched.

[service/compiler/rewriter/rewriter_rule_wrapper.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/rewriter/rewriter_rule_wrapper.vb)

Therefore, bstyle uses `logic_rule_wrapper`, while b2style uses `rewriter_rule_wrapper`. Their specific usage will also be analyzed in detail later.
