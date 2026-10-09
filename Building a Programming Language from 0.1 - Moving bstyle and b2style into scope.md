# Building a Programming Language from 0.1 - bstyle/b2style -> scope

As mentioned in the previous article [Zijie He: Building a Programming Language - Basic Structure of b3style](https://zhuanlan.zhihu.com/p/538312521), the `scope` of `b3style` is roughly equivalent to the `scope` of `bstyle` plus the `scope` of `b2style`. Therefore, the first step is to move the `scope` implementations of both `bstyle` and `b2style` into a shared location.

In principle, `bstyle`, `b2style`, and `b3style` will all use the exact same `scope` implementation, with a few language-specific specializations. However, some features in `scope` depend on others—most commonly `type_alias`. Although not strictly required, almost every type-related `scope` feature calls `type_alias` to resolve type aliases.

This requires the base `scope` class to provide all functionality, while the derived `scope` classes for each language implementation rely on simple accessors to access derived-class fields and check whether specific features are supported. Right now, we need to support `bstyle` and `b2style` while eliminating duplicate functionality such as `scope.variable`.

To do this, we need an accessor—simply put, a mechanism to read and write a specific variable in the current `scope`. This differs from the previous `scope.current().something()`, where `something()` itself decided whether to return the variable from the current `scope` or from one of its ancestor nodes. [service/compiler/scope/scope.accessor.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope/scope.accessor.vb)

```vb.net
    Private ReadOnly _accessor As lazier(Of accessor_t) = lazier.of(AddressOf Me.get_accessor)

    Private Function myself() As accessor_t
        Return +_accessor
    End Function

    Protected Overridable Function get_accessor() As accessor_t
        assert(False)
        Return Nothing
    End Function

    Protected Class accessor_t
    ' ...
        Public Overridable Function current_function() As current_function_t
            assert(False)
            Return Nothing
        End Function

        Public Overridable Sub current_function(ByVal c As current_function_t)
            assert(False)
        End Sub
    ' ... 
```

The accessor itself is a stub where all functions fail an assertion by default. A language-specific `scope` implementation overrides and returns only the functions it needs while ignoring the rest. As shown above, these functions include both getters and setters as needed. When `scope` calls the accessor, it uses `scope.current().myself()`.

We also need a feature table to indicate which features the current `scope` supports. Unlike the accessor, a language's `scope` implementation can have only one feature table. To avoid exposing the implementation outside `scope`, the feature table is not a template type parameter; instead, its instance exists only on the root `scope` and is retrieved via `from_root`, which also avoids needing `lazier` to prevent redundant instantiations. An added benefit is that the compiler implementation can dynamically toggle specific features based on certain parameters. [osi.net/scope.features.vb at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope/scope.features.vb)

```vb.net
    Private ReadOnly _features As features_t

    Protected Sub New(ByVal parent As T)
        MyBase.New(parent)
        _features = If(is_root(), get_features(), Nothing)
    End Sub

    Private Function features() As features_t
        Return from_root(Function(ByVal x As T) As features_t
                             assert(Not x Is Nothing)
                             Return x._features
                         End Function)
    End Function

    Protected Class features_t
        Public Overridable Function with_type_alias() As Boolean
            Return True
        End Function

        Public Overridable Function with_namespace() As Boolean
            Return True
        End Function
    End Class
```

Currently, the supported feature flags include whether type aliases and namespaces are enabled.

With this design in place, merging the `scope` implementations of `bstyle` and `b2style` into one place becomes straightforward, starting with the `scope` definition itself:

```vb.net
' An implementation for b*style implementations.
Partial Public Class scope(Of WRITER As {lazy_list_writer, New},
                              __BUILDER As func_t(Of String, WRITER, Boolean),
                              __CODE_GENS As func_t(Of code_gens(Of WRITER)),
                              T As scope(Of WRITER, __BUILDER, __CODE_GENS, T))
    Inherits scope(Of T)
```

The `scope` implementation requires the output `WRITER As {lazy_list_writer, New}` and `__BUILDER As func_t(Of String, WRITER, Boolean)` to recompile generated code—for example, the template implementation needs to compile expanded template code again. It also takes `__CODE_GENS As func_t(Of code_gens(Of WRITER))` to locate the `code_gen` responsible for handling specific AST nodes.

The design here is that `b3style` is a superset of `b2style`'s `nlexer` and `syntaxer` rules, so `scope` can safely reference node type names through `code_gens`. As a result, the previous `scope` implementations from `bstyle` and `b2style` can be reused almost entirely with only minor adjustments.

The rest involves nothing particularly special—just minor engineering details, such as updating references in the previous implementations to go through `scope.current()`.

The main changes are in:

[commit/1c7e73560dcbe0359edd3a7418110f2e4cc660ce](https://github.com/Hzj-jie/osi.net/commit/1c7e73560dcbe0359edd3a7418110f2e4cc660ce)

[Merge b2style/scope.variable with scope/scope.variable (#66) · Hzj-jie/osi.net@9478366](https://github.com/Hzj-jie/osi.net/commit/94783663be8eb4968165fddb64221fd486545c4e)

[commit/dd8137f8b80f91ac892d14d8cd8215f4286054be](https://github.com/Hzj-jie/osi.net/commit/dd8137f8b80f91ac892d14d8cd8215f4286054be)
