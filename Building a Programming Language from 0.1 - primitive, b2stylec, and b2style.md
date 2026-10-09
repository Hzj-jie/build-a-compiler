# Building a Programming Language from 0.1 - primitive, b2stylec, and b2style

At this point, the processor, compiler, and interpreter can all be implemented simply by wiring up the existing code.

Let's look at the interpreter first, as it is straightforward. The interpreter takes `b2style` source code as input and executes the compiled result on the `primitive` processor. Because it uses the default `primitive` interrupt handlers, standard input and output are available out of the box.

Here is a simple example: [production/b2style/1-to-100.cc](https://github.com/Hzj-jie/osi.net/blob/master/production/b2style/1-to-100.cc)

```cpp
void main() {
  int x = 0;
  for (int i = 0; i < 100; i = i++) {
    x = x + i + 1;
  }
  b2style::int_std_out(x);
  b2style::std_out("\n");
}

```

The code above can be executed directly via the `b2style` command-line tool: [production/b2style/b2style.vb](https://github.com/Hzj-jie/osi.net/blob/master/production/b2style/b2style.vb)

```text
...\b2style\bin\Release>osi.production.b2style.exe ..\..\1-to-100.cc
5050

```

However, repeating the compilation process every time is wasteful. For instance, running the operation above 2,000 times takes about 4.5 minutes, or roughly 130 milliseconds per compile-and-run cycle.

```text
$ time ./osi.production.b2style.exe ../../1-to-100.cc > /dev/null

real    4m21.686s
user    0m0.000s
sys     0m0.015s

```

Therefore, we can compile `b2style` code into a `primitive` instruction sequence, output it as a binary file, and then run it directly on the `primitive` processor. This gives rise to two separate command-line tools: `b2stylec` and `primitive`.

Slightly different from `b2style`, `b2stylec` outputs a binary or text-based `primitive` instruction sequence. [production/b2stylec/b2stylec.vb](https://github.com/Hzj-jie/osi.net/blob/master/production/b2stylec/b2stylec.vb)

Meanwhile, `primitive` accepts a binary or text-based `primitive` instruction sequence and executes it directly. [production/primitive/primitive.vb](https://github.com/Hzj-jie/osi.net/blob/master/production/primitive/primitive.vb)

Executing the precompiled code 10,000 times this way takes about 47 seconds—just 4.7 milliseconds per run.

```text
$ time ./osi.production.primitive.exe 1-to-100.primitive > /dev/null

real    0m46.629s
user    0m0.000s
sys     0m0.015s

```

With this, `b2style` is essentially complete. What remains is:

- Providing more built-in `b2style` functions.
- Implementing comments and `include` functionality.
- Adding floating-point support to the `primitive` processor. While the `bstyle` language syntax supports `float`, the `primitive` processor cannot yet perform floating-point arithmetic.
