---
title: Python-like enumeration in C++
permalink: /enumerate/
tags: [C++]
excerpt: Let's make C++ more pythonic!
---

## Story time

During a recent conversation with one of the architects at the company I work for, an interesting topic came up. While talking, about range-based for loops, he mentionned that there was quite a few fellow developpers that would love to have access to the index of an element without having to declare a variable outside the loop and manually managing it inside the loop.

In other words, they wish for the Python's enumerate function.

```python
some_list = ['a','b','c']
for idx, value in enumerate(some_list):
  print(idx, value)
```

{%
    include figure.html
    src="/assets/img/enumerate_py.png"
    caption="Enumerating in Python"
%}

While not the definitive implementation, I thought I'd share my design for such a function.

## What's in a range-based for loop?

The first thing to consider in our efforts is what makes up a range-based for loop. Essentially, it is syntactic sugar for the following ([note that this is a simplification and not precisely what's described in the C++ standard](http://arne-mertz.de/2015/07/new-c-features-stdbeginend-and-range-based-for-loops/)):

```cpp
auto && __range = range_expr ;
for (auto __it = begin(__range), __e = end(__range);
     __it != __e;
     ++__it)
{
  element_decl = *it;
  statement
}
```

To put it in simpler terms, a range-based for loop is a construct that facilitates the iteration over the full range of a sequence variable. To do so, it will invokes `std::begin` and `std::end` to get the limits of the sequence. It'll then construct a variable for us to use and we'll then be able to go on using it in the statements making up the loop body.

Note that the reason for using non-members `begin` and `end` is to make sure that we can generically get access to the endpoints of a sequence. We can't expect every sequence type to have `begin` and `end` member functions (for example, an array doesn't have these functions).

What we have to remember from that is that all it takes for a variable to be used in a range-based for loop are valid invocations of `std::begin` and `std::end` on it. With that in mind, let's begin our work.

## An enumerate iterator

The first thing we'll do is create a custom iterator that will be able to give us not only an element in a sequence but also the index of said element in the sequence. Since we want it to work for multiple types of iterators, we'll define it as follows:

```cpp
template <typename IteratorT>
class EnumerateIterator
  : std::iterator<std::forward_iterator_tag,
                  typename std::iterator_traits<IteratorT>::value_type>
```

Note that we could easily derive a bidirectional iterator from it but we won't do so for brevity's sake. I just wanted this implementation of the iterator to be as short and to the point as possible. (It's definitively not because I'm lazy!)

Now, our `EnumerateIterator` will be quite simple. It will wrap an iterator `mItr` and keep track of the current index (`mCurIdx`) we're at in a sequence.

```cpp
private:
  size_t mCurIdx;
  IteratorT mItr;
```

Let's add some always useful internal types to document the class.

```cpp
public:
  using ValueT = typename std::iterator_traits<IteratorT>::value_type;
  using IdxValPair = std::pair<size_t, ValueT>;
```

We'll define two constructors for our custom iterator. The first will take an iterator by forwarding reference. The purpose of this constructor will be to create an ```EnumerateIterator``` corresponding to the beginning of a sequence.

```cpp
explicit EnumerateIterator(IteratorT&& iterator)
  : mCurIdx{ 0 }, mItr{ std::forward<IteratorT>(iterator) } { }
```

The second constructor is essentially the same as the first except for the fact that'll allow you to specify the current index in a sequence associated with the iterator you've given it. This will be quite useful when we'll want to create an ```EnumerateIterator``` corresponding to the end of a sequence.

```cpp
EnumerateIterator(IteratorT&& iterator, size_t startingCount)
  : mCurIdx{ startingCount }, mItr{ std::forward<IteratorT>(iterator) }
  { }
```

Next, as per the inheritance contract, we need to define the iterator's operators. The first ones we'll define are the increment operators. For the pre-increment operator, we'll simply increment the underlying iterator while also incrementing our index count. Having done that, the post-increment operator is trivial to implement by making use of the pre-increment operator.

```cpp
EnumerateIterator& operator++()
{
  ++mItr;
  ++mCurIdx;
  return *this;
}

EnumerateIterator operator++(int)
{
  auto temp{ *this };
  operator++();
  return temp;
}
```

Next, we'll define the equality operators. We'll say that two ```EnumerateIterator``` are equal if they both wrap the same underlying iterator and they both are at the same index in a sequence.

```cpp
bool operator==(const EnumerateIterator& enumItr) const
{
  return (mCurIdx == enumItr.mCurIdx) && (mItr == enumItr.mItr);
}

bool operator!=(const EnumerateIterator& enumItr) const
{
  return !(*this == enumItr);
}
```

Finally, we'll write the dereference operators to produce the expected index-value pair.

```cpp
IdxValPair operator*()
{
  return std::make_pair(mCurIdx, *mItr);
}

IdxValPair operator*() const
{  
  return std::make_pair(mCurIdx, *mItr);
}
```

## Wrapping it up

While having a custom iterator that can give out both an element in a sequence and the index of said element is quite nice, it's unfortunately not enough for what we want. Remember, we want to be able to loop over index-element pair coming from any sequence through a function call as can be done in Python.

To this end, we'll introduce a wrapper over a sequence: ```EumerateWrapper```.

```cpp
template <typename T> struct EnumerateWrapper { T& range; };
```

With this wrapper, we can finally write the desired ```Enumerate``` function. Basically, this function will perform a cast to transform a value of some type ```T``` to a ```EumerateWrapper```.

```cpp
template <typename T>
EnumerateWrapper<T> Enumerate(T&& range) { return{ range }; }
```

I'm sure many of you see what's coming. As mentionned previously, if we can invoke ```std::begin``` and ```std::end``` on a variable, it's good enough for it to be used in a range-based for loop. Thus, the final step in creating a Python-like ```enumerate``` will be to define those functions for ```EumerateWrapper```.

First up, the ```begin``` function. Here, we create an ```EumerateIterator``` as a wrapper over the ```begin``` iterator of the sequence contained in the ```EumerateWrapper```. Note the use of ```auto``` as the return type. This is done to avoid any hassle over what the type iterator wrapped by ```EumerateIterator``` might be.

```cpp
template <typename T>
auto begin(EnumerateWrapper<T> wrapper)
{
  return EnumerateIterator<decltype(std::begin(wrapper.range))>(
               std::begin(wrapper.range));
}
```

For the ```end``` function we will make use of the second constructor we defined. As previously, we create a ```EumerateIterator``` by wrapping the iterator of interest in the sequence inside an ```EumerateWrapper```. We also provide the size of the sequence as an argument. This will indicate what the ```end``` index is. To do so generically, we make use of the ```std::distance``` function.

```cpp
template <typename T>
auto end(EnumerateWrapper<T> wrapper)
{
  return EnumerateIterator<decltype(std::end(wrapper.range))>(
               std::end(wrapper.range),
               std::distance(std::begin(wrapper.range),
                             std::end(wrapper.range)));
}
```

## Putting it to use

With all the work done so far, we can finally write the following pythonic code:

```cpp
int main()
{
  int arr[] = { 0,1,2,3,4 };
  std::vector<int> vec{ 5,6,7,8,9 };

  std::cout << "Array enumeration:\n";
  for (const auto& idxVal : Enumerate(arr))
    std::cout << "(" << idxVal.first << ", " << idxVal.second << ")\n";
  std::cout << "\n";

  std::cout << "Vector enumeration:\n";
  for (const auto& idxVal : Enumerate(vec))
    std::cout << "(" << idxVal.first << ", " << idxVal.second << ")\n";
}
```

{%
    include figure.html
    src="/assets/img/enumerate_cpp.png"
    caption="Enumerating in C++"
%}

## The future

While reading up on what was voted in for the C++17 standard, I realized that an even more Pythonic way of writing the preceding example will soon be available to C++ developpers. By using what is called structured bindings, basically a way to initialize multiple values of possibly different types simultaneously, we'll be able to rewrite the previous example as follow:

```cpp
int main()
{
  int arr[] = { 0,1,2,3,4 };
  std::vector<int> vec{ 5,6,7,8,9 };

  std::cout << "Array enumeration:\n";
  for (const auto& [idx, val] : Enumerate(arr))
    std::cout << idx << ", " << val << "\n";
  std::cout << "\n";

  std::cout << "Vector enumeration:\n";
  for (const auto& [idx, val] : Enumerate(vec))
    std::cout << idx << ", " << val << "\n";
}
```
