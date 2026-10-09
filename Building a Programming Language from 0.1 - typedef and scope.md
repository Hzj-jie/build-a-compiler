# Building a Programming Language from 0.1 - typedef and scope

Early `bstyle` and `b2style` compilers were essentially stateless—meaning the compilation of each statement was largely independent of other statements. Relying on such an implementation makes it impossible to support `struct`; the specific reasons will be discussed later, so let's first talk about how to implement `typedef` in a better way.

Previously, [code_gen_rule_wrapper had a `PARAMTERS` type parameter](https://github.com/Hzj-jie/osi.net/blob/7ae2c5ec1acbd623bb36644ee2250348c20f6e08/service/compiler/code_gen/code_gen_rule_wrapper.vb) used to define variables for a compilation unit to preserve state. In `logic_gen`, this [`paramters_t` contained a `type_alias`](https://github.com/Hzj-jie/osi.net/blob/7ae2c5ec1acbd623bb36644ee2250348c20f6e08/service/compiler/logic/logic_gen/logic_rule_wrapper.vb#L56) to define type aliases. This was the "state" in "essentially stateless."

This design had an obvious flaw: these `type_alias` definitions had no concept of `scope`. For example:

```cpp
void A() {
  typedef int Int32;
}

void main() {
  Int32 x = 0;
}

```

This should fail to compile: `Int32` is defined inside function `A()`, so `main()` should not be able to reference it. However, the previous implementation could not understand function scopes; each statement was simply translated directly into `logic` code. Therefore, the same [logic/scope](https://github.com/Hzj-jie/osi.net/blob/7ae2c5ec1acbd623bb36644ee2250348c20f6e08/service/compiler/logic/scope.vb) mechanism can also be used in `bstyle` and `b2style`.

First, let's recall the role of `logic/scope`. Simply put, it records the variables, types, etc. defined by the code compiled so far, and creates child scopes via `start_scope` and `end_scope`. For instance, a line of code can access variables already defined within the current function, as well as variables already defined in the namespace containing the function, or global variables already defined. However, once the function ends, subsequent code can no longer access the variables defined inside that function.

The same functionality can also be used in `bstyle` and `b2style`. In short, a compiler at any level—`logic`, `bstyle`, or `b2style`—can maintain a `scope` implementation during compilation. `scope` itself has no built-in domain features; the compiler provides `scope` with whatever features it needs. Consequently, `scope` itself is very simple, with only two key operations: `start_scope` and `end_scope`. [service/compiler/scope.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope.vb)

```vb.net
Public Class scope(Of T As scope(Of T))
    ' ...
    Public Function start_scope() As T
        assert(child Is Nothing)
        child = inject_constructor(Of T).invoke(Me)
        Return child
    End Function

    Public Function end_scope() As T
        when_end_scope()
        assert(object_compare(in_thread, Me) = 0)
        If Not parent Is Nothing Then
            assert(object_compare(parent.child, Me) = 0)
            parent.child = Nothing
        End If
        in_thread = parent
        Return parent
    End Function
    ' ...
End Class

```

[`scope` in `logic`](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/scope.vb) implements features such as types and variable names, which is largely consistent with the previous implementation.

[The `scope` implementation in `bstyle`](https://github.com/Hzj-jie/osi.net/tree/master/service/compiler/bstyle) is more structured, including [current_function](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.current_function.vb), [define](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.define.vb), [function](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.function.vb), [type_alias](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.type_alias.vb), and [variable](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.variable.vb), as well as [struct](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/bstyle/scope.struct.vb), which will be discussed later.

`start_scope` and `end_scope` are called at block boundaries, such as entering and leaving a function or namespace.

The features of `bstyle`'s `scope` are divided into two categories. One category is actually independent of lexical scope itself, such as `define` and `function`. These features are defined only in the root scope and referenced from any scope. Other features have a separate definition in each scope, such as `type_alias`, which is how `typedef` is implemented.

`type_alias` itself is just an `unordered_map<string, string>`, leveraging `scope` to control the visibility range of a `typedef`. Because `typedef`s can be chained or even form cycles, when a `typedef` is defined, it needs to be resolved to a canonical non-alias type while also checking for circular definitions.

```vb.net
Public Function define(ByVal [alias] As String, ByVal canonical As String) As Boolean
    assert(Not [alias].null_or_whitespace())
    assert(Not canonical.null_or_whitespace())
    If [alias].Equals(canonical) Then
        raise_error(error_type.user,
                    "Alias ",
                    [alias],
                    " cannot be defined to itself.")
        Return False
    End If
    If [alias].Equals(Me(canonical)) Then
        raise_error(error_type.user,
                    "Cycle typedefs detected, alias ",
                    [alias],
                    " equals to its canonical ",
                    canonical)
        Return False
    End If
    canonical = Me(canonical)
    If m.emplace([alias], canonical).second OrElse m([alias]).Equals(canonical) Then
        Return True
    End If
    raise_error(error_type.user,
                "Alias ",
                [alias],
                " of canonical type ",
                canonical,
                " has been defined already as ",
                m([alias]),
                ".")
    Return False
End Function

```

A common operation for `scope` is searching the current scope and all its enclosing scopes. Naturally, `type_alias` uses this as well.

```vb.net
Default Public ReadOnly Property _D(ByVal [alias] As String) As String
    Get
        Dim s As scope = Me.s
        While Not s Is Nothing
            [alias] = s.ta([alias])
            s = s.parent
        End While
        Return [alias]
    End Get
End Property

```

Such a `type_alias` implementation only concerns `bstyle`; the `logic` language is unaware of these `typedef`s.

Because both `bstyle` and `b2style` now support `scope`, the `macros` previously implemented in `logic` are no longer useful—`bstyle` itself can infer the types required by the `logic` language. The only issue is that type widths are unknown: `bstyle` previously used the `sizeof` macro to implement function overloading, for example:

```text
callee m&4 type0 ( n int ) { ... }
callee m&8 type0 ( n long ) { ... }

```

However, parameter types can be used for overloading instead. Because variable types are always resolved to canonical non-alias types, `typedef` does not affect function overloading.

```cpp
typedef Integer int;
void m(int x) { ... }
typedef int Int32;
Int32 x; m(x);

```

This works without any issues:

```text
callee m&Integer( x Integer ) { ... }
define x Integer
caller m&Integer ( x )

```

`macros` have also been removed from the `logic` language. ([Remove most of macros from bstyle · Hzj-jie/osi.net@a7cdd68](https://github.com/Hzj-jie/osi.net/commit/a7cdd6819c884e3940b5676768ff55c8697899a0) [Remove logic/macros · Hzj-jie/osi.net@5fa4c3b](https://github.com/Hzj-jie/osi.net/commit/5fa4c3bdd5d353f201695d3a14d7c1251e5eb3db))
