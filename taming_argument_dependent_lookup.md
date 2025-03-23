@SRNissen on [github](github.com/SRNissen/) and [twitter](https:://www.twitter.com/SRNissen)


# Taming argument-dependent lookup

I am aiming this text at *library developers*.

I use this technique to solve the problem where, when my library introduces a new function with a better match, ADL silently breaks my users' code.

With this technique, ADL never selects my version, which must instead either be called with the qualified name or introduced explicitly with `using lib::my_func`.

If you already understand ADL, you can skip the explanation and go straight to [the technique](#the-technique)

## What/why argument-dependent lookup?

C++ needs a way to figure out what to do when you write `auto a = b + c`. If `b` and `c` are fundamental types (`int`, `char const*`, etc.) ADL doesn't happen and doesn't matter.

If `b` and `c` are *not* fundamental types, their `operator+` is a function.

To make this work, C++ has ADL. For every function call you write in your source code, the compiler will *look up* which function to use depending on the *arguments* to the function, hence the name.

```c++
00 import std; // for std::string
01
02 namespace lib {
03 
04         struct S {
05                 S(std::string s):s_{s}{}
06                 std::string s_;
07                 auto size() { return s_.size();}
08         };
09
10         auto min(lib::S lhs, lib::S rhs) {
11
12                 return lhs.size() < rhs.size() ? lhs.size() : rhs.size();
13         }
14 }
15 
16 int main() {
17
18         using lib::min;
19         std::string a = "bbb";
20         std::cout << min(a,a);
21 }
```
[in godbolt](https://godbolt.org/z/c74h1xbh8)

Comparing `a` with `a` yields `a`, so we should see `3`. Unfortunately, the program prints `bbb`.

Function lookup for line 32 goes something like:

1. `int lib::min(lib::S, lib::S)` is found because it is introduced by explicit name on line 18.
2. `T const& std::min(T const&, T const&)` is found because the type of `a`, `std::string`, is from `namespace std`.
3. To use `lib::min`, we would need an implicit conversion from `std::string` to `lib::S`.
4. To use `std::min`, we would require nothing. `T` is a *perfect* match for any type.
5. `std::min` is the better candidate and it returns `a` (the smaller of the two arguments) rather than `3` (the size of the smaller argument.)

This is *not new*, this is always how C++ has resolved operator overloading and, since operators are functions, how C++ has always resolved *all function overloads*.

## Adding a function - the wrong way and the right way

### Initial state

Say a user is using my library `lib`

```c++
#include <iostream>
namespace lib {
    struct S {
        std::string s;
        S(std::string s) : s{s}{}
        operator std::string() {return s;}
    };
    void print(lib::S s) {
        std::cout << s.s;
    }
}

namespace user {
    void print_twice(std::string s) {
        std::cout << s << s;
    }
    void run() {
        auto s1 = lib::S{"1"};
        auto s2 = lib::S{"2"};
        print(s1); // lib::
        print_twice(s2); // user::
    }
}

int main() {
    user::run();
}
```

To the great shame of everybody involved, my library doesn't have `print_twice`, so the user had to implement it on their own.

Today, `user::run()` prints `122`

### ADL Ruins everything for everybody

I introduce `print_twice` to my library:

```c++
#include <iostream>
namespace lib {
    struct S {
        std::string s;
        S(std::string s) : s{s}{}
        operator std::string() {return s;}
    };
    void print(lib::S s) {
        std::cout << s.s;
    }
    void print_twice(lib::S s) {
        std::cout << "twice";
    }
}
namespace user {
    void print_twice(std::string s) {
        std::cout << s << s;
    }

    void run() {
        auto s1 = lib::S{"1"};
        auto s2 = lib::S{"2"};
        
        print(s1); // lib::
        print_twice(s2); // lib:: by mistake
    }
}
int main() {
    user::run();
}
```

But oh no, that's a breaking change. Now, `user::run()` prints `1twice` instead of `122`.

### Try again - Without ADL ruining everything for everybody

```c++
#include <iostream>
namespace lib {
    struct S {
        std::string s;
        S(std::string s) : s{s}{}
        operator std::string() {return s;}
    };
    void print(lib::S s) {
        std::cout << s.s;
    }
    namespace impl {
        void print_twice(lib::S s) {
            std::cout << "twice";
        }
    }
    using namespace ::lib::impl;
}

namespace user {
    void print_twice(std::string s) {
        std::cout << s << s;
    }
    void run() {
        auto s1 = lib::S{"1"};
        auto s2 = lib::S{"2"};
        
        print(s1); // lib::
        print_twice(s2); // user:: (as intended)
        lib::print_twice(s2); // lib:: explicitly intended
    }
}

int main() {
    user::run();
}
```

Now, `user::run()` prints `122twice`.

Side-by-side on godbolt: [https://godbolt.org/z/jKqK4Wasq](https://godbolt.org/z/jKqK4Wasq)

## The technique

When I want to add a function to `namespace snns` without triggering ADL problems for my users, I don't do this:

```c++
namespace snns {
  void func(snns::type t){}
}
```
Instead, I do this:

```c++
namespace snns {
  namespace impl {
    void func(snns::type t)
  }
  using namespace ::snns::impl;
}
```

## Final notes

### Why does this work?

I honestly have no idea. Comments are *very* welcome.

ADL for `func(snns::type t)` *definitely* looks in `snns` - which it why it would find my new `func(` if I didn't do this trick. But maybe, for ADL purposes, `snns::impl::func` isn't in `snns` despite the `using namespace`?

Here's some of the reading

- https://en.cppreference.com/w/cpp/language/namespace
- https://en.cppreference.com/w/cpp/language/adl
- https://en.cppreference.com/w/cpp/language/unqualified_lookup

### Pitfalls

If I provide the user with a type `snns::impl::type`, we're right back where we started.

So I'm not going to do that.

### Disabling ADL *without* doing weird tricks

Hey you know what could be cool for some future C++ version?
```c++
namespace snns {
  adl (false) namespace {
    // ADL does not find free functions declared in this block 
    void func(snns::type t) {
      // func only found if
      //   (1) called from namespace snns, or
      //   (2) call is qualifed with namespace as snns::func(t)
      //   (3) introduced with "using snns::func"
```
At this time I have no intention of even attempting to push on that, but it seems like it would solve some issues we've been hearing about.

### Some thanks to

- Howard Hinnant, whose 2012 answer to a [stackoverflow question](https://stackoverflow.com/a/9319060) about `using std::swap` started me down this path
- Matt Godbolt for providing us with [Compiler Explorer](https://godbolt.org)
- the regulars on /r/cpp_questions [https://www.reddit.com/r/cpp_questions/]