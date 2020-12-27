---
title: Let's implement a language - Part 3 - The Parser
permalink: /toslang-parser/
categories: [TosLang]
tags: [Compiler Development]
excerpt: In our biggest installment yet, we finally build the AST that was promised.
---

Hey everyone!

Continuing our journey toward a fully working compiler, we will now build the parser that will allow us to produce an AST out of program we received as input.

{%
    include figure.html
    src="/assets/img/Frontend3.png"
    caption="Ever closer to being done with the frontend"
%}

However, before diving into the parser, some things must be taken care of.

## Refactoring the lexer

The lexer that we previously built is fine as a first draft that (mostly) follows common (and old!) language building design. However, such a design might not scale well when working with sizeable programs from (verbose) programming languages.

{%
    include figure.html
    src="/assets/img/verbose.jpg"
    caption="Definitely not thinking about a language in particular"
%}

The thing is that lexing such programs all at once will rapidly consume way too much memory. And this is only for producing tokens. Just imagine if we want to also want to produce informations about the source location of a token or the identifier associated with a token. All of this adds up to a hefty price in memory consumption.

The thing to realize about the lexer and the parser is that they're not two out-of-touch entities that only communicate when the lexer hands off a token sequence to the parser. In reality, they're two cooperating entities that work together to produce an AST out of the text of a program. This is why they're so often used as [a motivating example for what coroutines might be useful for](https://www.youtube.com/watch?v=_fu0gx-xseY).

Thus, in the real world, lexers more often than not offers services to fetch tokens from a program one at a time and not to get them all at the same time. For those of you that know a bit about Flex and Bison, this should feel pretty familiar since this is how the lexers and parsers they generate operate.

With all that said, let's modernize our lexer. First, let's go over the ```Lexer``` class definition.

```cpp
class Lexer
{
public:
  Lexer(const std::string& programStr)
    : mProgram{ programStr },
      mProgramIt{ mProgram.cbegin() }.
      mProgramEnd{ mProgram.cend() },
      mLoc{ "" } {};

  // Advances the lexer to the next token and returns it
  Token GetNextToken();

  // Some accessors
  int GetCurrentNumber() const { return mCurrentNumber; }
  const std::string& GetCurrentIdentifier() const
  {
    return mCurrentIdentifier;
  }

private:
  std::string mProgram;           /*!< Program text */
  std::string::const_iterator mProgramIt;  /*!< Program iterator */
  std::string::const_iterator mProgramEnd; /*!< Program end */

  int mCurrentNumber;             /*!< Current number */
  std::string mCurrentIdentifier; /*!< Current var/func identifier */

  std::locale mLoc;               /*!< The source locale */
};
```

Compared to the previous version of the ```Lexer``` class definition, this one contains way more data members. The first three ones are there to enable the ```Lexer``` to go over a program's text. The next two (```mCurrentNumber``` and ```mCurrentIdentifier```) will allow us to transmit some informations that the parser might require. For example, what is the name of the function in a call expression or what is the value of a number literal. This all ties into the fact that in the ```Lexer```-```Parser``` relationship, the ```Lexer``` is the one that has to deal directly with the program's textual representation while the ```Parser``` toils away building the AST. Finally, the last one (```mLoc```) is there because we need to take into account the program's locale when lexing it.

The other major difference from the previous definition, is the function for getting tokens out of a program. As you can see from the function signature, we ditched the token stream approach and now the function will only give out one token at a time. Its implementation stays mostly the same, only now we don't loop over all the program's text ine one go.

```cpp
Token Lexer::GetNextToken()
{
  // Skip over whitespaces
  while ((mProgramIt != mProgramEnd) && isspace(*mPogramIt, loc))
  {
    ++programIt;
  }

  // End of file, we're done
  if (mProgramIt == mProgramEnd)
  {
    return Token::TOK_EOF;
  }

  // Caching the current char because we will use it A LOT
  const char currentChar = *programIt;

  switch (currentChar)
  {
  case ',':
    ++mProgramIt;
    return Token::COMMA;

  // Lots of similar cases for simple one character tokens

  default:  
    // We have a token coming from a more complex pattern,
    // let's find out what

    // If this starts with a letter,
    // it is either a keyword or an identifier
    if (isalpha(currentChar, loc))
    {
      // Let's greedily build a string with the pattern
      mCurrentIdentifier = currentChar;
      // We accept alphanumeric characters because it is legal for a
      // function or variable identifier to contains number
      while ((++mProgramIt != mProgramEnd) && isalnum(*mProgramIt, loc))
      {
        mCurrentIdentifier += *mProgramIt;
      }

      if (mCurrentIdentifier == "fn")
      {
        return Token::FUNCTION;
      }
      // Other cases where the pattern directly match
      // a keyword token...
      else
      {
        // If this doesn't match a keyword,
        // then it is a simple identifier.
        // We'll be able to ask for it throught GetCurrentIdentifier
        return Token::IDENTIFIER;
      }
    }
    // It starts with a digit, then it can only be a number
    else if (isdigit(currentChar, loc))
    {
      std::string numberStr;

      // Eat up all the digits making up the number
      while (isdigit(*(++mProgramIt, loc)))
      {
        numberStr += *mProgramIt
      }

      // We'll be able to ask for it throught GetCurrentNumber
      mCurrentNumber = std::stoi(numberStr);
      return Token::NUMBER;
    }
    else
    {
      // The hell is that?
      return Token::UNKNOWN;  
    }
  }
}
```

With these few changes, we should be able to scale a bit better now. Also, this new version of the ```Lexer``` could easily accomodate a lookahead feature if TosLang's grammar ever evolves in such a way that it might need it to disambiguate some of its rules.

## The parser

For a post about the parser there sure is a lack of it so far! Let's fix this. As the interface to our ```Parser```, I suggest the following:

```cpp
using ExprPtr = std::unique_ptr<Expr>;

class Parser
{
  public:
    Parser() : mLexer{ }, mCurrentToken{ Lexer::Token::UNKNOWN } { }
    std::unique_ptr<ASTNode> ParseProgram(const std::string& filename);

  private:  // Declarations
    std::unique_ptr<ASTNode>      ParseProgramDecl();
    std::unique_ptr<FunctionDecl> ParseFunctionDecl();
    std::unique_ptr<VarDecl>      ParseVarDecl();

  private:  // Expressions
    ExprPtr ParseExpr();
    ExprPtr ParseArrayExpr();
    ExprPtr ParseBinaryOpExpr(Lexer::Token operationToken,
                              ExprPtr&& lhs);
    ExprPtr ParseCallExpr(ExprPtr&& fn, bool isSpawnedExpr);

  private:    // Statements
    std::unique_ptr<CompoundStmt> ParseCompoundStmt();
    std::unique_ptr<IfStmt>       ParseIfStmt();
    std::unique_ptr<PrintStmt>    ParsePrintStmt();
    std::unique_ptr<ReturnStmt>   ParseReturnStmt();
    std::unique_ptr<ScanStmt>     ParseScanStmt();
    std::unique_ptr<WhileStmt>    ParseWhileStmt();

  private:
    Lexer mLexer;               /*!< Lexer used by the parser
                                     to acquire tokens */
    Lexer::Token mCurrentToken; /*!< Current token being treated
                                     by the parser */
};
```

This implementation ties into what I previously wrote about the roles in the ```Lexer```-```Parser``` relationship. To wit, the ```mLexer``` member variable will be there to analyze the program's text and lazily supply the ```Parser``` with tokens. The ```mCurrentToken``` is there as a caching mechanism because the ```Parser``` will put it through quite the grinder when trying to match [TosLang's grammar rules](https://github.com/faouellet/Tostitos/blob/master/docs/TosLangGrammar.txt).

The proposed interface also make echo to [what I wrote last month](https://faouellet.github.io/toslang-ast/) about the relations between nodes in the AST. For those of you that don't remember, the AST that we're about to build uses ```std::unique_ptr``` to both ease memory management and make clear the intent that a parent node is the unique owner of its child nodes. Consequently, it's only natural that the ```Parser``` that is going to create this AST be made of functions producing ```std::unique_ptr``` of nodes that can then be moved into a parent node.

The last thing to mention about this interface is that the only service this class will offer to its clients is to produce an AST from a file. This is merely for educational purposes, I just want the ```Parser``` to be standalone for the moment. Now then, let's have a look at the implementation.

```cpp
std::unique_ptr<ASTNode>
    Parser::ParseProgram(const std::string& filename)
{
  // We're only dealing with TosLang file
  if (std::filesystem::path(filename).extension() != ".tos")
  {
    return nullptr;
  }

  // Can't go ahead with the parsing if the lexer isn't ready
  if (!mLexer.Init(filename))
  {
    return nullptr;
  }

  // Let's get it on!
  return ParseProgramDecl();
}
```

Basically, this method just make sure that everything is set up for the parsing and then jumps into ```ParseProgramDecl``` to start the actual parsing. Remember that following the TosLang's grammar, the root of any program is a node of the ```ProgramDecl``` type.

Before continuing, however, I realize some may be stuck on the line where is invoke a standard function to ask about a file extension. No, you're not dreaming. This is [a C++17 feature](http://en.cppreference.com/w/cpp/filesystem) that's coming to a standard library near you (if your preferred standard library implementation doesn't already offers it).

Now let's see how we parse a ```ProgramDecl```:

```cpp
std::unique_ptr<ASTNode> Parser::ParseProgramDecl()
{
  // ProgramDecl ::= Decls
  auto programNode = std::make_unique<ProgramDecl>();
  mCurrentToken = mLexer.GetNextToken();
  
  // Decls ::= Decl Decls
  //       ::= Decl
  std::unique_ptr<Decl> node = std::make_unique<Decl>();
  while (mCurrentToken != Lexer::Token::TOK_EOF)
  {
    // Decl ::= FnDecl
    //      ::= VarDecl
    switch (mCurrentToken)
    {
      case Lexer::Token::FUNCTION:
        node.reset(ParseFunctionDecl().release());
        break;
      case Lexer::Token::VAR:
        node.reset(ParseVarDecl().release());
        break;
    }

    programNode->AddProgramDecl(std::move(node));

    // Go to next declaration.
    // In case an error happens, this will skip straight
    // to the next declaration
    while ((mCurrentToken != Lexer::Token::VAR) &&
           (mCurrentToken != Lexer::Token::FUNCTION) &&
           (mCurrentToken != Lexer::Token::TOK_EOF))
    {
      mCurrentToken = mLexer.GetNextToken();  
    }
  }

  return programNode;
}
```

Here, in lieu of comments I chose to describe part of the function with grammar production rules. This was done to make clear that implementing a parser, in its most basic form, is just a translation from these rules to code. More often than not, a language's grammar rules can be divided into three categories for parsing. Let's examine them with examples from TosLang.

## 1. Rules producing a terminal node

These rules produce a terminal node like a number or a string literal. In that case, we can just parse them and produce the corresponding node without much hassle. As an example, here's how to parse a string literal of TosLang.

```cpp
// StringExpr = '"' identifier '"'
std::unique_ptr<StringExpr> Parser::ParseStringExpr()
{
  if(mCurrentToken == Lexer::Token::STRING_LITERAL)
  {
    return make_unique<StringExpr>(mLexer.GetCurrentIdentifier());
  }

  return nullptr;
}
```

It's so simple, the lexer took care of everything!

## 2. Rules producing an intermediate node

These rules concern elements of the language such as declarations (either of functions or variables) or non-terminal expressions like arithmetic expressions. These will require that we dig down into each element that make up the intermediate node to parse them before dealing with the intermediate node. This digging may further continue if one or more elements of the intermediate node are also intermediate nodes themselves. An example of such a rule in TosLang is the rule concerning variable declarations.

```cpp
// VarDecl ::= 'var' IdentifierExpr ':' TypeExpr ( '=' Expr )? ';'
std::unique_ptr<VarDecl> Parser::ParseVarDecl()
{
  std::unique_ptr<VarDecl> node = std::make_unique<VarDecl>();
  // Get the variable name
  if ((mCurrentToken = mLexer.GetNextToken())
        != Lexer::Token::IDENTIFIER)
  {
    return nullptr;
  }
  const std::string varName = mLexer.GetCurrentStr();

  // Make sure the variable name is followed by a colon
  if ((mCurrentToken = mLexer.GetNextToken()) != Lexer::Token::COLON)
  {
    return nullptr;
  }

  // Check that a type token is following the colon
  // in the variable declaration
  if ((mCurrentToken = mLexer.GetNextToken()) != Lexer::Token::TYPE)
  {
    return nullptr;
  }

  // Get the type of the variable
  Common::Type vType = GetTypeFromIdentifier(mCurrentToken);
  
  // Is the variable a scalar or an array?
  mCurrentToken = mLexer.GetNextToken();
  int varSize = 0;    // We define a scalar as having a size of zero.
                      // A zero-length array is thus illegal.
  if (mCurrentToken == Lexer::Token::LEFT_BRACKET)
  {
    if (!ParseArrayType(varSize, vType))
    {
      return nullptr;
    }
  }

  VarDecl* vDecl = new VarDecl(varName, vType, /*isFunctionParam=*/false, varSize);

  // The variable may be initialized when it is declared
  if (mCurrentToken == Lexer::Token::ASSIGN)
  {
    mCurrentToken = mLexer.GetNextToken();
    // Adding initialization expression if possible
    // If it's not, then we're in an error state we need to get out of
    if (!vDecl->AddInitialization(ParseExpr()))
    {
      delete vDecl;
      return nullptr;
    }
  }

  // We can only produce the VarDecl node
  // if the statement is properly terminated
  if (mCurrentToken == Lexer::Token::SEMI_COLON)
  {
    node.reset(vDecl);
    return node;
  }
  else
  {
    return nullptr;
  }
}
```

## 3. Rules with recursion

There exists rules within a grammar to specify that a sequence of node of the same type will be produced. These will take the form of recursive rules such as the one in ```ParseProgramDecl``` that specify that a ```ProgramDecl``` is made of several ```Decls```. Such rules can be translated into code by turning them into ```while``` loops. As an example, let's look at what happens when parsing the parameters of a function declaration

```cpp
// ParamDecl     ::= '(' ParamVarDecls ')'
// ParamVarDecls ::= ParamVarDecl ',' ParamVarDecls
// ParamVarDecl  ::= IdentifierExpr ':' Type
std::unique_ptr<ParamVarDecl> ParseParamVarDecl()
{
  // We must start at a left parenthesis
  if(mCurrentToken != Lexer::Token::LEFT_PAREN)
  {
    return nullptr;
  }

  auto params = std::make_unique<ParamVarDecl>();
  std::unique_ptr<VarDecl> param;
  std::string varName;

  mCurrentToken = mLexer.GetNextToken();
  // We'll parse each parameter one at a time through this loop
  while ((mCurrentToken != Lexer::Token::RIGHT_PAREN) &&
         (mCurrentToken != Lexer::Token::TOK_EOF))
  {
    // The parameter must have an identifier
    if (mCurrentToken != Lexer::Token::IDENTIFIER)
    {
      return nullptr;
    }
    varName = mLexer.GetCurrentStr();

    // The identifier must be followed by a colon
    if ((mCurrentToken = mLexer.GetNextToken()) != Lexer::Token::COLON)
    {
      return nullptr;
    }

    // After the colon there must be a type specification
    if ((mCurrentToken = mLexer.GetNextToken()) != Lexer::Token::TYPE)
    {
      return nullptr;
    }
    Common::Type vType = GetTypeFromIdentifier(mCurrentToken);

    // Is the variable a scalar or an array?
    mCurrentToken = mLexer.GetNextToken();
    int varSize = 0;    // We define a scalar as having a size of zero.
                        // A zero-length array is thus illegal.
    if (mCurrentToken == Lexer::Token::LEFT_BRACKET)
    {
      if (ParseArrayType(varSize, vType))
      {
        return fnNode;
      }
    }

    // We successfully parse a parameter
    param.reset(std::make_unique<VarDecl>(varName, vType,
                                          /*isFunctionParam=*/true,
                                          varSize).release());

    // Bailing out of the function if we're not at the end of the
    // parameters declaration or in-between two parameter declarations.
    if ((mCurrentToken != Lexer::Token::RIGHT_PAREN) &&
        (mCurrentToken != Lexer::Token::COMMA))
    {
      return nullptr;
    }

    // Everything's good, let's add another parameter
    params->AddParameter(std::move(param));

    // We may be done here. If that's the case, get out of the loop
    if (mCurrentToken == Lexer::Token::RIGHT_PAREN)
    {
      break;
    }

    // Let's go for another one
    mCurrentToken = mLexer.GetNextToken();
  }

  // We'll only return the parameters declaration if
  // we're at the expected end of the statement
  if (mCurrentToken != Lexer::Token::RIGHT_PAREN)
  {
    return nullptr;
  }  

  return params;
}
```

Following these guidelines, one is able to implement what is commonly referred to in formal parlance as a recursive descent parser. In other words, what we described is a parser that will start by generating terminal nodes to then assemble them together to produce the AST.

Finally, take note that the implementation presented doesn't handle errors in any way. This is a topic for another post.

For more details on the implementation of TosLang's ```Parser```, you can go check the full source code [here](https://github.com/faouellet/Tostitos/tree/master/TosLang/Parse).

**Coming up next month**: Starting semantic analysis with something quite curious...
