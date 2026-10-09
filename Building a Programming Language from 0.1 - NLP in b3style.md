# Building a Programming Language from 0.1 - NLP in b3style

Implementing the NLP (lexer and parser) for b3style is very straightforward: for now, we can simply reuse b2style's `nlexer_rule` and `syntaxer_rule` directly.

```text
service/compiler/b3style/codegen/nlexer_rule.txt
@@ -0,0 +1,3 @@
﻿
INCLUDE ../b2style/nlexer_rule.txt

－－－－－－－－－－

service/compiler/b3style/codegen/syntaxer_rule.txt
@@ -0,0 +1,3 @@

INCLUDE ../b2style/syntaxer_rule.txt
```

The underlying base class is `logic_rule_wrapper` rather than `rewriter_rule_wrapper`, as the former generates logic code:

```vb.net
Partial Public NotInheritable Class b3style
    Inherits logic_rule_wrapper(Of nlexer_rule_t, syntaxer_rule_t, prefixes_t, suffixes_t, logic_gens_t, scope) 
```

The concrete implementations of the remaining components are currently empty; we only need the NLP part to work, which requires just `nlexer_rule_t` and `syntaxer_rule_t`.

One slight difference is that because b3style's `nlexer_rule_t` and `syntaxer_rule_t` `INCLUDE` the rule files from b2style's implementation, we must ensure that b2style's rule files have already been extracted to the temporary directory:

```vb.net
    Public NotInheritable Class nlexer_rule_t
        Inherits __do(Of String)

        Private Shared ReadOnly file As String = Path.Combine(folder, "nlexer_rule.txt")

        Shared Sub New()
            ' Ensure that b2style's rule files are extracted to the temporary directory.
            static_constructor(Of b2style.nlexer_rule_t).execute()
            b3style_rules.nlexer_rule.sync_export(file)
        End Sub

        Protected Overrides Function at() As String
            Return file
        End Function
    End Class 
```

### Test-Driven Verification

The implementation above can easily be tested using existing bstyle and b2style code. Of course, the current implementation cannot generate executable code yet, but testing the NLP stage works without any issues.

The existing tests for bstyle and b2style each fall into two categories: runnable tests and compilation-error tests. Compilation-error tests are further divided into syntax errors and semantic compilation errors—the former can be caught by the NLP stage, whereas the latter are considered syntactically valid by the NLP stage. For example:

```cpp
void main() {
  string s.;  // Syntax error: `.` cannot be the last character of a variable name.
  int x = 1;
  int x = 2;  // Compilation error: `x` is redefined.
}
```

Therefore, we need special handling during testing.

The basic test setup is also divided into two parts: one uses the previous `b2style_self_test_runner`, now renamed to [compiler_self_test_runner](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/compiler_self_test_runner.vb); the other uses reflection to discover all test data:

```vb.net
Private Shared Sub run_tests(ByVal t As Type)
    For Each v As FieldInfo In t.GetFields(binding_flags.static_public)
        Dim s As String = direct_cast(Of Byte())(v.GetValue(Nothing)).as_text()
        If v.Name().StartsWith("errors_") AndAlso
           ignored_compile_errors.find(v.Name()) = ignored_compile_errors.end() Then
            assertion.is_false(b3style.nlp().parse(s), v.full_name())
        Else
            assertion.is_true(b3style.nlp().parse(s), v.full_name())
        End If
    Next
End Sub

<test>
Private Shared Sub bstyle_tests()
    run_tests(GetType(_bstyle_test_data))
End Sub

<test>
Private Shared Sub b2style_tests()
    run_tests(GetType(_b2style_test_data))
End Sub
```

Here, `ignored_compile_errors` is used to distinguish between syntax errors and semantic compilation errors. These methods are all included in [b3style_nlp_tests](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b3style/b3style_nlp_test.vb).

All of the changes above are mainly in [Implement b3style nlexer and syntaxer with nlp test · Hzj-jie/osi.net@5f58561](https://github.com/Hzj-jie/osi.net/commit/5f58561146ba14c3eb63785f0e52d3d3a11124f5).
