# Building a Programming Language from 0.1 - Input of the primitive Processor

Initializing the `primitive` processor requires an interrupt function table, and interrupt handlers can have different implementations depending on the context—for example, redirecting output to an in-memory string during unit tests.

Once initialized, feeding a sequence of instructions into the processor is all it takes to run them.

Instructions can be provided in two formats: a human-readable text format and a less readable binary format. Both are simply different representations of the exact same instruction sequence.

Here is an example of the text format:

```text
push
push
push
push
push
push
push
push
## rel0 text
## rel1 len
## rel2 index
## rel3 int function id
## rel4 int return
## rel5 1 for cutl length
## rel6 equal return
## rel7 cutl return

cpc rel0 Ehello\x20world\n
sizeof rel1 rel0
cpc rel2 i0
cpc rel3 i0

## This is a simple way.
int rel3 rel0 rel4

## This is a complex way.
cpc rel5 i1
equal rel6 rel1 rel2
jumpif rel5 rel6
cutl rel7 rel0 rel2 rel5
int rel3 rel7 rel4
add rel2 rel2 rel5
jump rel-5

cpc rel2 i0
cpc rel3 i1

## This is a simple way.
int rel3 rel0 rel4

## This is a complex way.
cpc rel5 i1
equal rel6 rel1 rel2
jumpif rel5 rel6
cutl rel7 rel0 rel2 rel5
int rel3 rel7 rel4
add rel2 rel2 rel5
jump rel-5
stop

```

This snippet outputs the string `hello world` to both `stdout` and `stderr` via `int 0` and `int 1`, demonstrating two different ways to do so (printing the whole string at once vs. iterating character by character).

There are a few simple conventions in the text format. For example, constants use different single-character prefixes to denote their data types:

1. Integer, `i123`, means `123` in integer. 4 bytes.

2. Long, `l123`, means `123` in long, i.e. `int64`. 8 bytes.

3. Double, `d123.2`, means `123.2` in double. 8 bytes.

4. Boolean, `b1`, means `true`. 1 byte.

5. Byte array, `aABCDEF0123456789`. Two characters represent one byte.

6. C-escaped string, `Eabcde\t\20`. One character represents 1-3 bytes. i.e. utf-8. Note, lots of characters are considered as separators in the string format of instructions, they should be escaped.

7. string, `sabc`. One character represents 1-3 bytes. i.e. utf-8. No escape support.

Similarly, `rel` and `abs` prefixes denote relative and absolute stack indexing modes, respectively.

Since the binary format is not human-readable, we won't go into full detail here. The basic idea is to encode each instruction opcode as a `uint`, followed immediately by its operands (stack references and constant data blocks).
