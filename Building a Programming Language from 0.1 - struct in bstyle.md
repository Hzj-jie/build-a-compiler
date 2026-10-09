# Building a Programming Language from 0.1 - struct in bstyle

Structs are very common and are generally used to represent a combination of several variables with a specific meaning. The vast majority of programming languages support structs—usually via `struct` or `table`—to extend primitive types. Here, primitive types refer to the built-in types provided by the programming language itself, such as `int`, `bool`, `byte`, `string`, and so on.

For a language to support structs, several things need to be considered:

1. Syntactically defining a struct
2. Defining a variable of a struct type
3. Assigning a value to a struct variable
4. Assigning from a value, such as a function return value or another variable of the same type
5. Assigning directly via dedicated syntax
6. Referencing any member within a struct
7. Passing a struct as a parameter into a function
8. Returning a struct from a function

Let's use a simple example in C to illustrate each requirement, since `bstyle` itself is very close to C.

```c
// Define a struct
struct S {
  int x;
  int y;
};

// Return a struct from a function
S create_S() {
  S s = { .x = 100, .y = 100 }; // Assign directly via dedicated syntax
  return s;
}

// Pass a struct as a parameter into a function
void print_S(S s) {
  printf("%d %d", s.x, s.y); // Reference members within a struct
}

void main() {
  S s; // Define a variable of a struct type
  s = create_S(); // Assign from a function return value
  print_S(s);
}

```

Next, we need to solve these problems one by one. Before doing so, however, we should first think about what a struct actually is. In fact, there is nothing magical about a struct: a struct is simply a fixed-length array of heterogeneous types. The mindset shift needed here is that this array is not created at runtime, but at compile time. The users of structs are the programmers using the programming language, not the end users of the program. Therefore, structs do not need to exist at runtime—here we say "do not need to" exist because in some higher-level languages, such as Java and C#, struct definitions can be inspected at runtime via reflection, though we will not discuss that here. Let's look at how each layer understands structs:

- At runtime, the `primitive` processor only cares about stack addresses and has no concept of structs whatsoever.
- Whether the `logic` language needs to support structs is a question worth considering. To state the conclusion upfront: the `logic` language does not need to support structs, so the `logic` language does not need to know about structs either.
- Structs thus become a feature supported entirely at the `bstyle` layer, which connects back to the previous article. Supporting `scope` in `bstyle` now has a clear purpose: struct definitions and defined struct variables need to be recorded in the compiler, and since the compiler needs to know about struct variables, `bstyle` needs the concept of `scope` so that variables going out of scope can be automatically discarded by the compiler.

Syntactically defining a struct

This is actually quite simple. First, define the `nlexer` and `syntaxer` rules:

```text
# nlexer_rule.txt
kw-struct struct 

# syntaxer_rule.txt
struct kw-struct name start-paragraph [value-declaration-with-semi-colon]* end-paragraph semi-colon 

```

Then record the defined struct during compilation. This uses `scope.struct` mentioned earlier. Because the `logic` language does not know about structs, a struct variable in `logic` is a collection of several primitive-type variables. Therefore, when defining a struct, what needs to be saved is the fully flattened list of all primitive-type variables. "Fully flattened" is to account for nested structs; a simple example makes this clear:

```text
struct S {
  int x;
  int y;
};
S s;
// The set of variables in logic is:
// define s.x int
// define s.y int
struct S2 {
  int x;
  S s;
};
S2 s2;
// The set of variables in logic is:
// define s2.x int
// define s2.s.x int
// define s2.s.y int

```

Here, the expansion of `S2` contains three `int`s.

We use the simplest possible approach to minimize work: directly joining the expanded primitive types with a dot (`.`), so that `s2.s.x` can directly access the `x` variable of `s` inside `s2`. The way nested structs are expanded is also straightforward: if a member's type is a struct, expand it recursively; otherwise, it is a primitive type, which means `bstyle` does not need to know all the primitive types of `logic`.

[service/compiler/bstyle/scope.struct.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.struct.vb)

```vb.net
Private ReadOnly s As New unordered_map(Of String, vector(Of single_data_slot_variable))()

' TODO: Support value_definition
Private Function resolve_type(ByVal type As String,
                              ByVal name As String,
                              ByVal o As vector(Of single_data_slot_variable)) As Boolean
    assert(Not o Is Nothing)
    ' Struct member types are always resolved during the define / build stage, so scope.current() equals to
    ' the scope where the struct_t instance is being defined.
    type = scope.current().type_alias()(type)
    Dim sub_type As vector(Of single_data_slot_variable) = Nothing
    If Not s.find(type, sub_type) Then
        o.emplace_back(New single_data_slot_variable(type, name))
        Return True
    End If
    assert(Not sub_type Is Nothing)
    sub_type.stream().
             foreach(Sub(ByVal t As single_data_slot_variable)
                             assert(Not t Is Nothing)
                             o.emplace_back(New single_data_slot_variable(t.type, strcat(name, ".", t.name)))
                         End Sub)
    Return True
End Function

Public Function define(ByVal type As String,
                       ByVal members As vector(Of struct_member)) As Boolean
    assert(Not type.null_or_whitespace())
    assert(Not members Is Nothing)
    Dim o As New vector(Of single_data_slot_variable)()
    Dim i As UInt32 = 0
    While i < members.size()
        assert(Not members(i) Is Nothing)
        If Not resolve_type(members(i).type, members(i).name, o) Then
            Return False
        End If
        i += uint32_1
    End While
    If s.emplace(type, o).second() Then
        Return True
    End If
    raise_error(error_type.user, "Struct type ", type, " has been defined already as: ", s(type))
    Return False
End Function

```

`bstyle`'s struct handler directly calls `scope.struct.define` to define structs at compile time.

Defining a variable of a struct type

Earlier we mentioned how we simply join variables inside a struct with a dot (`.`) to simplify member references. Consequently, defining a struct variable only requires prepending the struct variable name to each member variable of the struct.

[service/compiler/bstyle/scope.struct.vb#L62-L81](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.struct.vb#L62-L81)

[service/compiler/bstyle/logic/nodes/struct.vb#L148-L161](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/struct.vb#L148-L161)

```vb.net
' scope.struct.vb
Public Function resolve(ByVal type As String,
                        ByVal name As String,
                        ByRef o As vector(Of single_data_slot_variable)) As Boolean
    assert(Not type.null_or_whitespace())
    ' name can be null or whitespace to check the availability of a struct definition.
    type = scope.current().type_alias()(type)
    If Not s.find(type, o) Then
        ' raise_error(error_type.user, "Struct type ", type, " has not been defined.")
        ' Do not log, value_declaration and value_definition always check if a struct is defined before
        ' forwarding the defintion directly to the logic.
        Return False
    End If
    o = o.stream().
          map(Function(ByVal member As single_data_slot_variable) As single_data_slot_variable
                  Return New single_data_slot_variable(member.type, strcat(name, ".", member.name))
              End Function).
          collect(Of vector(Of single_data_slot_variable))()
    Return True
End Function

' struct.vb
Public Function define_in_stack(ByVal type As String, ByVal name As String, ByVal o As writer) As Boolean
    assert(Not o Is Nothing)
    Dim v As vector(Of single_data_slot_variable) = Nothing
    If Not resolve(type, name, v) Then
        Return False
    End If
    assert(Not v Is Nothing)
    Return v.stream().
             map(Function(ByVal m As single_data_slot_variable) As Boolean
                     assert(Not m Is Nothing)
                     Return value_declaration.declare_single_data_slot(m.type, m.name, o)
                 End Function).
             aggregate(bool_stream.aggregators.all_true)
End Function

```

Referencing any member within a struct

This requires no special handling: variables in `logic` are separated by dots (`.`), so they can be referenced directly using C syntax.

Passing a struct as a parameter into a function

There is nothing particularly special about this either; function parameters simply expand the struct in the same way. However, previously `value.target` only considered handling a single primitive type at a time, whereas now a `value` can be a collection of several primitive types. In simple terms, a "value" can also be a struct variable, and struct variables are handled by directly expanding them.

[service/compiler/bstyle/logic/nodes/value.target.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/value.target.vb)

```vb.net
Public NotInheritable Class target
    Implements IComparable(Of target)

    Public ReadOnly type As String
    Public ReadOnly names As vector(Of String)

    Public Sub New(ByVal type As String, ByVal names As vector(Of String))
        assert(Not type.null_or_whitespace())
        assert(Not names.null_or_empty())
        type = scope.current().type_alias()(type)
        If Not scope.current().structs().defined(type) Then
            assert(names.size() = 1)
        End If

        Me.type = type
        Me.names = names
    End Sub

```

Returning a struct from a function

Because the `logic` language itself only supports a single return value, returning a struct from a function is slightly more complicated. The basic idea is to use the two `primitive` processor instructions `append_slice` and `cut_slice` to pack a struct into a single `data_slot`, and then unpack it at the call site.

[service/compiler/bstyle/logic/nodes/struct.vb#L97-L131](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/struct.vb#L97-L131)

```text
Public Shared Function pack(ByVal sources As vector(Of String),
                            ByVal target As String,
                            ByVal o As writer) As Boolean
    assert(Not sources Is Nothing)
    assert(Not target.null_or_whitespace())
    If sources.empty() Then
        Return True
    End If
    Return sources.stream().
                   map(Function(ByVal source As String) As Boolean
                           assert(Not source.null_or_whitespace())
                           Return builders.of_append_slice(target, source).to(o)
                       End Function).
                   aggregate(bool_stream.aggregators.all_true)
End Function

Public Shared Function unpack(ByVal source As String,
                              ByVal targets As vector(Of String),
                              ByVal o As writer) As Boolean
    assert(Not source.null_or_whitespace())
    assert(Not targets Is Nothing)
    If targets.empty() Then
        Return True
    End If
    Dim index As String = strcat(targets(0), "@index")
    assert(value_declaration.declare_single_data_slot(code_types.int, index, o))
    Return targets.stream().
                   map(Function(ByVal target As String) As Boolean
                           assert(Not target.null_or_whitespace())
                           ' TODO: Should include bstyle_constant.h automatically.
                           Return builders.of_cut_slice(target, source, index).to(o) AndAlso
                                  builders.of_add(index, index, "@@prefixes@constants@int_1").to(o)
                       End Function).
                   aggregate(bool_stream.aggregators.all_true)
End Function

```

Calling `struct.pack`: [service/compiler/bstyle/logic/nodes/return_clause.vb#L68](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/return_clause.vb#L68)

Calling `struct.unpack`: [service/compiler/bstyle/logic/nodes/function_call.vb#L77](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/function_call.vb#L77)

Assigning from a value

As mentioned above, a `value` can be a collection of several primitive types, so assignment also needs to copy multiple primitive data elements.

[service/compiler/bstyle/logic/nodes/struct.vb#L29-L95](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/logic/nodes/struct.vb#L29-L95)

```text
Public Shared Function copy(ByVal sources As vector(Of String),
                            ByVal target As String,
                            ByVal o As writer) As Boolean
    Return copy(sources,
                target,
                Function(ByVal vs As vector(Of single_data_slot_variable)) As Boolean
                    assert(Not vs Is Nothing)
                    assert(vs.size() = sources.size())
                    Dim i As UInt32 = 0
                    While i < vs.size()
                        If Not builders.of_copy(vs(i).name, sources(i)).to(o) Then
                            Return False
                        End If
                        i += uint32_1
                    End While
                    Return True
                End Function)
End Function

```

Note that the two types are not required to be strictly identical here; whether an assignment is valid depends on the check in the `logic` language. For example, copying an `int` to a `long` is allowed, whereas the reverse is not.

Assigning directly via dedicated syntax

This has not been implemented yet, though there is nothing special about it; we will update this once it is implemented.

Implementing `struct` in `b2style`

There is only one thing to watch out for: in `b2style`, every `name` is rewritten by `namespace` to prepend a `"namespace__"` prefix. However, this conflicts with how struct member variables are accessed, so `struct` needs a rewriter override in `b2style`.

[service/compiler/b2style/rewriter/nodes/struct.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/rewriter/nodes/struct.vb)

```text
Protected Overrides Function build(ByVal child As typed_node,
                                   ByVal index As UInt32,
                                   ByVal o As typed_node_writer) As Boolean
    assert(Not child Is Nothing)
    assert(Not o Is Nothing)
    If Not child.type_name.Equals("value-declaration-with-semi-colon") Then
        Return MyBase.build(child, index, o)
    End If

    ' TODO: Support value-definition
    assert(child.child_count() = 2)
    assert(child.child(0).child_count() = 2)
    If Not l.of(child.child(0).child(0)).build(o) Then
        Return False
    End If
    ' Ignore namespace prefix for variables within the structure.
    o.append(child.child(0).child(1))
    If Not l.of(child.child(1)).build(o) Then
        Return False
    End If
    Return True
End Function

```

Finally, add a few tests:

[service/tests/compiler/bstyle/codegen/function_struct.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/codegen/function_struct.txt)

[service/tests/compiler/bstyle/codegen/nested_struct.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/codegen/nested_struct.txt)

[service/tests/compiler/bstyle/codegen/return_struct.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/bstyle/codegen/return_struct.txt)
