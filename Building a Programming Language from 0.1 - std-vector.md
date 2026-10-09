# Building a Programming Language from 0.1 - std/vector

At this point, b2style can already support relatively complex data structures. We can start with a `vector`-like class. It is called `vector`-like because b2style does not yet support returning references, so methods such as `reference at()`, `reference operator[](size_type)`, `reference front()`, and `reference back()` cannot be implemented for now; instead, we implement simple `T get(int)` and `void set(int, T)` methods.

First, let's take a look at the entire [osi.net/vector at master · Hzj-jie/osi.net](https://github.com/Hzj-jie/osi.net/blob/master/service/compiler/b2style/lib/std/vector):

```cpp
#ifndef B2STYLE_LIB_STD_VECTOR
#define B2STYLE_LIB_STD_VECTOR

#include <b2style.h>
#include <assert.h>
#include <std/types.h>

namespace std {

template <T>
class vector {
  T _a;
  int _s;
  int _c;

  int size() {
    return this._s;
  }

  int capacity() {
    return this._c;
  }

  T get(int index) {
    ::assert(index >= 0, "get");
    ::assert(index < this.size(), "get");
    return this._a[index]; 
  }

  void set(int index, T v) {
    ::assert(index >= 0, "set");
    ::assert(index < this.size(), "set");
    this._a[index] = v;
  }

  void destruct() {
    logic "dealloc_heap std__this._a";
  }

  void reserve(int size) {
    T x[size];
    for (int i = 0; i < this.size(); i++) {
      x[i] = this.get(i);
    }
    if (this._a != -1) {
      this.destruct();
    }
    this._a = x;
    logic "undefine std__x";
    this._c = size;
  }

  void construct(int size) {
    this._a = -1;
    this.reserve(size);
  }

  void construct() {
    this.construct(4);
  }

  void push_back(T v) {
    if (this.size() == this.capacity()) {
      this.reserve(this.capacity() << 1);
    }
    this._s++;
    this.set(this.size() - 1, v);
  }

  void pop_back() {
    ::assert(this.size() > 0, "pop_back");
    this._s--;
  }

  void clear() {
    this._s = 0;
  }
};

}  // namespace std

#endif  // B2STYLE_LIB_STD_VECTOR
```

There is nothing particularly special here, except that `void reserve(int)` and `void destruct()` use two `logic` statements to release the heap pointer and deallocate the heap space.

Let's write a simple test in [service/tests/compiler/b2style/self-test/vector-test.txt](https://github.com/Hzj-jie/osi.net/blob/master/service/tests/compiler/b2style/self-test/vector-test.txt):

```cpp
#include <b2style.h>
#include <std/vector>
#include <testing.h>

void main() {
  std::vector<int> v();
  for (int i = 0; i < 10; i++) {
    v.push_back(i + 1);
  }
  b2style::testing::assert_equal<int>(v.size(), 10);

  for (int i = 0; i < 10; i++) {
    b2style::testing::assert_equal<int>(v.get(i), i + 1);
  }

  for (int i = 0; i < 10; i++) {
    v.set(i, i + 2);
  }

  for (int i = 0; i < 10; i++) {
    b2style::testing::assert_equal<int>(v.get(i), i + 2);
  }

  v.clear();
  b2style::testing::assert_equal<int>(v.size(), 0);

  b2style::testing::finished();
}
```
