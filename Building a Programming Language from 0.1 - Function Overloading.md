# Building a Programming Language from 0.1 - Function Overloading

As mentioned earlier, functions in the `bstyle` language need to support overloading. This decision affects the implementation of both `function-call` and `function`, through a feature called `function_name`.

Simply put, `function_name` rewrites (mangles) a function's name based on its definition.

- The `bstyle` language does not understand types; type handling is done entirely in the `logic` language, so the rewritten function name uses the `macros` feature of the `logic` language.
- Another question to consider is what information to include in the function name.

Since any function can be called directly with its return value ignored (`ignore-result-function-call`), only the parameter types of the function are critical. And because the `logic` language itself provides width-based type checking, appending the type widths to the function name is a fitting approach.

```text
Private Shared Function build(ByVal raw_name As String, ByVal parameters As vector(Of String)) As String
    assert(Not parameters Is Nothing)
    Dim s As StringBuilder = Nothing
    s = New StringBuilder()
    s.Append(raw_name)
    Dim i As UInt32 = 0
    While i < parameters.size()
        s.Append("&").Append(macros.size_of_type_of(parameters(i)))
        i += uint32_1
    End While
    Return Convert.ToString(s)
End Function

```

The same logic applies to both function definitions and function calls. For example:

```text
callee b2style__std_out&{size_of_type-type*} type0 ( i type* ) {
  interrupt stdout i @@prefixes@temps@string
}

callee b2style__bool_std_out&{size_of_type-bool} type0 ( i bool ) { ... }

callee b2style__std_out&{size_of_type-bool} type0 ( i bool ) {
  define raw_value_@639-640 {type_of-i}
  copy raw_value_@639-640 i
  caller b2style__bool_std_out&{size_of_type-{type_of-raw_value_@639-640}} ( raw_value_@639-640 )
}

caller b2style__std_out&{size_of_type-{type_of-raw_value_@6993-7006}} ( raw_value_@6993-7006 )

```

During compilation, the `logic` language expands these macros and selects the appropriate function. If no function exists for a given type, the `logic` language reports an error.
