---
title: Refactoring 101
permalink: /refactoring/
tags: [Software Development]
excerpt: Teaching pain to a new generation
---

---

All the code for this article can be found [here](https://github.com/faouellet/Refactoring).

---

## Foreword

This article is adapted from materials that I gave to students of [a course that I helped rebuild from the ground up](https://www.usherbrooke.ca/admission/fiches-cours/IGL601/). The intent is to demonstrate the whys and hows of refactoring while going through a (as close as possible) real-world example. While the article is primarily aimed at students and junior developpers, there's no reason why older folks might not appreciate it. Enjoy!

## Motivation

One does not simply refactor existing code for the fun of it. Demonstrating your "*1337 hack0rz skillz*" is also far from a good reason when trying to convince your colleagues to go ahead with a refactoring. And let's not even begin to talk about cries of "this piece of code isn't following my favorite programming convention" or "we should definitely use (insert framework *à la mode*)".

The fact of the matter is that time is valuable. You only have N working hours each day and you and your employer have to be smart on how to spend them (or not if your employer has a culture of crunch time). Therefore you have to be **1)** convinced and **2)** convincing that a given refactoring is a valuable investment. Why spend time on some already existing and functionning code instead of writing new code? Well, here's a few reasons you could give to your boss:

* **Not meeting the requirements anymore**: Times change, market opportunities change, requirements change and, logically, code will have to change to keep up. In the best of worlds, your codebase would be engineered in such a way that it could handle new requirements without much effort. You'd only need to extent some existing part of your codebase and, voila, a new cash cow feature! Unfortunately, most of us don't live in such a world. If, despite your best efforts, your codebase still refuses to accomodate evolving requirements, then a refactoring could be in order.
* **Lack of robustness**: If a particular component of your application keeps topping the charts of most buggy functionnalities as voted by your customers months after months, it might by time to look at it with a more critical eye. Troubles with your error handling or how you write data on disk can spell doom for your software. Remember, defensive programming is your friend!
* **Lack of maintainability**: Closely related to the two previous points, if almost nobody on the development team can extend a particular piece of code or fix a bug without introducing at least two new ones, then a refactor is certainly worth considering.
* **Long iteration cycles**: Here, iteration cycle means the time it takes to go through a "code-compile-test" cycle. Ideally, you'd want this cycle to be as short as possible so as not to let a developper's productivity go down the drain. To do so, you have to fight back against accumulating unnecessary complexities and dependencies in your codebase.
  {%
    include figure.html
    src="/assets/img/compiling.png"
    caption="Obligatory XKCD reference"
 %}
* **Sub-par performance**: From experience, I can tell you that when your software doesn't meets its performance expectations (either what you advertised or what your client expects), you will hear about it. **LOUDLY!** On the one hand, If you're fortunate, fixing a performance problem can be as simple as using a more suitable data structure for a particular operation. On the other end, you might need to strongly reconsider a large part of your codebase.

This is by no means an exhaustive list but it can get the ball rolling if you're in the process of building a case for your particular circumstances.

## Preparations

  {%
    include figure.html
    src="/assets/img/preparations.jpg"
    caption=""
 %}

Let's say you've convinced everyone that a refactoring is the only way forward. Do you then jump in head first? Answer: a resounding **no**! First you must make sure that you're as prepared as you can be before touching a single line of code. While there's no one size fits all preparation that will guarantee success every time (as most things in life there's no silver bullet to be found here), a good preparation will be able to answer the following questions:

* Is the feature you're about to rewrite sufficiently tested? If not, you might be committing to the programmatic equivalent of a bungee jump without a bungee cord. Good luck going all the way without breaking anything!
* Do you have a clear idea of the size of the changes you're about to make? If not, you might already be in over your head without realizing it.
* Did you validate the functionnal impacts of the incoming refactoring? If not, your clients might be in for an unwelcome surprise!
* Did you plan out the most optimal way to go about the refactoring? If not, you're probably creating more work for yourself down the line.

## Case Study

One of my goals when teaching students is to make them shed their fears of legacy code. To this end, I usually throw them deep into old and unmaintained codebase that shows its age and lack of love. It's a shock for most of them, but, then again, it's not so different than might what happen at their first day at a new job (be it an internship or their first real job).

In this instance, we'll look at a favorite of mine: the source code of the, aptly named, [Abuse](https://en.wikipedia.org/wiki/Abuse_(video_game)) video game. Some highlights:

* It was written in the mid-90s
* It is unmaintained since
* It contains a mix of C and (pre-standard) C++
* It has almost no error handling
* Its dependency network is... a sight to behold
* It has dubious memory management
* A modern compiler spews out a ton of warning when compiling it (even on lower warning levels)
* It contains an implementation of a Lisp interpreter used to run game scripts (not a con, I just find it so interesting that I have to mention it)

Of course, complety refactoring it to fix its many problems would be an exercise in madness. Especially in the scope of an university course. However, having a go at some well chosen components of Abuse is an interesting experience that can teach students how to plan and execute a refactoring. It also goes without saying that it'll also teach them some grit!

Narrowing it down for the scope of a single article, we'll look at how we can improve ```Abuse/src/lol/matrix.h``` and ```Abuse/src/lol/matrix.cpp```.

### Dead code

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- original/matrix.h
+++ diffs/1-Dead Code/matrix.h
@@ -54,14 +54,6 @@
     return *this = (*this)op val;                                              \
   }

-#define CAST_OP(elems, dest)                                                   \
-  inline operator Vec##dest<T>() const {                                       \
-    Vec##dest<T> ret;                                                          \
-    for (int n = 0; n < elems && n < dest; n++)                                \
-      ret[n] = (*this)[n];                                                     \
-    return ret;                                                                \
-  }
-
 #define OPERATORS(elems)                                                       \
   inline T &operator[](int n) { return *(&x + n); }                            \
   inline T const &operator[](int n) const { return *(&x + n); }                \
@@ -83,17 +75,6 @@
   SCALAR_OP(elems, *)                                                          \
   SCALAR_OP(elems, /)                                                          \
                                                                                \
-  CAST_OP(elems, 2)                                                            \
-  CAST_OP(elems, 3)                                                            \
-  CAST_OP(elems, 4)                                                            \
-                                                                               \
-  template <typename U> inline operator Vec##elems<U>() const {                \
-    Vec##elems<U> ret;                                                         \
-    for (int n = 0; n < elems; n++)                                            \
-      ret[n] = static_cast<U>((*this)[n]);                                     \
-    return ret;                                                                \
-  }                                                                            \
-                                                                               \
   inline T sqlen() const {                                                     \
     T acc = 0;                                                                 \
     for (int n = 0; n < elems; n++)                                            \
@@ -106,10 +87,6 @@
     return sqrtf((float)sqlen());                                              \
   }

-template <typename T> struct Vec2;
-template <typename T> struct Vec3;
-template <typename T> struct Vec4;
-
 template <typename T> struct Vec2 {
   inline Vec2() {}
   inline Vec2(T val) { x = y = val; }
@@ -120,89 +97,13 @@

   OPERATORS(2)

-  union {
-    T x;
-    T a;
-    T i;
-  };
-  union {
-    T y;
-    T b;
-    T j;
-  };
+  T x;
+  T y;
 };

 typedef Vec2<float> vec2;
 typedef Vec2<int> vec2i;

-template <typename T> struct Vec3 {
-  inline Vec3() {}
-  inline Vec3(T val) { x = y = z = val; }
-  inline Vec3(T _x, T _y, T _z) {
-    x = _x;
-    y = _y;
-    z = _z;
-  }
-
-  OPERATORS(3)
-
-  union {
-    T x;
-    T a;
-    T i;
-  };
-  union {
-    T y;
-    T b;
-    T j;
-  };
-  union {
-    T z;
-    T c;
-    T k;
-  };
-};
-
-typedef Vec3<float> vec3;
-typedef Vec3<int> vec3i;
-
-template <typename T> struct Vec4 {
-  inline Vec4() {}
-  inline Vec4(T val) { x = y = z = w = val; }
-  inline Vec4(T _x, T _y, T _z, T _w) {
-    x = _x;
-    y = _y;
-    z = _z;
-    w = _w;
-  }
-
-  OPERATORS(4)
-
-  union {
-    T x;
-    T a;
-    T i;
-  };
-  union {
-    T y;
-    T b;
-    T j;
-  };
-  union {
-    T z;
-    T c;
-    T k;
-  };
-  union {
-    T w;
-    T d;
-    T l;
-  };
-};
-
-typedef Vec4<float> vec4;
-typedef Vec4<int> vec4i;
-
 #define SCALAR_GLOBAL(elems, op, U)                                            \
   template <typename T>                                                        \
   static inline Vec##elems<U> operator op(U const &val,                        \
@@ -224,87 +125,6 @@
   SCALAR_GLOBAL2(elems, /)

 GLOBALS(2)
-GLOBALS(3)
-GLOBALS(4)
-
-template <typename T> struct Mat4 {
-  inline Mat4() {}
-  inline Mat4(T val) {
-    for (int j = 0; j < 4; j++)
-      for (int i = 0; i < 4; i++)
-        v[i][j] = (i == j) ? val : 0;
-  }
-  inline Mat4(Vec4<T> v0, Vec4<T> v1, Vec4<T> v2, Vec4<T> v3) {
-    v[0] = v0;
-    v[1] = v1;
-    v[2] = v2;
-    v[3] = v3;
-  }
-
-  inline Vec4<T> &operator[](int n) { return v[n]; }
-  inline Vec4<T> const &operator[](int n) const { return v[n]; }
-
-  T det() const;
-  Mat4<T> invert() const;
-
-  static Mat4<T> ortho(T left, T right, T bottom, T top, T near, T far);
-  static Mat4<T> frustum(T left, T right, T bottom, T top, T near, T far);
-  static Mat4<T> perspective(T theta, T width, T height, T near, T far);
-  static Mat4<T> translate(T x, T y, T z);
-  static Mat4<T> rotate(T theta, T x, T y, T z);
-
-  void printf() const;
-
-  inline Mat4<T> operator+(Mat4<T> const val) const {
-    Mat4<T> ret;
-    for (int j = 0; j < 4; j++)
-      for (int i = 0; i < 4; i++)
-        ret[i][j] = v[i][j] + val[i][j];
-    return ret;
-  }
-
-  inline Mat4<T> operator+=(Mat4<T> const val) { return *this = *this + val; }
-
-  inline Mat4<T> operator-(Mat4<T> const val) const {
-    Mat4<T> ret;
-    for (int j = 0; j < 4; j++)
-      for (int i = 0; i < 4; i++)
-        ret[i][j] = v[i][j] - val[i][j];
-    return ret;
-  }
-
-  inline Mat4<T> operator-=(Mat4<T> const val) { return *this = *this - val; }
-
-  inline Mat4<T> operator*(Mat4<T> const val) const {
-    Mat4<T> ret;
-    for (int j = 0; j < 4; j++)
-      for (int i = 0; i < 4; i++) {
-        T tmp = 0;
-        for (int k = 0; k < 4; k++)
-          tmp += v[k][j] * val[i][k];
-        ret[i][j] = tmp;
-      }
-    return ret;
-  }
-
-  inline Mat4<T> operator*=(Mat4<T> const val) { return *this = *this * val; }
-
-  inline Vec4<T> operator*(Vec4<T> const val) const {
-    Vec4<T> ret;
-    for (int j = 0; j < 4; j++) {
-      T tmp = 0;
-      for (int i = 0; i < 4; i++)
-        tmp += v[i][j] * val[i];
-      ret[j] = tmp;
-    }
-    return ret;
-  }
-
-  Vec4<T> v[4];
-};
-
-typedef Mat4<float> mat4;
-typedef Mat4<int> mat4i;

 } /* namespace lol */
```

</div>
</details>

Good news everyone! Looking at the contents of ```matrix.h```, we can clearly see that the ```Vec3```, ```Vec4``` and ```Mat4``` classes are all dead. Nobody uses them anymore. This means that we can swiftly bury them which will greatly reduce our workload! For example, ```matrix.cpp``` just disappears because it only contained definition of the dead ```Mat4``` class. Also, while the ```Vec2``` class is still used by large swaths of the application, only its members ```x``` and ```y``` are accessed. All other members can therefore be put to the guillotine. Furthermore, there's no instance in *Abuse*'s code of casting a ```Vec2``` of some type to a ```Vec2``` of some other type. This means that we can also rid ourselves of ```CAST_OP```. As a colleague of mine once said to me: "A deleted line of code is easier to maintain than an added one".

*Do note that we can be this expeditive only because we're working in an unmaintained codebase. In the Real World&trade;, please do make sure that the owner of what you consider dead code is aware of your intentions. Sometimes a piece of code may only appear dead because the functionnality it implements is impossible to test automatically with the tools currently available to the development team. In such instances, a code coverage tool might "falsely" indicate that the code is dead because it can only be tested manually.*

{%
  include figure.html
  src="/assets/img/lovecraft.png"
  caption="Beware!"
%}

### Files organization

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/1-Dead Code/matrix.h
+++ diffs/2-Files Organization/vec2.h
@@ -18,6 +18,8 @@

 #include <cmath>

+#include "vec2fwd.h"
+
 namespace lol {

 #define VECTOR_OP(elems, op)                                                   \
@@ -101,9 +103,6 @@
   T y;
 };

-typedef Vec2<float> vec2;
-typedef Vec2<int> vec2i;
-
 #define SCALAR_GLOBAL(elems, op, U)                                            \
   template <typename T>                                                        \
   static inline Vec##elems<U> operator op(U const &val,                        \
```

```cpp
// File: vec2fwd.h

#ifndef __LOL_VEC2FWD_H__
#define __LOL_VEC2FWD_H__

namespace lol
{

template<typename T>
class Vec2;

using vec2 = Vec2<float>;
using vec2i = Vec2<int>;

} /* namespace lol */

#endif // __LOL_VEC2FWD_H__
```

</div>
</details>

Now that ```Vec2``` is all that's left in ```matrix.h```, we can rename this file to ```vec2.h``` to reflect more accuratly its contents.

The other modification that we'll perform at this stage is adding a ```vec2fwd.h``` file. As can be seen above, this file will contain a forward declaration and two aliases for ```Vec2```. Why such a change? Well, the thing to know about C++ headers is that they are nothing more that glorified copy-paste. If you ask your favorite compiler to show you what it sees when it compiles a given file, you'll be presented with a humongous amalgamation of the file and every file it includes (and every file they include and so on and so forth). In a single word: headers are **transitive**. It thus follows that if a file included by another changes, you will need to recompile both. With a deep enough inclusion network, this will make you suffer when compiling a foundational file included by many others (which is the case of ```vec2.h```).

To circumvent this nightmarish scenario, we can exploit the fact that a compiler only needs to know a type's definition when it needs to know its size. In concrete terms, it only needs to truly know a type when it is used as a member variable in another type or as a base type for a derived type. In every other cases, the compiler can be satisfied by only knowing that the type's symbol exists. This particular piece of information can be made available to the compiler by forward declaring a type. ```vec2fwd.h``` is thus expected to be favored over ```vec2.h``` when possible. In an ideal world, only ```.cpp``` files would include ```vec2.h```. Since we don't typically include ```.cpp``` in other files, this would keep recompilations to a minimum.

The same idea applies to the ```vec2``` and ```vec2i``` aliases. We want to have them exposed in a file without a lot of moving parts.

### Class

With all the cruft that had accumulated in these files removed, we can focus our attention on the ```Vec2``` struct itself.

#### Data and data access

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/2-Files Organization/vec2.h
+++ diffs/3-Class contents/vec2.h
@@ -16,7 +16,10 @@
 #if !defined __LOL_MATRIX_H__
 #define __LOL_MATRIX_H__

+#include <array>
 #include <cmath>
+#include <numeric>
+#include <type_traits>

 #include "vec2fwd.h"

@@ -75,34 +78,52 @@
   SCALAR_OP(elems, -)                                                          \
   SCALAR_OP(elems, +)                                                          \
   SCALAR_OP(elems, *)                                                          \
   SCALAR_OP(elems, /)                                                          \
                                                                                \
-  inline T sqlen() const {                                                     \
-    T acc = 0;                                                                 \
-    for (int n = 0; n < elems; n++)                                            \
-      acc += (*this)[n] * (*this)[n];                                          \
-    return acc;                                                                \
-  }                                                                            \
-                                                                               \
-  inline float len() const {                                                   \
-    using namespace std;                                                       \
-    return sqrtf((float)sqlen());                                              \
-  }

-template <typename T> struct Vec2 {
+template <typename TVec> class Vec2 {
+    static_assert(std::is_same_v<TVec, int> || std::is_same_v<TVec, float>);
+
+    using TData = std::array<TVec, 2>;
+
+public:
   inline Vec2() {}
-  inline Vec2(T val) { x = y = val; }
-  inline Vec2(T _x, T _y) {
+  inline Vec2(TVec val) { x = y = val; }
+  inline Vec2(TVec _x, TVec _y) {
     x = _x;
     y = _y;
   }

+    // Accessors
+    constexpr TVec X() const { return m_data[0]; }
+    constexpr TVec Y() const { return m_data[1]; }
+
+    constexpr TVec& X() { return m_data[0]; }
+    constexpr TVec& Y() { return m_data[1]; }
+
+    // Iterators
+    constexpr typename TData::iterator begin() noexcept              { return m_data.begin(); }
+    constexpr typename TData::iterator end() noexcept                { return m_data.end(); }
+    constexpr typename TData::const_iterator cbegin() const noexcept { return m_data.cbegin(); }
+    constexpr typename TData::const_iterator cend() const noexcept   { return m_data.cend(); }
+
+    // Length
+    TVec sqlen() const
+    {
+        return std::accumulate(m_data.cbegin(), m_data.cend(), TVec{});
+    }
+
+    float len() const
+    {
+        return std::sqrt(static_cast<float>(sqlen()));
+    }
+
   OPERATORS(2)

-  T x;
-  T y;
+private:
+    TData m_data{};
 };

+
 #define SCALAR_GLOBAL(elems, op, U)                                            \
   template <typename T>                                                        \
   static inline Vec##elems<U> operator op(U const &val,                        \
```

</div>
</details>

The first problem that we have to fix is the data access API of ```Vec2```. The fact of the matter is that ```Vec2```, in its current form, is an exhibitionist and should be arrested for indecent exposure. To fix this problem, we can simply transform it into a class and hide its members inside it. Speaking of which, we'll transform the two data members into a single one; an ```std::array``` of two elements. This then enables us to leverage some nice features of the underlying container such as its iterators. This, in turn, will allow us to use ```Vec2``` in many generic algorithms working with iterators (such as the ones in the C++ standard library). As proof, you have to look no further than the revamped ```sqlen``` method which now leverages ```std::accumulate``` to do its work.

A side benefit of reimplementing ```sqlen``` and ```len``` in terms of standard functions is that it allows us rid ourselves of pesky C-style casts. Even though they aren't that problematic in this particular instance, C-style cast should be used sparingly in C++ code. The reason why is that a C-style cast will try to succeed by any means necessary (read by trying unholy combinations of all the C++ cast operators). These casts can therefore punch a hole in C++'s type system. This is something we'd rather avoid since the type system in a statically typed language can be a tool to find potential bug. Don't break your tools!

Modifying the data members, if only to hide them away, necessarily has an impact on the data access API of ```Vec2```. Again, in an ideal world, we would not give out access to mutable references to our data. Dedicated setters would be made available instead. However, to minimize the impacts of our refactoring, we will accept this hole in our design for now. Pragmatism must prevail if we want to make it to the other side of this refactoring.

Keen-eyed readers will have taken notice of the ```static_assert``` that appeared at the start of ```Vec2```'s definition. The fact is that ```Vec2``` can't contain values of arbitrary types. For example, to allow for arithmetic operations with two ```Vec2```s, the values contained in them must be from types supporting such operations. In C++20, we can encode such requirements on types using [concepts](https://en.cppreference.com/w/cpp/language/constraints). Since we're restraining ourselves to C++17, we unfortunately can't take advantages of them. In this situation, I make the pragmatic choice to limit the accepted types to ```int``` and ```float```; the two types which we have aliases for. This should be good enough until someone wants to extend the codebase far beyond its current capabilities.

Lastly, I want to touch on the qualifiers (```constexpr``` and ```noexcept```) that decorate some of the new class methods. What they mean is of little importance in this context. You can read up on them on the internet. What is important is that these qualifiers were originally present on methods of ```std::array``` as a contract that they have to honor. In turn, we have to reflect on the possibility of ```Vec2``` providing the same guarantees as its underlying container. In theses cases, there are no reason why it couldn't so we propagate the qualifiers to ```Vec2```'s interface. This is an important question that you should ask yourselves when designing a class: how can I leverage the undelying data structures' guarantees in my new class? Remember, code, as science, is built upon the works of others.

#### Constructors

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/3-Class contents/vec2.h
+++ diffs/4-Constructors/vec2.h
@@ -86,12 +86,9 @@
     using TData = std::array<TVec, 2>;

 public:
-  inline Vec2() {}
-  inline Vec2(TVec val) { x = y = val; }
-  inline Vec2(TVec _x, TVec _y) {
-    x = _x;
-    y = _y;
-  }
+    // Ctor
+    explicit Vec2(TVec val = TVec{}) : m_data{val, val} { }
+    Vec2(TVec _x, TVec _y) : m_data{_x, _y} { }

     // Accessors
     constexpr TVec X() const { return m_data[0]; }
```

</div>
</details>

Attentive readers might have noticed that the changes made in the previous section will cause some compilation errors. Don't worry this will be fixed in the current section.

First of, we'll eliminate the default constructor since it's empty. In such a case, it's better to let the compiler generate it for us anyway.

Then, we'll simply adapt the two arguments constructor to the new ```std::array``` data member. Yep, that one is also quite straight forward.

This leaves us with the single argument constructor. Adapting it to ```m_data``` is, again, quite trivial. Yet there is more to this constructor than meets the eye. In fact, we need to make to serious decisions:

1. Do we allow implicit conversion from ```TVec``` to ```Vec2```?
2. How do we make sure that client code never uses a ```Vec2``` with invalid values?

Given the current usage of ```Vec2``` in the *Abuse* codebase, we can answer 1 with a resounding no. In no case are we justified to convert a single ```int``` or ```float``` into a ```Vec2```. If client code wants to do that, it can simply use the single argument constructor of ```Vec2```. (How I wish C++ would get rid of the implicit conversion *feature*. This would eliminate a whole class of potential bugs from otherwise working software. /rant)

Concerning 2, I see two possible answers. One would be to use [defaut member initialization](https://en.cppreference.com/w/cpp/language/data_members#Member_initialization) to make sure that even a default-constructed ```Vec2``` has valid data members. The other, which I went for, is to use a default value in the single argument constructor. Since we're in templated code, I can't give a precise default value. It will be different for each type allowed by ```Vec2```. This is why I rely on the default constructor of ```TVec``` to produce the value I require.

Whatever answer you choose for both of these questions in your future refactorings, please make sure to arm yourselves with at least one good static analyzer. Part of what we did here was to minimize misuses of ```Vec2```. A good static analyzer will be your best friend in this endeavor.

#### Indexing

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/4-Constructors/vec2.h
+++ diffs/5-Indexing/vec2.h
@@ -17,6 +17,7 @@
 #define __LOL_MATRIX_H__

 #include <array>
+#include <cassert>
 #include <cmath>
 #include <numeric>
 #include <type_traits>
@@ -68,9 +69,6 @@
   }

 #define OPERATORS(elems)                                                       \
-  inline T &operator[](int n) { return *(&x + n); }                            \
-  inline T const &operator[](int n) const { return *(&x + n); }                \
-                                                                               \
   VECTOR_OP(elems, -)                                                          \
   VECTOR_OP(elems, +)                                                          \
   VECTOR_OP(elems, *)                                                          \
@@ -98,6 +96,13 @@
     explicit Vec2(TVec val = TVec{}) : m_data{val, val} { }
     Vec2(TVec _x, TVec _y) : m_data{_x, _y} { }

+    // Indexing
+    constexpr const TVec& operator[](int n) const
+    {
+        assert(0 <= n && n < 2);
+        return m_data[n];
+    }
+
     // Accessors
     constexpr TVec X() const { return m_data[0]; }
     constexpr TVec Y() const { return m_data[1]; }
```

</div>
</details>

Once again, the changes applied here are borderline trivial. This make it the perfect moment to take the time to talk about error handling. Looking at the original indexing operators, we can see that both safety and security were thrown out the window. Pass any index you want, they'll take it!

So, how can we do better? Well, there's plenty of choices (and you don't necessarily need to use only one!):

* **Exceptions**: While they don't have the best of reputation in C++, exceptions still have their legitimate uses. The key is to use them in *exceptional* situations. I know this sounds obvious but I've seen time and time again code that used exceptions as the only mean of communicating errors. Client pressed a wrong key? Exception! Client made a typo? Exception! Client clicked on the wrong widget? Exception!
{%
  include figure.html
  src="/assets/img/exception.jpg"
  caption=""
%}
* **Suicide**: You've read that right. Let's just crash the whole system! Though it might seem extreme at first glance, this is actually a very valid strategy for systems which can't or won't accept any erroneous state. It can also be an acceptable solution for when the program's state has become too corrupted to continue safely. Be aware that this approach is best suited for systems that can reboot themselves (such as a [Mars Rover](https://www.youtube.com/watch?v=3SdSKZFoUa8)) or that can still produce an error message to inform the user that everything is doomed beyond salvation (such as the classic BSoD).
* **Error code**: A tried-and-true technique from ages past which can still serve today. Combined with a good atlas of messages to inform a user, these codes can be a perfectly acceptable choice for many applications. They do have some problems though. For example, if you want to return values other than the error code itself, you need to have output parameters which can sometimes be a hassle to deal with. Also, this technique is less suited than other for composing errors to provide a clearer picture of what failed.
* **Error type**: This can be considered an evolution of the humble error code. With an error type, you can have rich (and possibly hierachical) errors that would contain all the information needed for the client to either fix reported error or fill out a detailled error report. What's more, you can add validations within the type's methods (in the form of assertions or by using the ```[[nodiscard]]``` attribute) to make sure that no error is forgotten. Unfortunately, this still doesn't fix the need for output parameters when you when to return values out of a function.
* **Optional value**: This can either be a ```std::optional``` or a good old pointer. What's nice with this approach is that we longer have to deal with output parameters. What's less nice is that we lose information about what problems were encountered during a function call. We have no value and no explanations for the caller. This can be somewhat mitigated if there's only a single cause of error, but it illustrate the limited use cases of this approach.
* **Result type**: A type that would combine the error type and the optional value you say? I'd say you might be on to something! In all seriousness, this is indeed an area that recently got a lot of attention (just look at [```std::expected```](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0323r7.html) or Rust's [```Result```](https://doc.rust-lang.org/std/result/enum.Result.html)). If you choose this option, make sure that the type you're using is as lean as possible since it will most likely be plastered all over your codebase. You don't want it to become a performance bottleneck.

What did I choose in the end? None of the above! You see choosing the error handling model of your codebase (or just part of it) is something ripe with consequences. Expect an avalanche of changes once your choice is made. Since we're focusing on a single, albeit foundational, class we want to contain the ripple effects of our changes on the rest of the codebase. As was mentionned previously, when doing a refactoring, you need to control the scope of your changes otherwise you'll bite off more than you can chew.

Still, I did add an assertion to detect potential problems during development. With an extensive test suite, this could even be good enough to ship as is!

#### Comparison and equality operators

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/5-Indexing/vec2.h
+++ diffs/6-Comparison operators/vec2.h
@@ -19,6 +19,8 @@
 #include <array>
 #include <cassert>
 #include <cmath>
+#include <functional>
+#include <limits>
 #include <numeric>
 #include <type_traits>

@@ -40,14 +42,6 @@
     return *this = (*this)op val;                                              \
   }

-#define BOOL_OP(elems, op, op2, ret)                                           \
-  inline bool operator op(Vec##elems<T> const &val) const {                    \
-    for (int n = 0; n < elems; n++)                                            \
-      if (!((*this)[n] op2 val[n]))                                            \
-        return !ret;                                                           \
-    return ret;                                                                \
-  }
-
 #define SCALAR_OP(elems, op)                                                   \
   inline Vec##elems<T> operator op(T const &val) const {                       \
     Vec##elems<T> ret;                                                         \
@@ -66,18 +60,27 @@
   VECTOR_OP(elems, *)                                                          \
   VECTOR_OP(elems, /)                                                          \
                                                                                \
-  BOOL_OP(elems, ==, ==, true)                                                 \
-  BOOL_OP(elems, !=, ==, false)                                                \
-  BOOL_OP(elems, <=, <=, true)                                                 \
-  BOOL_OP(elems, >=, >=, true)                                                 \
-  BOOL_OP(elems, <, <, true)                                                   \
-  BOOL_OP(elems, >, >, true)                                                   \
-                                                                               \
   SCALAR_OP(elems, -)                                                          \
   SCALAR_OP(elems, +)                                                          \
   SCALAR_OP(elems, *)                                                          \
   SCALAR_OP(elems, /)

+namespace details
+{
+    template <typename TVec, typename TFunc>
+    inline bool compareVector(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs, TFunc&& func)
+    {
+        for(int iElem = 0; iElem < 2; ++iElem)
+        {
+            if(!func(lhs[iElem], rhs[iElem]))
+            {
+                return false;
+            }
+        }
+        return true;
+    }
+}
+
 template <typename TVec> class Vec2 {
     static_assert(std::is_same_v<TVec, int> || std::is_same_v<TVec, float>);

@@ -125,6 +128,26 @@
     TData m_data{};
 };

+// Equality operators
+inline bool operator==(const Vec2<int>& lhs, const Vec2<int>& rhs)
+{
+    return details::compareVector(lhs, rhs, std::equal_to<int>{});
+}
+inline bool operator==(const Vec2<float>& lhs, const Vec2<float>& rhs)
+{
+    return details::compareVector(lhs, rhs, [](float lhs, float rhs){ return std::fabs(lhs - rhs) > std::numeric_limits<float>::epsilon(); } );
+}
+template<typename TVec> inline bool operator!=(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs) { return !(lhs == rhs); }
+
+// Comparison operators
+template<typename TVec> inline bool operator<(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs)  
+{
+    return details::compareVector(lhs, rhs, [](TVec lhs, TVec rhs){ return lhs < rhs; });
+}
+template<typename TVec> inline bool operator<=(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs) { return !(lhs > rhs); }
+template<typename TVec> inline bool operator>=(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs) { return !(lhs < rhs); }
+template<typename TVec> inline bool operator>(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs)  { return rhs < lhs; }
+

 #define SCALAR_GLOBAL(elems, op, U)                                            \
   template <typename T>                                                        \                                                    \
```

</div>
</details>

I won't go into the details of why you should prefer to write equality and comparison operators as non-member functions since there's already [a quite thorough explanation of SO](https://stackoverflow.com/questions/4421706/what-are-the-basic-rules-and-idioms-for-operator-overloading/4421729#4421729). I will however point out that when implementing these operators, you only really have to implement two of them (```operator==``` and ```operator<```) since every other can implemented in terms of these two. Once more, I encourage you to be lazy and write as less code as possible. This will help you save up on maintainance in the future.

Speaking of being lazy, we can implement both ```operator==``` and ```operator<``` in a single free function that will invoke a callable (either an equality or comparison) on a pair of elements from two ```Vec2```. Since we need to implement it in ```vec2.h``` (it needs to be templated after all), we'll also need to hide it away inside a ```details``` namespace. We wouldn't want to expose implementation details to client code. Some may argue that this is a weak attempt at encapsulation and they would be right. Still, it's the best we can do until [the standard modules](https://en.cppreference.com/w/cpp/language/modules) are made available to all.

#### Mathematical operations

<details>
<summary><b>Changes</b></summary>
<div markdown="1">

```diff
--- diffs/6-Comparison operators/vec2.h
+++ diffs/7-Mathematical operators/
@@ -28,43 +28,6 @@

 namespace lol {

-#define VECTOR_OP(elems, op)                                                   \
-  template <typename U>                                                        \
-  inline Vec##elems<T> operator op(Vec##elems<U> const &val) const {           \
-    Vec##elems<T> ret;                                                         \
-    for (int n = 0; n < elems; n++)                                            \
-      ret[n] = (*this)[n] op val[n];                                           \
-    return ret;                                                                \
-  }                                                                            \
-                                                                               \
-  template <typename U>                                                        \
-  inline Vec##elems<T> operator op##=(Vec##elems<U> const &val) {              \
-    return *this = (*this)op val;                                              \
-  }
-
-#define SCALAR_OP(elems, op)                                                   \
-  inline Vec##elems<T> operator op(T const &val) const {                       \
-    Vec##elems<T> ret;                                                         \
-    for (int n = 0; n < elems; n++)                                            \
-      ret[n] = (*this)[n] op val;                                              \
-    return ret;                                                                \
-  }                                                                            \
-                                                                               \
-  inline Vec##elems<T> operator op##=(T const &val) {                          \
-    return *this = (*this)op val;                                              \
-  }
-
-#define OPERATORS(elems)                                                       \
-  VECTOR_OP(elems, -)                                                          \
-  VECTOR_OP(elems, +)                                                          \
-  VECTOR_OP(elems, *)                                                          \
-  VECTOR_OP(elems, /)                                                          \
-                                                                               \
-  SCALAR_OP(elems, -)                                                          \
-  SCALAR_OP(elems, +)                                                          \
-  SCALAR_OP(elems, *)                                                          \
-  SCALAR_OP(elems, /)
-
 namespace details
 {
     template <typename TVec, typename TFunc>
@@ -122,9 +85,33 @@
         return std::sqrt(static_cast<float>(sqlen()));
     }

-  OPERATORS(2)
+    // Vector operators
+    Vec2<TVec>& operator+=(const Vec2<TVec>& val) { *this = vectorOpImpl(val, std::plus<TVec>{});       return *this; }
+    Vec2<TVec>& operator-=(const Vec2<TVec>& val) { *this = vectorOpImpl(val, std::minus<TVec>{});      return *this; }
+    Vec2<TVec>& operator*=(const Vec2<TVec>& val) { *this = vectorOpImpl(val, std::multiplies<TVec>{}); return *this; }
+    Vec2<TVec>& operator/=(const Vec2<TVec>& val) { *this = vectorOpImpl(val, std::divides<TVec>{});    return *this; }
+
+    // Scalar operators
+    Vec2<TVec>& operator+=(const TVec& val) { *this = scalarOpImpl(val, std::plus<TVec>{});       return *this; }
+    Vec2<TVec>& operator-=(const TVec& val) { *this = scalarOpImpl(val, std::minus<TVec>{});      return *this; }
+    Vec2<TVec>& operator*=(const TVec& val) { *this = scalarOpImpl(val, std::multiplies<TVec>{}); return *this; }
+    Vec2<TVec>& operator/=(const TVec& val) { *this = scalarOpImpl(val, std::divides<TVec>{});    return *this; }

 private:
+    template<typename TFunc>
+    Vec2<TVec>& vectorOpImpl(const Vec2<TVec>& val, TFunc&& func)
+    {
+        std::transform(val.cbegin(), val.cend(), m_data.begin(), m_data.begin(), func);
+        return *this;
+    }
+
+    template<typename TFunc>
+    Vec2<TVec>& scalarOpImpl(const TVec& val, TFunc&& func)
+    {
+        std::transform(m_data.begin(), m_data.end(), m_data.begin(), [&func, &val](TVec elem){ return func(elem, val); });
+        return *this;
+    }
+
     TData m_data{};
 };

@@ -148,28 +135,17 @@
 template<typename TVec> inline bool operator>=(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs) { return !(lhs < rhs); }
 template<typename TVec> inline bool operator>(const Vec2<TVec>& lhs, const Vec2<TVec>& rhs)  { return rhs < lhs; }

-
-#define SCALAR_GLOBAL(elems, op, U)                                            \
-  template <typename T>                                                        \
-  static inline Vec##elems<U> operator op(U const &val,                        \
-                                          Vec##elems<T> const &that) {         \
-    Vec##elems<U> ret;                                                         \
-    for (int n = 0; n < elems; n++)                                            \
-      ret[n] = val op that[n];                                                 \
-    return ret;                                                                \
-  }
-
-#define SCALAR_GLOBAL2(elems, op)                                              \
-  SCALAR_GLOBAL(elems, op, int)                                                \
-  SCALAR_GLOBAL(elems, op, float)
-
-#define GLOBALS(elems)                                                         \
-  SCALAR_GLOBAL2(elems, -)                                                     \
-  SCALAR_GLOBAL2(elems, +)                                                     \
-  SCALAR_GLOBAL2(elems, *)                                                     \
-  SCALAR_GLOBAL2(elems, /)
-
-GLOBALS(2)
+// Vector operators
+template<typename TVec> inline Vec2<TVec> operator+(Vec2<TVec> lhs, const Vec2<TVec>& rhs) { lhs += rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator-(Vec2<TVec> lhs, const Vec2<TVec>& rhs) { lhs -= rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator*(Vec2<TVec> lhs, const Vec2<TVec>& rhs) { lhs *= rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator/(Vec2<TVec> lhs, const Vec2<TVec>& rhs) { lhs /= rhs; return lhs; }
+
+// Scalar operators
+template<typename TVec> inline Vec2<TVec> operator+(Vec2<TVec> lhs, const TVec& rhs) { lhs += rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator-(Vec2<TVec> lhs, const TVec& rhs) { lhs -= rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator*(Vec2<TVec> lhs, const TVec& rhs) { lhs *= rhs; return lhs; }
+template<typename TVec> inline Vec2<TVec> operator/(Vec2<TVec> lhs, const TVec& rhs) { lhs /= rhs; return lhs; }

 } /* namespace lol */
```

</div>
</details>

As it turns out, the mathematical operators are surprisingly easy to implement. The original implementation had the right idea: there are only two high-level operations to think about. One is the scalar operation and the other is the vector operation. Whatever concrete operation we're talking about, it's bound to be a specialization of either of these. However, what's less than ideal about the original code is that it implements these operations in terms of macros. Goodbye debuggability!  

To allow for a broad range of functions to be applied to ```Vec2```, our new operations will therefore use templates to abstract both the type of the data contained within ```Vec2``` and the operation to apply on it. As mentionned before, we only have two operations to think about: scalar (```scalarOpImpl```) and vector (```vectorOpImpl```). In both cases, we can leverage the C++ standard library, more precisely ```std::transform```, to do the work for us. Once these core operations are dealt with, all that's left is invoking them with the right function/functor for a given operator.

Also, notice how only the operators which do an assignment in addition to a given operation are implemented as member functions. This is because they're the only ones that will have an effect on ```this```. Knowing this (pun intended), I can free all other operators to reduce the API of ```Vec2``` to a minimum.

One other thing that might seem strange to some is that we take the first operand of a mathematical operator by value. The fact is that, for these operators, a new ```Vec2``` is guaranteed to be constructed. Armed with this knowledge, I've decided to create this new ```Vec2``` as soon as possible. This then shortens the implementation of the operators. Again, the less code you write, the less you have to maintain.

### Naming convention

Attentive readers might have noticed that, in every  diff that I presented up until now, I made sure to rename each and every variable that had a non-significative name to something more evocative of its purpose. Remember that a key to maintainable software is that a new developper should be able to figure out (at least at a high level) what a given piece of code does by reading it. As such, using non-descriptive names (or even worse: single letter name) is working actively against that goal.

### Documentation

The documentation of ```Vec2``` leaves a lot to be desired. What's worse is I didn't make any effort to improve the situation! If I had to do a code review on myself, I probably would have refused my patch for lack of proper documentation on at least the general intent and purpose of the class. Please be better than me on this!

## Conclusion

In lieu of a conclusion, I'd encourage you to go back and reread the **Motivation** and **Preparations** section. I hoped I've demonstrated that, when undertaking a refactoring, you need to make sure you understand what you're getting yourself into. If not, then you may be comitting [one of the worst sin of software development](https://www.joelonsoftware.com/2000/04/06/things-you-should-never-do-part-i/).
