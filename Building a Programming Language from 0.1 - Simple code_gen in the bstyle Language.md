# Building a Programming Language from 0.1 - Simple code_gen in the bstyle Language

Several `code_gen` implementations in bstyle are quite simple, so we will list them all together here.

`logic`: Directly outputs the string of the second child node after stripping the surrounding double quotes. Since the `logic` language itself does not use quotation marks, there is no need to handle escaped quotes (`\"`). [service/compiler/bstyle/logic/nodes/logic.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/logic.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    o.append(n.child(1).word().str().Trim(character.quote))
    o.append(newline.incode())
    Return True
End Function

```

`multi-sentence-paragraph`: Except for the first `{` and the last `}`, all other children are `sentence` nodes. We simply invoke `code_gens` to select the `code_gen` for `sentence`. [service/compiler/bstyle/logic/nodes/multi_sentence_paragraph.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/multi_sentence_paragraph.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Dim i As UInt32 = 1
    While i < n.child_count() - 1
        If Not l.of(n.child(i)).build(o) Then
            Return False
        End If
        i += uint32_1
    End While
    Return True
End Function

```

`paragraph`: Even simpler—its child node is either a `sentence` or a `multi-sentence-paragraph`, so we just invoke `code_gens` to select the child's `code_gen`. [service/compiler/bstyle/logic/nodes/paragraph.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/paragraph.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Return l.of(n.child()).build(o)
End Function

```

`sentence`, `sentence-with-semi-colon`, `sentence-without-semi-colon`: Just like `paragraph`, they simply select the `code_gen` of their child node. [service/compiler/bstyle/logic/nodes/sentence.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/sentence.vb)

`value-definition-with-semi-colon`: Ignores the trailing `;` and processes only the first child node. [service/compiler/bstyle/logic/nodes/value_definition_with_semi_colon.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_definition_with_semi_colon.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Return l.of(n.child(0)).build(o)
End Function

```

`value-with-bracket`: Ignores the surrounding `(` and `)` and processes only the middle child node. [service/compiler/bstyle/logic/nodes/value_with_bracket.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_with_bracket.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Return l.of(n.child(1)).build(o)
End Function

```

`value-with-comma`: Ignores the trailing `,` and processes only the first child node. [service/compiler/bstyle/logic/nodes/value_with_comma.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_with_comma.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Return l.of(n.child(0)).build(o)
End Function

```

`value-without-bracket`: Selects the `code_gen` of its single child node. [service/compiler/bstyle/logic/nodes/value_without_bracket.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value_without_bracket.vb)

```vb.net
Public Function build(ByVal n As typed_node, ByVal o As writer) As Boolean Implements logic_gen.build
    Return l.of(n.child()).build(o)
End Function

```
