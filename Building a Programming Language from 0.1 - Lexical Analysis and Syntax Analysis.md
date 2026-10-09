# Building a Programming Language from 0.1 - Lexical Analysis and Syntax Analysis

The `primitive` processor and the `logic` language greatly simplify the implementation of high-level languages. A high-level language compiler only needs to translate source code into `logic`, which is then compiled into the `primitive` instruction set and executed directly on the `primitive` processor. Thus, implementing a high-level language only requires lexical analysis, syntax analysis, and translating the resulting syntax tree into `logic` code.

Lexical analysis converts a sequence of characters into an array of typed tokens (words). Its interface can be summarized as follows:

```text
class typed_word {
 public:
  const string& ref;
  const size_t start;
  const size_t end;
  const size_t len;  // end - start.
  const uint type;  // A unique int to define a type.
}

bool lexer(const string& input, vector<typed_word>& result);

```

Syntax analysis, in turn, converts this array of typed tokens into a tree, commonly known as an Abstract Syntax Tree (AST). Its interface can be summarized as follows:

```text
class typed_node {
 public:
  const vector<typed_word>& ref;
  const size_t start;
  const size_t end;
  const uint type;  // A unique int to define a type.
  const typed_node* const parent;  // nullable.
  const vector<typed_node> subnodes;
}

bool syntaxer(const vector<typed_word>& words, typed_node& root);

```

Of course, errors can occur during lexical or syntax analysis, which are indicated by the boolean return value.

Lexical analysis and syntax analysis themselves are completely independent of any specific programming language or how the generated tree is subsequently processed; they belong to the domain of automata theory. While certain implementation details may influence later compilation stages, the analyzers themselves can automatically construct automata from declarative rules to match the input.
