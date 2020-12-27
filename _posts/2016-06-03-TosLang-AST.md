---
title: Let's implement a language - Part 2 - The AST
permalink: /toslang-ast/
categories: [TosLang]
tags: [Compiler Development]
excerpt: In which we learn what those three letters mean.
---

Hey everyone!

Last time, we opened up the black box that is a modern compiler to see that it what is fact three black boxes linked together.

{%
    include figure.html
    src="/assets/img/SimpleCompiler.png"
    caption="Those three again"
%}

We then delved into the frontend of the compiler to discover that (surprise!) once again we can divide it up into three black boxes. Finally, in the words of Samuel L. Jackson, having had enough of these motherf\*\*\*ing black boxes in this motherf\*\*\*ing compiler, we opened the first of them by building a lexer for TosLang.

{%
    include figure.html
    src="/assets/img/Frontend2.png"
    caption="That's one down"
%}

Now, logic would dictate that we jump right into the parser. However, since what we want to produce next (the mysterious AST) is quite a bit more complex than a simple sequence of tokens we first need to think about what we are going to build.

## How do we structure a (programming) language?

Let's think back to our days in primary school. One of things that you (hopefully) learned there, is how to construct sentences. As you should know, a traditional sentence is composed of a subject that will perform the action of a verb. Sometime you throw in a complement or an adverb just to have enough word to respect the minimum word count. Basically, we learned that a language is a set of words that we can freely mix as long as we follow some rules which are defined by what we call a grammar.

Well, it might be disappointing to some, but programming languages are exactly the same in that regard. They have some syntactic elements (tokens) that we can mix together as we want as long we follow the grammar of the language.

However, take note that merely stitching language constructs together so that they respect the grammar won't necessarily produce a valid program. For example, many languages that require a variable to be declared before it is used do not have this rule in their grammar. In a later post, I'll go over the techniques used by a compiler make sure that a program makes sense.

{%
    include figure.html
    src="/assets/img/grammar.jpg"
    caption="Grammatically correct"
%}

Moving along, a programming language grammatical elements can be divided into three categories (declaration, statement, expression) which I will proceed to describe.

## What is a declaration?

The easiest one first. A declaration simply introduces a new named element into a program. For example, this a variable declaration in TosLang:

```cpp
var MyInt : Int = 42; // Declaration of MyInt
```

## What is a statement?

A statement is a piece of a program that is to be executed. Note that 'piece of a program' doesn't necessarily means a single line in a program. Witness:

```cpp
while SomeVar < 10
{
  MyInt = MyIntVar + SomeVar;
  SomeVar = SomeVar - 1;
}
```

This here is a while statement. The interesting part is that this while statement also contains a compound statement (i.e. a list of statements within braces) that serves as its body.

## What is an expression?

An expression is fragment of a program that will produce a value. In other word, a statement that produces something. Some examples (focus on the right hand side):

```cpp
MyInt = 1;                // A literal number expression
MyInt = MyInt + 2;        // A binary expression between an identifier
                          // expression and a number expression
MyInt = SomeFunc(MyInt);  // A function call expression
```

## You still didn't explain what an AST is?

I see you were paying attention! The abstract syntax tree (so that's what AST means!) is a data structure that models the relations between grammatical elements found in a program written in a given language. The 'abstract' part comes from the fact that this tree doesn't model every single detail of the language syntax. For example, in our case, a 'if' statement will be represented solely by two nodes: one for the condition and one for the statements to execute when the condition is true (remember: no 'else' in TosLang for the moment).

Here's a concrete representation of what I'm trying to say. If you take this program here:

```cpp
fn main() -> Void
{
  print "Hello World!";
  return 0;
}
```

And create an AST out of it, you will get this:

{%
    include figure.html
    src="/assets/img/AST.bmp"
    caption="That's an AST!"
%}

## Let's start implementing

You could probably guess by my examples of the different grammatical categories that we would be creating quite a hierarchy of AST nodes. For our base class, we will have the following:

```cpp
class ASTNode;

using ChildNodes = std::vector<std::unique_ptr<ASTNode>>;

class ASTNode
{
public:
  enum class NodeKind
  {
    // Misc
    ERROR,

    // Declarations
    FUNCTION_DECL, PROGRAM_DECL, PARAM_VAR_DECL, VAR_DECL,

    // Expressions
    BINARY_EXPR, BOOLEAN_EXPR, CALL_EXPR, IDENTIFIER_EXPR,
    NUMBER_EXPR, STRING_EXPR,

    // Statements
    COMPOUND_STMT, IF_STMT, PRINT_STMT, RETURN_STMT,
    SCAN_STMT, WHILE_STMT,
  };

public:
  explicit ASTNode(NodeKind kind = NodeKind::ERROR)
      : mKind{ kind }, mName{ } { }
  virtual ~ASTNode() = default;

public:
  NodeKind GetKind() const { return mKind; }
  const std::string& GetName() const { return mName; }
  const ChildNodes& GetChildrenNodes() const { return mChildren; }

protected:
  void AddChildNode(std::unique_ptr<ASTNode>&& node)
  {
    mChildren.emplace_back(std::move(node));
  }

  void AddChildNodes(std::vector<std::unique_ptr<ASTNode>>&& nodes)
  {
    mChildren.insert(mChildren.end(),
                     std::make_move_iterator(args.begin()),
                     std::make_move_iterator(args.end()));
  }

private:
  NodeKind mKind;       /*!< Kind of the AST node */
  std::string mName;    /*!< Name of the AST node. */
  ChildNodes mChildren; /*!< List of children nodes linked
                             to this AST node */
};
```

As you can see, ```ASTNode``` has only the bare necessities. It has a way to identify what it is ( ```mKind``` ) that will allow us to forego the need for RTTI. It has a way to name it ( ```mName``` ) which will come in handy when we'll want to print an AST to a stream. Lastly, it has a way to access its child nodes ( ```mChildren``` ) if it has any. It also offers methods by which every derived node class will be able to add one or more child nodes to its collection. This will be done by taking the ownership of any child node. Indeed, we are going to take full advantage of ```std::unique_ptr``` to clearly express the relation between a parent node and a child node in our AST.

Just below ```ASTNode``` in the node hierarchy are the three grammatical categories expressed as classes.

```cpp
class Decl : public ASTNode
{
public:
  explicit Decl(NodeKind kind) : ASTNode{ kind } { }
  virtual ~Decl() = default;
};

class Expr : public ASTNode
{
public:
  explicit Expr(NodeKind kind) : ASTNode{ kind } { }
  virtual ~Expr() = default;
  // Quickly indicates if we're dealing with a literal or
  // any other kind of expression
  virtual bool IsLiteral() const { return false; }
};

class Stmt : public ASTNode
{
public:
  explicit Stmt(NodeKind kind) : ASTNode{ kind } { }
  virtual ~Stmt() = default;
};
```

With these three defined, all that's left is to implement a representation for every elements of the language.

## A quick example of a concrete AST node

Since there are quite a lot of concrete nodes to define, I'll only go over one of them to show you how it can be done. For more details, you can check out the TosLang compiler over [here](https://github.com/faouellet/Tostitos/tree/master/TosLang).

So, let's say you want to define what a function call should look like as an AST node.

```cpp
someFunc("hello", true, MyInt + 2);
```

A quick glance at the above code, reveal that we need two things to characterize a call. First and most obvious is the name of the function being called. Second is a sequence of values that we call the arguments. AS we saw, a piece of program producing a value is classified as an expression. While on the suject of expressions, it is important to realize that a function call is also an expression. It can produce a value that can later be use by another part of a program. Putting these observations into code, we are able to produce the following implementation of a function call node.

```cpp
class CallExpr : public Expr
{
public:
  CallExpr(const std::string& fnName,
           std::vector<std::unique_ptr<Expr>>&& args)
      : Expr{ NodeKind::CALL_EXPR }
  {
    mName = fnName;
    AddChildNodes(args);
  }

  virtual ~CallExpr() = default;
};
```

So there you have it.

Now, you might be wondering how we can instantiate and assemble nodes from a given program to create an AST. Well, you'll have to wait until next month to find out!

**Coming up next month**: A simple parser for a simple language
