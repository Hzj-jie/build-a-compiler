# Building a Programming Language from 0.1 - If Linked Lists Are Needed

In [Zijie He: Building a Programming Language from 0.1 - std/vector](https://zhuanlan.zhihu.com/p/478065416), b2style was already able to support a very simple `vector`-like data structure. However, compared with a full-featured language, there is still a long way to go—for instance, implementing a linked list or a binary tree is something the current b2style cannot support.

First, let's look at an example ([http://cpp.sh/5zw32](http://cpp.sh/5zw32)):

```cpp
struct S {
  int x;
  S next;
};

int main() {}
```

This example fails to compile; the compiler reports an error like `7:5: error: field 'next' has incomplete type 'S'`. However, [http://cpp.sh/6yhwm](http://cpp.sh/6yhwm) works fine:

```cpp
struct S {
  int x;
  S* next;
};

int main() {}
```

To most C and C++ programmers, this seems completely natural and nothing to be puzzled about, because in the second example, `S*` is not of type `S`, but a pointer. Syntactically, however, this pointer also carries type information along with it. If it were rewritten as `void* next;`, it would have no effect on the layout of the compiled struct—it would only affect how the field is used.

To implement similar functionality, b2style also needs to support this kind of pointer. In principle, it is the same as in standard C. For example:

```cpp
struct S {
  int x;
  string y;
  type_ptr p;
};
```

Here, `p` is a pointer type, and what it actually points to is independent of `struct S`. Moreover, when heap memory is allocated, the fields of a struct are always allocated consecutively. For example, if `S.p` is actually a pointer to an `S`, then `S.p` points to `S.x` and `S.p + 1` points to `S.y`. Therefore, as long as we know the starting address of the heap allocation and the type of the heap object, we can reconstruct all of the heap pointers for its fields.

Because b2style does not have a syntax like `S*`, the type information is lost once `type_ptr` is used. To access the corresponding heap space, we need to know the original type `S`. For example:

```cpp
class S {
  type_ptr p;

  template<T>
  T get(int index) {}

  template<T>
  void destruct() {}
}; 
```

The destructor undergoes the same change. However, this syntax is not currently supported. While supporting this syntax alone would not be difficult, combining it with inheritance is beyond what b2style can implement:

```cpp
class Base {
  template <T>
  overridable void destruct() {}  // template<T> void destruct(Base& this);
}

class Derived : Base {
  template <T>
  override void destruct() {}  // template<T> void destruct(Derived& this);
}
```

The underlying issue here has existed for a while, though we have put off dealing with it. b2style does not know the type of each value, so template functions carry no parameter type information—meaning `template <T> void f(int x)` and `template <T> void f(string x)` are treated as the exact same template. Otherwise, b2style would have no way of knowing which template function `f(1)` and `f("abc")` should each call, since function overloading is currently handled entirely in bstyle.

Now let's look at the actual operations needed when accessing heap space:

```cpp
template<T>
T get(type_ptr p, int index) {}
```

Here we must convert `type_ptr p` into a pointer of type `T`. Unlike `reinterpret_cast`, this operation needs to emit instructions, which can roughly be understood as follows:

```cpp
struct S {
  int x;
  string y;
  type_ptr p;
};

type_ptr heap_offset(type_ptr base, type_ptr offset) {}

S get(type_ptr p, int index) {
  S s_p;
  s_p.x = heap_offset(p, 0);
  s_p.y = heap_offset(p, 1);
  s_p.p = heap_offset(p, 2);
  return s_p;
}
```

Here, `type_ptr heap_offset(type_ptr base, type_ptr offset)` is a processor-dependent function. Simply put, it returns a heap location understood by the processor that points to the start of the `offset`-th heap allocation after `base`. On the primitive processor, this is simply `base + offset * MAX_UINT32`.

Note that the `get` function here cannot be implemented using a template, because a template function itself does not know how many member variables type `S` has. Thus, we have to introduce a new syntax and let the compiler generate the code above automatically. And since it generates runtime instructions, we borrow the name `static_cast` from C++.

With that, the desired "untyped pointer" looks roughly like this ([service/compiler/b2style/lib/b2style/raw_heap_ptr.h](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/b2style/raw_heap_ptr.h)):

```cpp
class raw_heap_ptr {
  type_ptr p;

  bool is_null() {
    bool r;
    logic "empty b2style__r this.p";
    return r;
  }

  template <T>
  T get() {
    type_ptr v = this.p;
    static_cast(v, T);
    return v;
  }

  void set_null() {
    type_ptr x;
    this.p = x;
  }

  // TODO: Find a good way to automatically dealloc heap resources.
  template <T>
  void destruct() {
    ::assert(!this.is_null());
    T x = this.get<T>();
    dealloc(x);
    type_ptr x;
    this.p = x;
  }

  template <T>
  void set(T t) {
    static_cast(t, type_ptr);
    this.p = t;
  }
};
```

However, implementing this in b2style is very difficult. Rather than forcing b2style to support even more complex syntax, it makes more sense to build a brand-new language—b3style—that compiles code directly into the logic language.
