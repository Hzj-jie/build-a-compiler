# Building a Programming Language from 0.1 - Adding Integer Bitwise Shift Operations to the b2style Language

Compared to floating-point arithmetic, bitwise shift operations are much simpler: we only need to consider the arithmetic instructions in the `primitive` processor, the lexical and syntactic rule files, and the built-in functions in `b2style`'s `prefix`. The main difference is that the integer types themselves are already handled properly.

As a result, everything can be contained in a single change, with most of the code being auto-generated. [commit/93ec9158b7271028320e8f9f132757390dfad5fe](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe)

- Modify `commands.txt` in `primitive` to add the `lfs` and `rfs` instructions. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-b24b8eea1107c7fec637fb01845ac62a](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-b24b8eea1107c7fec637fb01845ac62a)

```text
lfs	data_ref	data_ref	data_ref	#(*A big_uint) = left_shift((*B big_uint), (*C big_uint))
rfs	data_ref	data_ref	data_ref	#(*A big_uint) = left_shift((*B big_uint), (*C big_uint))

```

- The serialization and deserialization code for `primitive` instructions is auto-generated; we only need to implement the instruction execution methods. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-f2e0c5ef5e140add4a8ed2159df2341e](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-f2e0c5ef5e140add4a8ed2159df2341e)

```vb.net
        Partial Public NotInheritable Class lfs
            Implements instruction

            Public Sub execute(ByVal imi As imitation) Implements instruction.execute
                assert(Not imi Is Nothing)
                Dim b1 As big_uint = Nothing
                Dim b2 As big_uint = Nothing
                b1 = New big_uint(+p1(imi))
                b2 = New big_uint(+p2(imi))
                p0(imi).set(b1.left_shift(b2).as_bytes())
            End Sub
        End Class

        Partial Public NotInheritable Class rfs
            Implements instruction

            Public Sub execute(ByVal imi As imitation) Implements instruction.execute
                assert(Not imi Is Nothing)
                Dim b1 As big_uint = Nothing
                Dim b2 As big_uint = Nothing
                b1 = New big_uint(+p1(imi))
                b2 = New big_uint(+p2(imi))
                p0(imi).set(b1.right_shift(b2).as_bytes())
            End Sub
        End Class

```

- Modify `definition.txt` in `logic` to add the `left_shift` and `right_shift` operations. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-457e8f6876f7d1fbc3d74c6b6d4413a2](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-457e8f6876f7d1fbc3d74c6b6d4413a2)

```text
# execute left_shift command against two variables and store the result in the first variable:
#  left_shift target source1 source2
left_shift	string	string	string

# execute right_shift command against two variables and store the result in the first variable:
#  right_shift target source1 source2
right_shift	string	string	string

```

- The parsing and formatting code for `logic` operations is also auto-generated; we only need to implement the classes that lower these two operations into `primitive` instructions. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-35cd128add63f9f8827e85f783ab411c](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-35cd128add63f9f8827e85f783ab411c)

```vb.net
    Public NotInheritable Class left_shift
        Inherits binary_math_operator

        Public Sub New(ByVal types As types,
                       ByVal result As String,
                       ByVal left As String,
                       ByVal right As String)
            MyBase.New(types, result, left, right)
        End Sub

        Protected Overrides Function instruction() As command
            Return command.lfs
        End Function
    End Class

    Public NotInheritable Class right_shift
        Inherits binary_math_operator

        Public Sub New(ByVal types As types,
                       ByVal result As String,
                       ByVal left As String,
                       ByVal right As String)
            MyBase.New(types, result, left, right)
        End Sub

        Protected Overrides Function instruction() As command
            Return command.rfs
        End Function
    End Class

```

- Modify the `nlexer` and `syntaxer` rule files in `b2style` to add the four operators `<<`, `>>`, `<<=`, and `>>=`. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-1ae02234724e9f8774d7916a14e09a59](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-1ae02234724e9f8774d7916a14e09a59) and [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-2db5a520bfbdd37dc32335859d8337bf](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-2db5a520bfbdd37dc32335859d8337bf)

```text
+ left-shift <<
+ right-shift >>

self-add \+=
self-minus -=
@@ -65,6 +67,8 @@ self-bit-and &=
self-bit-or \|=
self-and &&=
self-or \|\|=
+ self-left-shift <<=
+ self-right-shift >>=

```

----------------------------------

```text
- self-value-clause name [self-add, self-minus, self-multiply, self-divide, self-mod, self-power, self-bit-and, self-bit-or, self-and, self-or] value
+ self-value-clause name [self-add, self-minus, self-multiply, self-divide, self-mod, self-power, self-bit-and, self-bit-or, self-and, self-or, self-left-shift, self-right-shift] value

condition kw-if start-bracket value end-bracket paragraph [else-condition]?
else-condition kw-else paragraph
@@ -42,7 +42,7 @@ pre-operation-value [not, self-inc, self-dec] value
# Should support post-operation-value value [self-inc, self-dec]
post-operation-value raw-value [self-inc, self-dec]
# Should support binary-operation-value value [add, minus, multiply, divide, mod, power, bit-and, bit-or, and, or, less-than, greater-than, less-or-equal, greater-or-equal, equal, not-equal] value
- binary-operation-value raw-value [add, minus, multiply, divide, mod, power, bit-and, bit-or, and, or, less-than, greater-than, less-or-equal, greater-or-equal, equal, not-equal] value
+ binary-operation-value raw-value [add, minus, multiply, divide, mod, power, bit-and, bit-or, and, or, left-shift, right-shift, less-than, greater-than, less-or-equal, greater-or-equal, equal, not-equal] value

```

- These operator tokens are not rewritten directly by `b2style` on their own, because `self-value-clause` and `binary-operation-value` look up the predefined `b2style` functions. Therefore, they simply use the default `rewriter` rule. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-2e6318d9b111e47b899f3005f3448656](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-2e6318d9b111e47b899f3005f3448656)

```vb.net
+                       default_registerer("left-shift"),
+                       default_registerer("right-shift"),
                       default_registerer("self-inc"),
                       default_registerer("self-dec"),
                       default_registerer("self-add"),
@@ -77,6 +79,8 @@ Public NotInheritable Class b2style
                       default_registerer("self-bit-or"),
                       default_registerer("self-and"),
                       default_registerer("self-or"),
+                       default_registerer("self-left-shift"),
+                       default_registerer("self-right-shift"),

```

- Implement the `b2style` `prefix` in `bstyle`, adding the `b2style::left_shift` and `b2style::right_shift` functions. [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-ae2e6fef07399fac289b733a90c8e9ed](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-ae2e6fef07399fac289b733a90c8e9ed)

```cpp
biguint b2style__left_shift(biguint i, biguint j) {
  biguint r;
  logic "left_shift r i j";
  return r;
}

biguint b2style__right_shift(biguint i, biguint j) {
  biguint r;
  logic "right_shift r i j";
  return r;
}

```

With that, the implementation is complete, and we add a test case: [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-0ee0dd5fc218094c54f8701de0d33065](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-0ee0dd5fc218094c54f8701de0d33065)

```cpp
void main() {
  int i = 100;
  i <<= 2;
  b2style::std_out(i);
  int j = 0;
  j = i >> 5;
  b2style::std_out(i);
  b2style::std_out(j);
}

```

And the expected output: [commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-219ddef7d331564d316683b724f5cfca](https://github.com/Hzj-jie/osi.net/commit/93ec9158b7271028320e8f9f132757390dfad5fe#diff-219ddef7d331564d316683b724f5cfca)

```vb.net
    <test>
    Private Shared Sub shift()
        Dim io As console_io.test_wrapper = Nothing
        io = New console_io.test_wrapper()
        Dim e As executor = Nothing
        assertion.is_true(b2style.with_functions(New interrupts(+io)).
                          parse(_b2style_test_data.shift.as_text(), e))
        assertion.is_not_null(e)
        e.execute()
        assertion.equal(io.output(), "40040012")
    End Sub

```
