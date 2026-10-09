# Building a Programming Language from 0.1 - Adding Comments

This is almost too simple to mention: in short, we just provide two `nlexer` rules and ignore them in the `syntaxer` rules.

```text
single-line-comment //[*|\n]*
multi-line-comment /\*[*|\*/]*\*/

```

Whitespace, single-line comments, and multi-line comments are all ignored:

```text
IGNORE_TYPES blank, single-line-comment, multi-line-comment

```

The same logic is implemented in both `bstyle` and `b2style`.

After that, comments can be added to the code. [service/tests/compiler/b2style/codegen/comments.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/comments.txt)

Similarly, string literals themselves do not terminate at newline characters, much like multi-line comments. So although this differs from C++, multi-line strings are naturally supported in `bstyle`. [service/tests/compiler/b2style/codegen/multiline_string.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/multiline_string.txt)
