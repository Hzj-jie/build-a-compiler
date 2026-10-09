# Building a Programming Language from 0.1 - Generating Code from logic Instruction Definitions

How `logic` statements are compiled into `primitive` processor instructions depends on the semantics of each individual statement. However, the mechanics of generating and parsing `logic` code itself depend solely on the structural definitions of those commands.

The parser for `logic` code converts raw statements into intermediate representation instances. For example, a type-definition statement like `type int 4` is parsed into an instance of the `define` (or `type`) structure containing the two arguments `"int"` and `4`. When the parser encounters the token `"type"`, it expects the following two tokens to be the arguments for a `type` instance—a string followed by an unsigned integer. Any mismatch is reported as a parse error.

The code generator that produces this parser from the command definitions is located at [service/compiler/logic/codegen/importer_gen.cs](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/codegen/importer_gen.cs).

The resulting generated code is at [service/compiler/logic/importer.definition.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/importer.definition.vb).

At the same time, constructing raw `logic` source strings by hand is error-prone, so we also generate strongly-typed builders from the command definitions to prevent argument type or count mismatches. For instance, instead of manually concatenating a raw string like `"type int 4"`, we can call `builders.of_define("int", 4)`, which is much less error-prone. These builders can be generated from the exact same command definitions.

The generator that produces the builder code is at [service/compiler/logic/codegen/builder_gen.cs](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/codegen/builder_gen.cs).

The generated builder code is at [service/compiler/logic/builder/builders.implementation.vb](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/logic/builder/builders.implementation.vb).

With these in place, feeding input into the `logic` language requires almost no handwritten boilerplate.
