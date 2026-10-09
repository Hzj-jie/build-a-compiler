# 实现一个编程语言 - b3style的基本结构

前篇[Zijie He：从0.1开始实现一个编程语言 - 如果需要链表](https://zhuanlan.zhihu.com/p/524962667)提到，b2style无法支持类的模板函数，因为生成模板的时候b2style并没有调用的函数的参数类型信息。

```cpp
// 此处应是 template <T> int f(int a, string b);
// 但b2style并不知道`1 + 1`是一个整型，或者`"abc"`是一个字符串。
int x = f<int>(1 + 1, "abc");

```

在b2style的实现中，这些完全依赖于bstyle。当然也可以把bstyle的功能再加入b2style中，比如之前的[scope.call_hierarchy](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/scope.call_hierarchy.vb)就同时在bstyle和b2style中存在，不过略微有些变化。但这样下去只会让b2style在实现上成为bstyle的一个超集，而不只是在语法上。把b2style编译成bstyle的语句，然后再编译成logic语言就变得冗余而低效了。而不如考虑实现一个b3style，

- 在词法和语法上，暂时和b2style一致。以后可能会再增加别的功能，不过这并不是当前的需求。
- 在实现上，输入的是类c++语言的代码，输出的则直接是logic语言的代码。
- 某些实现上可能会有优化，比如`#include`可以引用多个目录；在使用结构体参数时也进行`pack`和`unpack`，以减少复杂度。不过这些也不是当前的需求。
- 与b2style和bstyle完全兼容，可以直接编译这两个语言的代码。

所以可以说b3style是一个bstyle的编译器，同时增加了b2style特定的词法、语法和功能。既然“是一个”bstyle的编译器，简单看来b3style应该继承自bstyle，不过具体实现的时候并不会这么做。

再来看看现有的bstyle和b2style的实现，基本上分成三个部分：

- 词法和语法，这两个部分很简单，只需要规则文件就可以用nlexer和syntaxer了。
- 无状态的代码生成器，基本上是一个syntaxer规则对应一个代码生成器，生成这个规则的输出对应的logic语言或者bstyle语言的代码。
- scope，用以管理编译器的状态，比如存储类型、变量、模板等。

其实包括logic语言的实现也都是一样的，只是logic语言并不用nlexer和syntaxer自动机来完成输入匹配罢了。

### 词法和语法

这个什么也不用做，直接使用b2style的即可。

### 代码生成器

可以认为是bstyle的代码生成器，加上b2style的代码生成器串联bstyle的代码生成器。比如`function`就是bstyle的代码生成器，而`class`就是用b2style的代码生成器的输出串联上bstyle的`struct`和`function`的代码生成器。不过这个串联有点麻烦，比方说`variable`的代码并不被语法认可为一个`ROOT_TYPE`所以无法直接编译。而且简单的串联效率很低，绝大多数b2style的代码生成器还是需要重新实现成直接输出logic语言的模式。

### scope

大约等于bstyle.scope加上b2style.scope，两者的功能都需要，除了少量的重复。

### 由测试驱动

这是个和bstyle或者b2style不一样的地方，这两者都是由功能驱动，但b3style的实现则由测试驱动。简单说来先得实现一个兼容b2style语法的nlp，然后是兼容bstyle语法的编译器，再是兼容b2style语法的编译器，之后再考虑增加新的功能。而这三个步骤都是由测试来驱动的，基本要求就是能执行现有的测试样例并且输出的primitive指令序列一致，或者至少是运行结果一致。

指令序列一致是个强要求，需要保证实现完全一致，比如说删除无用代码逻辑的实现；而运行结果一致是一个弱要求，可以包含一些优化或折中。
