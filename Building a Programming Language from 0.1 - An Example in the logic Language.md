# Building a Programming Language from 0.1 - An Example in the logic Language

Beyond the topics covered so far, there are a few additional implementation details in [service/compiler/logic](https://github.com/Hzj-jie/osi.net/tree/master/service/compiler/logic). With `logic` in place, we no longer need to target the `primitive` instruction set directly; instead, we can compile high-level languages into the `logic` language, as shown in the following example:

```text
 type uint 4
 type int 4
 type bool 1
 type byte 1
 type float 8
 define @@prefixes@constants@int_0 int
 copy_const @@prefixes@constants@int_0 a00000000
 define @@prefixes@constants@int_1 int
 copy_const @@prefixes@constants@int_1 a01000000
 define @@prefixes@constants@size_of_uint int
 copy_const @@prefixes@constants@size_of_uint a04000000
 define @@prefixes@constants@size_of_int int
 copy_const @@prefixes@constants@size_of_int a04000000
 define @@prefixes@constants@size_of_bool int
 copy_const @@prefixes@constants@size_of_bool a01000000
 define @@prefixes@constants@size_of_byte int
 copy_const @@prefixes@constants@size_of_byte a01000000
 define @@prefixes@constants@size_of_float int
 copy_const @@prefixes@constants@size_of_float a08000000
 define @@prefixes@temps@bigint type*
 define @@prefixes@temps@biguint type*
 define @@prefixes@temps@string type*
 callee b2style__std_out type0 ( i type* ) { interrupt stdout i @@prefixes@temps@string
 }
 callee b2style__std_err type0 ( i type* ) { interrupt stderr i @@prefixes@temps@string
 }
 callee b2style__bool_std_out type0 ( i bool ) { define _true type*
 define raw_value_@245-251 type*
 copy_const raw_value_@245-251 a54727565
 move _true raw_value_@245-251
 define _false type*
 define raw_value_@272-279 type*
 copy_const raw_value_@272-279 a46616C7365
 move _false raw_value_@272-279
 define raw_value_@288-289 {type_of-i}
 copy raw_value_@288-289 i
 if raw_value_@288-289 { define raw_value_@315-320 {type_of-_true}
 copy raw_value_@315-320 _true
 caller b2style__std_out ( raw_value_@315-320 )
 } else { define raw_value_@357-363 {type_of-_false}
 copy raw_value_@357-363 _false
 caller b2style__std_out ( raw_value_@357-363 )
 }
 }
 callee b2style__bool_std_err type0 ( i bool ) { define _true type*
 define raw_value_@432-438 type*
 copy_const raw_value_@432-438 a54727565
 move _true raw_value_@432-438
 define _false type*
 define raw_value_@459-466 type*
 copy_const raw_value_@459-466 a46616C7365
 move _false raw_value_@459-466
 define raw_value_@475-476 {type_of-i}
 copy raw_value_@475-476 i
 if raw_value_@475-476 { define raw_value_@502-507 {type_of-_true}
 copy raw_value_@502-507 _true
 caller b2style__std_err ( raw_value_@502-507 )
 } else { define raw_value_@544-550 {type_of-_false}
 copy raw_value_@544-550 _false
 caller b2style__std_err ( raw_value_@544-550 )
 }
 }
 callee b2style__current_ms int ( ) { define result int
 interrupt current_ms @@prefixes@temps@string result
 define raw_value_@681-687 {type_of-result}
 copy raw_value_@681-687 result
 return b2style__current_ms raw_value_@681-687
 }
 callee b2style__str_concat type* ( i type* j type* ) { append_slice i j
 define raw_value_@783-784 {type_of-i}
 copy raw_value_@783-784 i
 return b2style__str_concat raw_value_@783-784
 }
 callee b2style__and bool ( i bool j bool ) { and i i j
 define raw_value_@860-861 {type_of-i}
 copy raw_value_@860-861 i
 return b2style__and raw_value_@860-861
 }
 callee b2style__or bool ( i bool j bool ) { or i i j
 define raw_value_@935-936 {type_of-i}
 copy raw_value_@935-936 i
 return b2style__or raw_value_@935-936
 }
 callee b2style__not bool ( i bool ) { not i i
 define raw_value_@1002-1003 {type_of-i}
 copy raw_value_@1002-1003 i
 return b2style__not raw_value_@1002-1003
 }
 callee b2style__equal bool ( i int j int ) { define result bool
 equal result i j
 define raw_value_@1102-1108 {type_of-result}
 copy raw_value_@1102-1108 result
 return b2style__equal raw_value_@1102-1108
 }
 callee b2style__greater_than bool ( i int j int ) { define result bool
 more result i j
 define raw_value_@1213-1219 {type_of-result}
 copy raw_value_@1213-1219 result
 return b2style__greater_than raw_value_@1213-1219
 }
 callee b2style__less_than bool ( i int j int ) { define result bool
 less result i j
 define raw_value_@1321-1327 {type_of-result}
 copy raw_value_@1321-1327 result
 return b2style__less_than raw_value_@1321-1327
 }
 callee b2style__less_or_equal bool ( i int j int ) { define raw_value_@1389-1448 {return_type_of-b2style__or}
 define raw_value_@1401-1425 {return_type_of-b2style__less_than}
 define raw_value_@1420-1421 {type_of-i}
 copy raw_value_@1420-1421 i
 define raw_value_@1423-1424 {type_of-j}
 copy raw_value_@1423-1424 j
 caller b2style__less_than raw_value_@1401-1425 ( raw_value_@1420-1421 raw_value_@1423-1424 )
 define raw_value_@1427-1447 {return_type_of-b2style__equal}
 define raw_value_@1442-1443 {type_of-i}
 copy raw_value_@1442-1443 i
 define raw_value_@1445-1446 {type_of-j}
 copy raw_value_@1445-1446 j
 caller b2style__equal raw_value_@1427-1447 ( raw_value_@1442-1443 raw_value_@1445-1446 )
 caller b2style__or raw_value_@1389-1448 ( raw_value_@1401-1425 raw_value_@1427-1447 )
 return b2style__less_or_equal raw_value_@1389-1448
 }
 callee b2style__greater_or_equal bool ( i int j int ) { define raw_value_@1513-1575 {return_type_of-b2style__or}
 define raw_value_@1525-1552 {return_type_of-b2style__greater_than}
 define raw_value_@1547-1548 {type_of-i}
 copy raw_value_@1547-1548 i
 define raw_value_@1550-1551 {type_of-j}
 copy raw_value_@1550-1551 j
 caller b2style__greater_than raw_value_@1525-1552 ( raw_value_@1547-1548 raw_value_@1550-1551 )
 define raw_value_@1554-1574 {return_type_of-b2style__equal}
 define raw_value_@1569-1570 {type_of-i}
 copy raw_value_@1569-1570 i
 define raw_value_@1572-1573 {type_of-j}
 copy raw_value_@1572-1573 j
 caller b2style__equal raw_value_@1554-1574 ( raw_value_@1569-1570 raw_value_@1572-1573 )
 caller b2style__or raw_value_@1513-1575 ( raw_value_@1525-1552 raw_value_@1554-1574 )
 return b2style__greater_or_equal raw_value_@1513-1575
 }
 callee b2style__fit_in_int int ( x int ) { define len int
 sizeof len x
 define raw_value_@1660-1689 {return_type_of-b2style__greater_than}
 define raw_value_@1682-1685 {type_of-len}
 copy raw_value_@1682-1685 len
 define raw_value_@1687-1688 int
 copy_const raw_value_@1687-1688 a04000000
 caller b2style__greater_than raw_value_@1660-1689 ( raw_value_@1682-1685 raw_value_@1687-1688 )
 if raw_value_@1660-1689 { subtract len len @@prefixes@constants@size_of_int
 cut_slice x x @@prefixes@constants@int_0 len
 }
 define raw_value_@1831-1832 {type_of-x}
 copy raw_value_@1831-1832 x
 return b2style__fit_in_int raw_value_@1831-1832
 }
 callee b2style__add int ( i int j int ) { add i i j
 define raw_value_@1905-1927 {return_type_of-b2style__fit_in_int}
 define raw_value_@1925-1926 {type_of-i}
 copy raw_value_@1925-1926 i
 caller b2style__fit_in_int raw_value_@1905-1927 ( raw_value_@1925-1926 )
 return b2style__add raw_value_@1905-1927
 }
 callee b2style__minus int ( i int j int ) { subtract i i j
 define raw_value_@2007-2029 {return_type_of-b2style__fit_in_int}
 define raw_value_@2027-2028 {type_of-i}
 copy raw_value_@2027-2028 i
 caller b2style__fit_in_int raw_value_@2007-2029 ( raw_value_@2027-2028 )
 return b2style__minus raw_value_@2007-2029
 }
 callee b2style__multiply int ( i int j int ) { multiply i i j
 define raw_value_@2112-2134 {return_type_of-b2style__fit_in_int}
 define raw_value_@2132-2133 {type_of-i}
 copy raw_value_@2132-2133 i
 caller b2style__fit_in_int raw_value_@2112-2134 ( raw_value_@2132-2133 )
 return b2style__multiply raw_value_@2112-2134
 }
 callee b2style__divide int ( i int j int ) { define result int
 divide result @@prefixes@temps@string i j
 define raw_value_@2257-2284 {return_type_of-b2style__fit_in_int}
 define raw_value_@2277-2283 {type_of-result}
 copy raw_value_@2277-2283 result
 caller b2style__fit_in_int raw_value_@2257-2284 ( raw_value_@2277-2283 )
 return b2style__divide raw_value_@2257-2284
 }
 callee b2style__mod int ( i int j int ) { define result int
 divide @@prefixes@temps@string result i j
 define raw_value_@2404-2431 {return_type_of-b2style__fit_in_int}
 define raw_value_@2424-2430 {type_of-result}
 copy raw_value_@2424-2430 result
 caller b2style__fit_in_int raw_value_@2404-2431 ( raw_value_@2424-2430 )
 return b2style__mod raw_value_@2404-2431
 }
 callee b2style__power int ( i int j int ) { power i i j
 define raw_value_@2508-2530 {return_type_of-b2style__fit_in_int}
 define raw_value_@2528-2529 {type_of-i}
 copy raw_value_@2528-2529 i
 caller b2style__fit_in_int raw_value_@2508-2530 ( raw_value_@2528-2529 )
 return b2style__power raw_value_@2508-2530
 }
 callee b2style__bit_and int ( i int j int ) { and i i j
 define raw_value_@2607-2608 {type_of-i}
 copy raw_value_@2607-2608 i
 return b2style__bit_and raw_value_@2607-2608
 }
 callee b2style__bit_or int ( i int j int ) { or i i j
 define raw_value_@2683-2684 {type_of-i}
 copy raw_value_@2683-2684 i
 return b2style__bit_or raw_value_@2683-2684
 }
 callee b2style__self_inc int ( x int ) { add x x @@prefixes@constants@int_1
 define raw_value_@2780-2802 {return_type_of-b2style__fit_in_int}
 define raw_value_@2800-2801 {type_of-x}
 copy raw_value_@2800-2801 x
 caller b2style__fit_in_int raw_value_@2780-2802 ( raw_value_@2800-2801 )
 return b2style__self_inc raw_value_@2780-2802
 }
 callee b2style__self_dec int ( x int ) { subtract x x @@prefixes@constants@int_1
 define raw_value_@2903-2925 {return_type_of-b2style__fit_in_int}
 define raw_value_@2923-2924 {type_of-x}
 copy raw_value_@2923-2924 x
 caller b2style__fit_in_int raw_value_@2903-2925 ( raw_value_@2923-2924 )
 return b2style__self_dec raw_value_@2903-2925
 }
 callee is_even bool ( i int ) { define raw_value_@2968-3022 {return_type_of-b2style__equal}
 define raw_value_@2986-3013 {return_type_of-b2style__mod}
 define raw_value_@3002-3004 {type_of-i}
 copy raw_value_@3002-3004 i
 define raw_value_@3007-3010 int
 copy_const raw_value_@3007-3010 a02000000
 caller b2style__mod raw_value_@2986-3013 ( raw_value_@3002-3004 raw_value_@3007-3010 )
 define raw_value_@3016-3019 int
 copy_const raw_value_@3016-3019 a00000000
 caller b2style__equal raw_value_@2968-3022 ( raw_value_@2986-3013 raw_value_@3016-3019 )
 return is_even raw_value_@2968-3022
 }
 callee main type0 ( ) { define i int
 define raw_value_@3064-3067 int
 copy_const raw_value_@3064-3067 a00000000
 move i raw_value_@3064-3067
 define raw_value_@3070-3102 {return_type_of-b2style__less_than}
 define raw_value_@3091-3093 {type_of-i}
 copy raw_value_@3091-3093 i
 define raw_value_@3095-3100 int
 copy_const raw_value_@3095-3100 a64000000
 caller b2style__less_than raw_value_@3070-3102 ( raw_value_@3091-3093 raw_value_@3095-3100 )
 while_then raw_value_@3070-3102 { define x int
 define raw_value_@3154-3180 {return_type_of-b2style__current_ms}
 caller b2style__current_ms raw_value_@3154-3180 ( )
 move x raw_value_@3154-3180
 define raw_value_@3190-3206 {return_type_of-is_even}
 define raw_value_@3201-3203 {type_of-x}
 copy raw_value_@3201-3203 x
 caller is_even raw_value_@3190-3206 ( raw_value_@3201-3203 )
 if raw_value_@3190-3206 { define raw_value_@3217-3241 {return_type_of-b2style__self_dec}
 define raw_value_@3237-3239 {type_of-x}
 copy raw_value_@3237-3239 x
 caller b2style__self_dec raw_value_@3217-3241 ( raw_value_@3237-3239 )
 move x raw_value_@3217-3241
 }
 define raw_value_@3272-3288 {return_type_of-is_even}
 define raw_value_@3283-3285 {type_of-x}
 copy raw_value_@3283-3285 x
 caller is_even raw_value_@3272-3288 ( raw_value_@3283-3285 )
 caller b2style__bool_std_out ( raw_value_@3272-3288 )
 define raw_value_@3110-3134 {return_type_of-b2style__self_inc}
 define raw_value_@3130-3132 {type_of-i}
 copy raw_value_@3130-3132 i
 caller b2style__self_inc raw_value_@3110-3134 ( raw_value_@3130-3132 )
 move i raw_value_@3110-3134
 copy raw_value_@3091-3093 i
 copy_const raw_value_@3095-3100 a64000000
 caller b2style__less_than raw_value_@3070-3102 ( raw_value_@3091-3093 raw_value_@3095-3100 )
 }
 }
 caller main ( )

```

This example is quite dry and tedious, and certainly not meant to be read directly by humans. However, this code uses the `current_ms` interrupt to read the current time 100 times; if the value is even, it decrements it by one, and then outputs as a `bool` whether the resulting value is even. Consequently, after invoking `main` at the end, this program prints `False` 100 times via the `stdout` interrupt.
