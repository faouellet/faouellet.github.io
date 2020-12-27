---
layout: post
title: Adding coroutines to LLVM Kaleidoscope
permalink: /kaleidoscope-coro/
published: false
categories: ['C++', 'LLVM', 'Compiler Development']
---

# Lexer extension

# Parser extension

# Code generation

NOTES: 

- introducing CoroutineCreator to have on demand resume block generation
- cloning functions to change return type from double default to coroutine handle type
- dead argument elimination for cloning function
- introducing suspend calls at IR level to simplify subsequent coroutine calls
  and coroutine destruction
- yield yields its expression's value so you can yield yields (sup dawg!)