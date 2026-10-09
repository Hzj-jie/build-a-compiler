# Building a Programming Language from 0.1 - Basic Architecture of the bstyle Implementation

We have already touched upon various implementation details of the bstyle language earlier, and here we will bring them all together.

First, the original design goal was to create a subset of the C language such that, apart from inline `logic` code, code written in it can be compiled by a C compiler.

This subset is quite small—for example, it only supports function calls and has no operators, yet functions support overloading.

Here we can distinguish two different kinds of features. One kind has an "is-a" relationship with existing features. For example, an operator "is a" function call. `namespace` is another example: although C does not have `namespace`, b2style implements `namespace`, and a `namespace` "is" simply a special way of naming functions.

The other kind does not have an "is-a" relationship with existing features. For instance, function overloading requires support from the `logic` language itself (via the `logic` macro functionality mentioned earlier), whereas b2style does not directly generate `logic` code. Therefore, if bstyle did not support function overloading, supporting function overloading in b2style would be very complicated.

Thus, the bstyle language uses nlexer and syntaxer for lexical and syntax analysis, uses `logic_rule_wrapper` to generate `logic` code, and then compiles that into a sequence of `primitive` instructions using the `logic` compiler for execution.

nlexer rule file: [service/compiler/bstyle/codegen/nlexer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/codegen/nlexer_rule.txt)

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

syntaxer rule file: [service/compiler/bstyle/codegen/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/codegen/syntaxer_rule.txt)

```text
IGNORE_TYPES blank
ROOT_TYPES function, value-definition-with-semi-colon

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

value-definition name name assignment value
value-declaration name name
value-clause name assignment value

condition kw-if start-bracket value end-bracket paragraph [else-condition]?
else-condition kw-else paragraph

while kw-while start-bracket value end-bracket paragraph

for-loop kw-for start-bracket [value-declaration, value-definition]? semi-colon [value]? semi-colon [value-clause]? end-bracket paragraph

value [value-with-bracket, value-without-bracket]
value-with-bracket start-bracket value end-bracket
value-without-bracket [function-call, variable-name, integer, float, bool, string]

variable-name name

ignore-result-function-call function-call

function-call name start-bracket [value-list]? end-bracket
value-list [value-with-comma]* value
value-with-comma value comma

return-clause kw-return [value]?

# Embed a logic statement directly, like logic "int stdout a_string"
logic kw-logic string

```

bstyle has three `prefixes`:

Built-in types: [service/compiler/bstyle/logic/prefixes/types.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/prefixes/types.vb)

```text
bigint: type*
int: 4
long: 8
bool: 1
byte: 1
string: type*
float: 8
void: type0

```

Constants: [service/compiler/bstyle/logic/prefixes/constants.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/prefixes/constants.vb)

```text
int_0, int_1, size_of_int, size_of_long, size_of_bool, size_of_byte, size_of_float

```

Temporary variables: [service/compiler/bstyle/logic/prefixes/temps.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/prefixes/temps.vb)

```text
bigint, string

```

Neither constants nor temporary variables are strictly required; they are simply used to reduce unnecessary duplication.

bstyle has one `suffix`, which is responsible for generating the call to the `main` function: [service/compiler/bstyle/logic/suffixes/main.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/suffixes/main.vb)

```text
caller main ( )

```

What remains are the code generators corresponding to each syntaxer node type.
