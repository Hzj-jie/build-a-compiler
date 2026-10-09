# Building a Programming Language from 0.1 - Implementation Details of syntaxer

Note that syntaxer and nlexer interact through `typed_word`, and syntaxer needs to understand the `uint` type values output by nlexer. At the same time, because syntaxer's rules form a graph, a rule has often not yet been defined when it is referenced while reading the rules.

This is where the previously mentioned `syntax_collection` comes into play. It serves two functions: first, it maintains the mapping between the numeric and string representations of `type`; second, it records all the rules defined in the rule file. This allows the syntaxer implementation to use `matching_delegate` to look up the corresponding rules. [service/automata/syntax/matching_delegate.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/syntax/matching_delegate.vb)

The rest of the implementation is quite close to nlexer, including implementations of `matching_group`, `multi_matching_group`, `optional_matching_group`, and `any_matching_group`. Most of the implementation could be merged with nlexer, though the entire implementation is already very simple at only around 1,500 lines of code. [service/automata/syntax](https://github.com/Hzj-jie/osi.net/tree/master/service/automata/syntax)
