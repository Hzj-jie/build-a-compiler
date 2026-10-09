# Building a Programming Language from 0.1 - Defining a Programmable Lexer nlexer

The purpose of lexical analysis is to relax formatting constraints on the source language—after all, nobody wants to distinguish between:

```text
int i = 100;

```

and

```text
int i=          100     ;

```

Both are valid C statements and have the exact same meaning.

The conventional approach is to use regular expressions to extract different substrings. However, since this is an experimental language project, I decided to implement a different kind of lexer, which I call `nlexer` (`n-lexer`, or natural lexical analysis). It performs lexical analysis by simulating how a human reads source code.

When isolating a substring, a rule always accepts one set of characters (the positive/matching set) and rejects another set of characters (the negative/unmatching set). Whenever a character fails to match the accepted set or matches the rejected set, the current substring ends.

For example:

```text
String: starts with "; accepted set is any character (including newline) or \", rejected set is "; ends with ".
Integer: optional + or -; accepted set is 0-9, rejected set is any other character.
Floating-point number: optional + or -; accepted set is 0-9, rejected set is any other character; the decimal point . itself; accepted set is 0-9, rejected set is any other character.

```

Each accepted/rejected group can be optional (such as the leading `+` or `-` sign of an integer or floating-point number), repeating (such as the digits after the decimal point in a floating-point number), or both optional and repeating (such as the characters between the two `""` quotes of a string, since an empty string is also a valid string).

When an accepted pattern matches, the substring cursor advances; when a rejected pattern matches, the cursor remains unchanged.

In principle, this rule system is a superset of what is needed to support high-level languages. For instance, a rule with no accepted set and only a rejected set is meaningless because it can never advance the cursor, which would cause an infinite loop. However, this is not a major issue in practice—we can simply avoid writing such nonsensical combinations rather than burdening the lexer implementation itself with complex restrictions.

With this model, we only need to design a notation to describe these rules, and the lexer can then perform lexical analysis by reading a list of rules.

Rule file:

A rule file consists of multiple rules, where each rule defines a specific token type. The type itself is named by a string, while its corresponding numeric type ID is simply its line number in the file.

Rule:

Each rule consists of a sequence of groups executed in order. If execution successfully reaches the end of the last group, the substring is considered a match for that rule.

How conflicts between multiple matching rules are resolved will be covered in the next article.

Group:

A group contains an accepted set and a rejected set, and may be suffixed with a quantifier indicating whether it is optional, repeating, or optional-and-repeating. To simplify rule file definitions, a group is allowed to be written as a plain string, which is treated as a group containing a single accepted literal that is required and non-repeating.

Matcher:

Each item inside a group's accepted or rejected set is called a matcher. For example, `"if"` is a string matcher that matches the two characters `"if"`. Several built-in special matchers are also provided to support common character classes:

Single-character matcher `\w`: matches any single English letter and advances by one character. `\W` is its negation.

Digit matcher `\d`: matches a single digit character and advances by one character. `\D` is its negation.

Whitespace matcher `\b`: matches any whitespace character (including newline, tab, etc.) and advances by one character. `\B` is its negation.

Detailed specification: [service/automata/nlexer/design.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/nlexer/design.txt)

```text
n-lexer: natural lexical analysis

It simulates the process of a human lexical analysis. Basically it's a list of rules when parsing a program language.
Considering the following paragraph:

void function_a(type1 param1, type2 param2) {
  string s = "a bc\"";
  int x = -1.9;
  return param1 + param2 + s;
}

v -> letter -> go ahead
o -> letter -> go ahead
i -> letter -> go ahead
d -> letter -> go ahead
space -> blank -> go ahead | finish last word void
...
" -> quote -> go ahead
a -> letter -> go ahead
space -> blank -> go ahead
b -> letter -> go ahead
c -> letter -> go ahead
\" -> escaped " -> go ahead
" -> quote -> finish last word "abc\""
...
- > negative -> go ahead
1 -> digit -> go ahead
. -> dot -> go ahead
9 -> digit -> go ahead
; -> semi-colon -> finish last word -1.9

How human selects void,
  - The first character is v.
  - The second character is o.
  - The third character is i.
  - The forth character is d.
  - The fifth character is blank - ignore.

How human selects "a bc\""
  - The first character is ".
  - The second character is not ".
  - The third character is not ".
  - The forth character is not ".
  - The fifth character is not ".
  - The sixth and seventh characters are \".
  - The eighth character is ".

How human selects -1.9
  - The first character is -.
  - The second character is 1.
  - The third character is ..
  - The forth character is 9.
  - The fifth character is ; - ignore.

So either rule can be represented as a set of conditions:
  - If it's X, move forward length(X).
  - If it's Y, move forward length(Y) then stop.
  - If it's not X, stop.
Repeating above steps to finish the lexical analysis.

Each rule is {match, unmatch}{match, unmatch}...{match-stop, unmatch-stop}.
Each {match, unmatch} pair is a group, the relationship between match and unmatch is AND.
It can be suffixed by ? - 0 or 1, * - 0 or more, + 1 or more.

Sample rules:
void -> void
"a bc\"" -> "[*,\"|"]*" - * is a macro representing any character.
-1.9 -> [+,-]?[\d]*.[\d]+ - \d is a macro representing all digits.
function_a [\w,_]+ - \w is a macro representing any English character.

Reserved characters [],|?*+\, these characters can be escaped by using c-style.

Processing steps:
1. Preprocess
  1.1 INCLUDE, include files
  1.2 DEFINE, define macros
  1.3 Expand macros
2. Parse
  - A group - starting from [ and ending with ], may contain a | as the separator of match and unmatch rules.
    - Several special match rules:
	  - \d: Digits
	  - \D: Non-digits
	  - \w: English characters
	  - \W: Non-English characters
	  - \b: Spaces
	  - \B: Non-spaces
  - A raw string - a language sugar of [string]

```
