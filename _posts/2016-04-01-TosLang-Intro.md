---
title: Let's implement a language - Part 0 - Intro
permalink: /toslang-intro/
categories: [TosLang]
excerpt: In which I explain WTF is TosLang and why I subject myself to the hardships of compiler development. 
---

Hey everyone!

Starting this month, I'll take you through my personal journey to develop a compiler for a language of my own design. While the next posts in this series will be quite technical, I'd like to start off by offering a global view of what I'll try to accomplish and the reasoning behind some of my decisions.

## Story time

As I've previously mentionned ([right here](http://faouellet.github.io/csgames-intro/)), I was involved in the 2015 edition of the CS Games where one of the things I did was co-orgnanizing the parallelism challenge. But this wasn't the entire extension of my involvement. My other major involvement was helping with the implementation of the OS challenge. You see, at first, the idea that the organizers of the CS Games had was to simply take the homework of the OS course given at our university and adapt them to fit the 3 hours timeslot of a CS Games challenge. This was a bad decision because of two reasons. First, it would give an unfair advantage to our university students since, in a way, they would have already been exposed to the challenge. Second, [NachOS](https://homes.cs.washington.edu/~tom/nachos/).

{%
    include figure.html
    src="/assets/img/nachos.jpg"
    caption="NachOS hates puny humans."
%}

For those of you that naver got to play with NachOS, let's just say that while it might have been a good tool to teach OS concepts (maybe over a decade ago!), nowadays it teaches more about the hardships of dealing with legacy software than anything else.

With this in mind, the organizer of the OS challenge decided to say "fuck this, I'll do my own thing". This gave birth to [Tostitos](https://github.com/faouellet/Tostitos), a new hobbyist OS to be used for teaching OS concepts. Tough OS is a generous term in this case, since it is more of platform to build an OS than a true OS. Thus, it is closer in spirit to an [exokernel](https://en.wikipedia.org/wiki/Exokernel) or a [unikernel](http://queue.acm.org/detail.cfm?id=2566628).

Being good friend with the guy, I somehow ended up being dragged into the challenge organization because of my knowledge of virtual machines and multithreading. This resulted in me implementating the virtual machine Tostitos runs on and being responsible for everything thread-related in Tostitos.

Fast-forward a couple months, the CS Games are over and the feedback that we got about Tostitos is really positive. So why not go ahead and improve Tostitos so that one day it could replace NachOS in our university OS course? Being unable to come up with a reason to stop, we went on to build something that looked less like a proof of concept and more like a real OS.

Along the way, I proposed that we create a programming language to be used on top of Tostitos that could help improve the students' testing experience when developping new features as part of programming assignments. This effort came to be known as TosLang.

## WTF is TosLang?

TosLang is a small general programming language inspired by [C++](http://www.stroustrup.com/C++.html), [Chapel](http://chapel.cray.com/) and [Rust](https://www.rust-lang.org/). Like its ancestors, it has a static type system meaning everything about types has to be known (or deduced! But we'll save that for a future version...) at compile time. Also as its ancestors, each statement in the language must be terminated with a ```;```.

For builtin types, it offers three, namely ```Int```, ```Bool``` and ```String```. Each of them can be used as the type of elements in an unidimensional array (no support for multidimensional array is planned yet).

```cpp
var MyIntVar: Int = 42;
var MyIntArrayVar: Int[3] = { 1, 2, 3 };
var MyTrueVar : Bool = True;
var HelloWorldVar : String = "Hello World!";
```

It also offers two control structures: ```if``` (though no ```else``` for the moment) and ```while```. This limited offer is to keep the implementation effort of a first version of the language low while still having a Turing complete language.

```cpp
if isGreater(SomeVar, OtherVar) {
    MyIntVar = 42;
}
```

```cpp
while SomeVar < 10 {
    MyIntVar = MyIntVar + SomeVar;
    SomeVar = SomeVar - 1;
}
```

Evidently, you can define functions with the language as evidenced by the next example (and they can be recursive!).

```cpp
fn fibonacci(i : Int) -> Int {
    if i < 2 {
        return i;
    }
    return fibonacci(i-1) + fibonacci(i-2);
}
```

There are two builtin functions in the language: ```print``` and ```scan``` to allow interactions with the standard output. Later on, the ability to read and write from a file (managed by Tostitos) will be added. For the moment, I just want to make those two works to have experience with builtin functions and I'm also not totally decided yet on the file I/O functions' declarations.

```cpp
fn main() -> Void {
    var Message: String;

    scan Message;
    print Message;

    return;
}
```

Last but definitely not least, TosLang will include some keywords to manage threads i.e. parallelism will be a concept in the language itself. Unfortunately, this part is still at the analysis stage so the example below might not reflect the final product, but it may look like this (inspired by [Cilk](https://www.cilkplus.org/)).

```cpp
fn fibonacci(i : Int) -> Int {
    if i < 2 {
        return i;
    }

    var FibMinOne : Int = spawn fibonacci(i-1);
    var FibMinTwo : Int = spawn fibonacci(i-2);

    sync;

    return FibMinOne + FibMinTwo;
}
```

As you can probably tell, as of this writing, the TosLang language and its compiler are still a work in progress, which leads us to the next topic.

## Look ma no hands!

Here's where it gets really interesting. **The implemention of the TosLang compiler is done using only (modern) C++ and its standard library**. That means no lexer or parser generator like Flex or Bison and no pre-baked code generator such as LLVM. While these are perfectly fine tools to get the job done, I wanted to keep the compiler's dependencies to a minimum so that students don't have to struggle during Tostitos' installation. Also, on a more personal note, I've always wanted to build a compiler from scratch, so having a chance to do just that, I'm not going to let it pass up.

## So what's next?

In short, everything. Over the next months (years?) I plan to document my every step of building the TosLang compiler. This includes discussing not only the expected compiler related subjects (lexing, parsing, code generation, etc.) but also the architecture of the compiler itself, some nice idioms and patterns I'll be putting to use and also some alternative implementations of certain parts of the compiler.

**Coming up next month**: Enough talk, let's start by building a lexer!
