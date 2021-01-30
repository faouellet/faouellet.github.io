---
title: Of source control and databases - Annex B - Error management
permalink: /dvcs-error/
tags: [Source Control, Database]
categories: [Of source control and databases]
excerpt: What to do when everything goes wrong?
---

---

All the code for the articles in this series can be found [here](https://github.com/faouellet/DVCSUS).

---

To cap off this series, I wanted to talk about how I approached error management in DVCSUS. I already went into details for some parts (for instance, how I decided to deal with streaming errors). However, I didn't offer a global reasoning to explain why I went the way I did.

## Exceptions

### Filesystem interactions

If you look closely at the ```std::filesystem``` functions used throughout this series, you'll noticed that every time I use one of them, they're enclosed in a ```try...catch``` block. This is important because some, but not all, ```std::filesystem``` functions have a dual API where one version throws an exception on error and the other version sets an ```std::error_code``` output argument. As much as possible, I wanted to have one, and only one, error management strategy when dealing with these functions. The fact is that a dual API is also a double-edged sword. On the one hand, you can service different needs. On the other hand, it places a burden on the user to not incorrectly mix and match different versions of some functions of such an API. If no special care is taken or if no guidelines are enforced, this could lead to inelegant code (to put it midly) or, worse, missing out on errors. The last one is especially dangerous since those overlooked errors might bite our asses hard later on.

### noexcept

A question you might ask at this point is: why catch all the exceptions instead of simply rethrowing them? This is indeed a path that we could've taken. In such a scenario the onus of error handling would've fallen onto the users of the library implementing DVCSUS's functionalities. I find that this is an awful lot to drop on their laps. Furthermore, we have to ask ourselves: faced with an exception, can they take any measures to properly recover? The fact is that they can't. DVCSUS takes care of its own internal state and, more importantly, of the state of a repository on disk. Nothing will, nor should, leak through when things go spectacularly wrong.

I'll also briefly mention that there are C++ codebases that doesn't allow exceptions. Having exceptions cross the API boundary would automatically blacklist the DVCSUS library from being used in such codebase. It might not be a concern for a prototype, but it's something to keep in mind if you ever want to develop a C++ library that you want others to use.

With both these things in mind, it becomes clear that it's preferable that we catch and deal with any exception coming from a ```std::filesystem``` function.

This is where the ```noexcept``` keyword becomes useful. It clearly announces to both the users and the compiler that a given function won't throw under any circumstances. The user would then be reassured that major errors are dealt with within the library. As for the compiler, the presence of the ```noexcept``` keyword will have him help us spot any unwanted exception leakage by emitting compiler errors or warnings.

## Return types

The question as to what is the best way to propagate errors through a callstack is truly one for the ages. [Some](https://docs.oracle.com/javase/tutorial/essential/exceptions/index.html) [languages](https://doc.rust-lang.org/book/ch09-00-error-handling.html) are quite opiniated on the subject. Others, such as C++, not so much. In such cases, a library author has the freedom to choose the method by which to present an error to the caller. These methods can range from [simple error codes](https://docs.microsoft.com/en-us/windows/win32/debug/system-error-codes--0-499-) to [richer types](https://www.boost.org/doc/libs/1_75_0/libs/outcome/doc/html/index.html). In the end, which one to use depends on your use case.

For DVCSUS, as mentionned earlier, there's no action that a user can take when receiving an error notice. The tools offered by the API are stricly for interacting with a repository. Anything else is handled internally. This reduces the risk that a user makes a bad situation even worse. Still, the client needs to know when something went wrong. This is why we return a boolean value indicating whether or not everything went as expected.

### Error Information

While I still consider that returing booleans was a good choice for a prototype, I do recognize that it can be less than ideal in the long run. For one, booleans don't transmit much information to the caller. This is why I resorted to print errors as soon as they were found. It's better than nothing! 

Yet, it forbids any client to format the error information in a language that a client might understand better. Furthermore, it prevents users of our API to produce messages that might give hints to a user as to why an operation failed and what he can do to fix the problem. If DVCSUS was to be developped further, I'd definitively use something other than booleans to convey errors.

### [[nodiscard]]

Regardless of the type I'd use, the decision to qualify the return value of most functions as [```[[nodiscard]]```](https://en.cppreference.com/w/cpp/language/attributes/nodiscard) is one that I stand by. For those who've never seen this, [```[[nodiscard]]```](https://en.cppreference.com/w/cpp/language/attributes/nodiscard) is an attribute that'll encourage the compiler to issue a warning if the return value of a function isn't used at the call site. This forces a caller to, at least, acknowledge the return value of a function. Yes, he can just discard it right after assigning it to temporary variable. Still, it'll give him pause and perhaps make him think twice about what to do with the function's return value.

## Transactions

### Strong Exception Safety

One of the major advantages of using databases was that they made it really easy to guarantee at least [strong exception safety](https://en.wikipedia.org/wiki/Exception_safety) for DVCSUS' commands. In fact, most operations where reduced to only a bit of string formatting to produce a query which was shipped off to the databases for execution. This made it so no internal state was needed for any command. Then, since every query were framed in a transaction, if anything went wrong, the query was rollbacked, leaving things as they were. If it's at all possible, I encourage you to use transactional semantics in your code since it tremendously reduces the possibility of corruption both in memory and on disk.

It's worth mentionning however that there's one function in DVCSUS that breaks the strong exception safety: ```Init```. The problem here is that if the initialization of the repository's databases failed, the ```.dvcs``` will still be left on disk. This would prevent any further attempt at inializing a repository until the folder would be removed. To fix this problem, we could look at any of these options:

* Use transactional file system operations if available
* Look not only for a ```.dvcs``` folder but also for databases when validating that a repository has not already been initialized
* Offer an option to overwrite a repository when initializing

### Time Of Check To Time Of Use

On the subject of transactions, we also have to mention the ```Commit``` function. While all modifications this function does to the repository are correctly enclosed in a single transaction, the way it queries the ```Staging``` database could be considered incorrect. The problem, if there's one, is that the fetching of the current commit's hash is done in a transaction which is separate from the main commit transaction. This could lead to some problems(of the [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) nature) if DVCSUS was to be used in a multithreaded environment.

Fixing this problem is far from trivial. One way of doing it would involve integrating the SHA1 computation of the to-be-created commit into the main commit transaction. This could be done using [sha1_query provided by SQLite](https://www.sqlite.org/src/file/ext/misc/sha1.c) which would be given a blob made of ```Commit```'s arguments that would be combined with the result of querying for the current commit's hash.

A similar problem also presents itself in our handling of branches. In those cases, we have to make sure that a branch exists (or not depending on what we want to do) before going ahead with our operation.

While an interesting problem to think about (what happens when multiple users try to access a source control system concurrently), we have to admit that this is far from a normal use case. We would be better served by asking the user what he was trying to do. From then on, we could guide him through a better use of our system.
