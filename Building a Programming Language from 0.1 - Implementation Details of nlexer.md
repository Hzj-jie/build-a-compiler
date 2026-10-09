# Building a Programming Language from 0.1 - Implementation Details of nlexer

The design of `nlexer` is quite concise. Aside from basic utilities such as reading rule files, the core lexer itself is only about 1,100 lines of code. Following the design, it is organized into:

`rule`: [service/automata/nlexer/rule.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/nlexer/rule.vb). Responsible for parsing a single rule definition and matching a substring against that rule.

`group`: [service/automata/nlexer/group](https://github.com/Hzj-jie/osi.net/tree/master/service/automata/nlexer/group). Responsible for parsing a group and matching a substring against it, including handling suffixes for optional and repeating quantifiers.

`matcher`: [service/automata/nlexer/matcher](https://github.com/Hzj-jie/osi.net/tree/master/service/automata/nlexer/matcher). Responsible for parsing a single matcher and matching a substring against it, including specialized implementations for digit matching, character matching, negated matching, never-matching, and so on. Among these is `or_matcher` ([service/automata/nlexer/matcher/or_matcher.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/nlexer/matcher/or_matcher.vb)), which selects one match from a set of candidate matchers. Because a group allows choosing from multiple matchers, `or_matcher` is used in the group implementation and implements the longest-match and priority-match rules discussed in the previous article.

With these building blocks, `nlexer` itself is very simple: an `nlexer` instance holds a list of rules and only needs to apply longest match and priority match across them: [service/automata/nlexer/nlexer.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/nlexer/nlexer.vb).

Every `rule`, `group`, and `matcher` implements the following unified interface:

```text
Public Interface matcher
    'start matching string i from pos and return the end of the match.
    Function match(ByVal i As String, ByVal pos As UInt32) As [optional](Of UInt32)
End Interface

```

This keeps composition uniform and straightforward.
