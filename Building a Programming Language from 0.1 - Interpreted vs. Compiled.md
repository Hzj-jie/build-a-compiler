# Building a Programming Language from 0.1 - Interpreted vs. Compiled

A programming language must ultimately be executed. There are two approaches to execution: one is interpreted, where the compiler and executor do not produce any intermediate output; the other is compiled, where the compiler compiles the code and outputs a result, and the executor runs what the compiler outputs.

Of course, there is no strict boundary between these two execution models. For instance, if you hide the invocation of `clang` behind the scenes, C code can appear to be "interpreted"—as seen in common online C++ IDEs. Conversely, some interpreted languages support outputting compilation results into a more efficient intermediate representation, such as `PyCodeObject` / `.pyc` files.

Interpreted languages hide the compilation process inside the executor, making them better suited for simpler scenarios without strict performance requirements, since compilation happens on every run. Compiled languages achieve significantly higher execution efficiency because the compilation step does not need to be repeated, making them better suited for performance-critical scenarios. However, the compilation overhead itself makes it difficult to simply load source code as data at runtime—after all, a production server might not even have `clang` installed.

Since these distinctions are all relative and the compilation process itself cannot be bypassed, let's first assume that we have a compiler.

What gets executed from source code is the compiler's output. This output also falls into two categories. The first can be executed directly on a hardware processor—often referred to as native code, binary code, and so on. Languages that take this approach include assembly, C, C++, and Go. The second is executed on a virtual machine, where the compiled output is an intermediate representation known as bytecode, CIL, or similar.

There are many processor instruction sets—x86, amd64, arm, arm64, MIPS, and the trendy OpenRISC—each with numerous versions. A compiler's output must be tailored to different processor instruction sets. That is not something an average user can deal with; nobody wants to download source code and compile it locally—neither users nor developers enjoy that. For this reason, Sun originally introduced the concepts of Java, the JVM, and bytecode. Bytecode is an abstraction over hardware processor instructions: the Java compiler compiles Java source code into bytecode, and the JVM executes that bytecode. This allows the JVM to optimize and adapt to the underlying hardware without requiring the bytecode to be recompiled. At the same time, bytecode is difficult to decompile back into original Java source code, which keeps developers happy too.

Following this model, we will also assume that our compiler outputs an intermediate representation—which, in the current implementation, is called `primitive`.
