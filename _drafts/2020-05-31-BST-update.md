---
title: Building a build system - Part 4 - Update
permalink: /bst-update/
categories: [Building a build system]
---

Coming off the heels of the [last article in the series](), CPUs were blazing on all cylinders, the build system was scaling up, developpers were happy and managers (in a rare showing of emotions) were happy. In other words, life was good! 

... until you saw how much electricity was needed to power the multiple build cycles that happened throughout the day. Do you even know where this electricity is coming from? For all we know, all of it could be produced by companies who pride themselves by how much CO<sub>2</sub> they emit each year. Using the parallel bazooka as a first resort to swat the build system mosquito might have been going overboard. But that's what we did and now environmental doom looms over us. Bad code kills the planet.

Still, wasn't going parallel our only hope of scaling up the build system? 

No. As Yoda said, there is another...

# Identifying Changes

At the heart of everything that is to come is the realization that most of the time, changes to a project's source code won't affect everything. For instance, changing a variable's name in a higher level module of your code shouldn't have an impact on the lower level modules. That is if your code doesn't [smell](https://en.wikipedia.org/wiki/Code_smell) as bad as a dirty jockstrap. Accordingly, if we can pinpoint what has changed since the last build cycle, then we can identify the minimal sub-graph of the dependency graph to compile anew.

(TODO MINIMAL SPANNING TREE CHANGES IMAGE)

### How We Won't Solve The Problem

Before looking at my proposed approach to minimal recompilation, I want to suggest some ways this problem could be solved if our build system wasn't so minimal.

* If the build system was integrated inside another application (for instance, an IDE), it would have been relatively easy to ask other constituents of the application to keep give us the list of changes that have happened since the last compilation.

* If the build system wasn't a monolithic application, we could have considered creating a background service that would have stored a change list somewhere on disk (in some arbitrary file format or in a lightweight database).

### Serializing The Dependency Graph

In order to spot modifications to the source code, our first order of business will be to keep track of the past. In other words, we need to have a record of a previous compilation. Since we mapped compilation to a graph walk problem, it goes without saying that what we need to do is write the dependency graph to disk. The format in which you write it matters little. You can use XML or JSON or whatever else you fancy. I won't judge (openly). 

What matters though is the information that is put on disk. Evidently, you'll want to record the links between nodes. You'll also want a way to remember the file a node refers to; its absolute path would serve us well in that case. 

Most importantly, you'll want something that can uniquely identify the every part of the source code repository at the precise moment that a build cycle takes place. Why? Well, it's true that we could deduce changes just by looking at files' timestamps. If a timestamp has changed then a file's contents have necessarily changed, right? Unfortunately, no. If, after numerous modifications, a file ends up in the same state as it was when it was last compiled, we would waste CPU cycles recompiling it. This a serious setback in our fight against environmental apocalypse. 

Rather, I propose to represent the source code repository as a [Merkle tree](https://en.wikipedia.org/wiki/Merkle_tree). In other words, we'll encode a snapshot of the the repository as a tree where each leaf node will be a cryptographic hash of a file's contents and every other node will be a hash of its children's hashes. TODO

```cpp
template<typename TWriter, typename THash>
bool Write(const DependencyGraph& graph, TWriter&& writer, THash hashFunc)
{
  // First, write the dependency graph

  // Second, write a snapshot of the source code repository
}
```

# Recompiling

### Computing the change list



### Partial Graph Compilation


That's it folks!