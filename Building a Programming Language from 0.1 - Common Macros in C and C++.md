# Building a Programming Language from 0.1 - Common Macros in C and C++

C and C++ provide a series of so-called "standard predefined macros" ([Standard Predefined Macros (The C Preprocessor)](https://gcc.gnu.org/onlinedocs/cpp/Standard-Predefined-Macros.html)), which are converted into values of their respective types at compile time. For example, `__FILE__` is converted into a string literal containing the name of the current source file.

An important use case of these macros is providing source location information for debugging code such as logging. Because these macros are expanded during preprocessing, they do not incur any runtime performance overhead—unlike using debugging symbols (such as debug symbols on Linux or PDB files on Windows).

Note that bstyle and b2style still have no concept of debug symbols, and the generated primitive instructions are essentially impossible to map back to the original source code. When writing slightly more complex programs, debugging becomes very difficult—for instance, when hitting `assert(false)`, you cannot even tell where it was triggered. Of course, you could manually pass location strings as arguments, but that is quite tedious, so let's consider implementing several of these macros.

First of all, bstyle and b2style do not have the oft-criticized C preprocessor macro system; code is transformed in a single pass during compilation. Therefore, what implements these "macros" is not actually a macro processor, but simply dedicated token types.

[service/compiler/shared/nlexer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/shared/nlexer_rule.txt)

```text
kw-file __FILE__
kw-func __func__
kw-line __LINE__
kw-statement __STATEMENT__
```

Here we implement four "macros": file name, function name, line number, and current statement. These four "macros" can be used as ordinary values.

[service/compiler/shared/syntaxer_rule.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/shared/syntaxer_rule.txt)

```text
- base-value-without-bracket [..., string]
+ base-value-without-bracket [..., string, kw-file, kw-func, kw-line, kw-statement]
```

Naturally, they also correspond to four `code_gen` implementations.

[service/compiler/bstyle/bstyle.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/bstyle.vb)

```vb.net
+     with(kw_file.instance).
+     with(kw_func.instance).
+     with(kw_line.instance).
+     with(kw_statement.instance)
```

[service/compiler/b2style/b2style.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/b2style.vb)

```vb.net
+    with(kw_statement.instance).
+    with(code_gen.of_input(Of typed_node_writer)("kw-file")).
+    with(code_gen.of_input(Of typed_node_writer)("kw-func")).
+    with(code_gen.of_input(Of typed_node_writer)("kw-line")).
```

(In the initial implementation, `kw-file` did not return the file name in b2style, so b2style did not need its own implementation of `kw-file` and simply forwarded it to bstyle.)

The underlying principle is the same across all of them: converting the token into a value of a type supported by the lower-level language. This is relatively simple in b2style, since bstyle supports these types well and only requires a straightforward transformation. For example:

```text
__FILE__  ->  "b2style/stdio.h"
__STATEMENT__  ->  "assert(__STATEMENT__, i > 0);"
```

The implementation of `__STATEMENT__` in b2style is very simple ([service/compiler/b2style/rewriter/nodes/kw_statement.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/kw_statement.vb)):

```vb.net
 Public Function build(ByVal n As typed_node,
                      ByVal o As typed_node_writer) As Boolean Implements code_gen(Of typed_node_writer).build
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    Return assert(o.append("""" + n.ancestor_of("sentence").input().c_escape() + """"))
End Function
```

In bstyle, it is slightly more involved because temporary variables must be generated using the `logic/copy_const` instruction. A simple way to do this is to reuse the logic in [service/compiler/bstyle/logic/nodes/integer.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/integer.vb) to generate the code:

```vb.net
Public Shared Function build(ByVal n As typed_node, ByVal i As Int32, ByVal o As logic_writer) As Boolean
    assert(Not n Is Nothing)
    assert(Not o Is Nothing)
    Return builders.of_copy_const(value.with_single_data_slot_temp_target(code_types.int, n, o),
                                  New data_block(i)).
                    to(o)
End Function
```

For instance, [service/compiler/bstyle/logic/nodes/kw_line.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/kw_line.vb) only needs to call this function:

```vb.net
Public Function build(ByVal n As typed_node,
                      ByVal o As logic_writer) As Boolean Implements code_gen(Of logic_writer).build
    Return _integer.build(n, CInt(n.char_start()), o)
End Function
```

Since `syntaxer` has no concept of "lines" at all, this actually uses the character offset rather than the line number.

What remains is how to obtain the values themselves. A slightly more interesting one is [service/compiler/bstyle/logic/nodes/kw_statement.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/kw_statement.vb):

```vb.net
public function build(byval n as typed_node,
                      byval o as logic_writer) as boolean implements code_gen(of logic_writer).build
    assert(not n is nothing)
    assert(not o is nothing)
    return _string.build(n, n.ancestor_of("sentence").input(), o)
end function
```

It needs to find the nearest enclosing `sentence` node and output the source text of that node.

The newly added `ancestor_of` method in [service/automata/typed_node.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/automata/typed_node.vb) is responsible for finding the nearest ancestor node of a given type:

```vb.net
 Public Function ancestor_of(ByVal name As String, ByRef o As typed_node) As Boolean
    assert(Not name.null_or_empty())
    Dim n As typed_node = Me
    While True
        assert(Not n Is Nothing)
        If n.type_name.Equals(name) Then
            o = n
            Return True
        End If
        If n.root() Then
            Return False
        End If
        n = n.parent
    End While
    Return assert(False)
End Function

Public Function ancestor_of(ByVal name As String) As typed_node
    Dim o As typed_node = Nothing
    assert(ancestor_of(name, o))
    Return o
End Function
```

Another one is [service/compiler/bstyle/logic/nodes/kw_file.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/kw_file.vb). Because `#include` must be supported, the current file name is maintained using a stack in [osi.net/code_gen_rule_wrapper.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/code_gen/code_gen_rule_wrapper.vb):

```vb.net
 Public Shared Function current_file() As String
    Dim r As String = Nothing
    If instance_stack(Of String, parse_wrapper).back(r) Then
        Return r
    End If
    Return "unknown_file"
End Function

' @VisibleForTesting
Public Shared Function with_current_file(ByVal filename As String) As IDisposable
    assert(Not filename.empty_or_whitespace())
    instance_stack(Of String, parse_wrapper).current() = filename
    Return defer.to(Sub()
                        instance_stack(Of String, parse_wrapper).current() = Nothing
                    End Sub)
End Function

Public Function parse_file(ByVal filename As String, ByRef e As executor) As Boolean
    Dim o As executor = Nothing
    If Not with_current_file(filename, Function(ByVal s As String) As Boolean
                                           Return parse(s, o)
                                       End Function) Then
        Return False
    End If
    e = o
    Return True
End Function
```

At the same time, [osi.net/includes.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/includes.vb) needs to use the new `with_current_file` function to push the current file name onto the stack:

```vb.net
Public NotInheritable Class file_parser
    Inherits __do(Of String, logic_writer, Boolean)

    Public Overrides Function at(ByRef i As String, ByRef j As logic_writer) As Boolean
        If i Is Nothing Then
            ' The file has been included already.
            Return True
        End If
        Dim o As logic_writer = j
        Return parse_wrapper.with_current_file(i, Function(ByVal s As String) As Boolean
                                                      Return code_builder.current().build(s, o)
                                                  End Function)
    End Function
End Class
```

All that remains is adding a `string` parameter in places like [service/compiler/b2style/lib/assert.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/assert.h) to make use of `__STATEMENT__`, as shown in [service/tests/compiler/b2style/codegen/assert_with_statement.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/codegen/assert_with_statement.txt):

```cpp
#include <b2style.h>
#include <assert.h>

void main() {
  int i = 100;
  assert(__STATEMENT__, i >= 100, "line 7");
  assert(__STATEMENT__, i < 100, "line 8");
}
```

The changes above can mainly be found in:

[commit/1487ab07880497ed42ca7ab62fe925024db67516](https://github.com/Hzj-jie/osi.net/commit/1487ab07880497ed42ca7ab62fe925024db67516)

[commit/a91e2951acfd69283764cbe1b92892f394a489ab](https://github.com/Hzj-jie/osi.net/commit/a91e2951acfd69283764cbe1b92892f394a489ab)

[commit/84ca64924d8df42a962e8e86b21c2aafc81d88e0](https://github.com/Hzj-jie/osi.net/commit/84ca64924d8df42a962e8e86b21c2aafc81d88e0)
