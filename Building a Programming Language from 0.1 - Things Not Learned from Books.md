# Building a Programming Language from 0.1 - Things Not Learned from Books

I took a compiler construction course back in college, but it felt like a pretty meaningless class—most importantly, I didn't really learn anything from it.

Later, while doing HTML development, I realized that this kind of work is best suited for a language compiled at runtime. Treating web page rendering itself as data can significantly improve development efficiency. Just imagine how horrifying it would be if changing a single CSS class required recompiling the code. That led me to start researching how to implement a programming language, and I even built a simple one called `elpmis`.

Looking back now, that language had far too many bizarre implementation choices, such as fully dynamic addressing and an extremely convoluted interpretation mechanism. Still, it was at least usable.

After that, I kept exploring the subject. I took plenty of detours along the way, but finally managed to sort out the whole workflow and decided to document it here.

English version: [implement-a-compiler on github](https://github.com/Hzj-jie/implement-a-compiler)
