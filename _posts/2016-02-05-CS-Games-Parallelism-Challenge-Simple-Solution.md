---
title: CS Games 2015 Parallelism Challenge - A Simple Solution
permalink: /csgames-simple/
categories: [CS Games 2015 Parallelism Challenge]
tags: [Parallelism]
---

![Desktop View]({{ "/assets/img/csgames-logo.png" }})

What's up people!

Now that a month has passed, surely you have found a, at least simple, solution to the challenge? Now, as I promised last month, let's go over a simple solution for this challenge...

...or not!

See, after pondering the thought for a while, I've decided against publishing a full solution to the challenge. I think people should be able to come up with their own without any shortcuts. So instead of giving away a solution, I'll be giving off some tips on how you can make the most of this challenge without losing your mind.

## Tip 0: Rage quit

{%
    include figure.html
    src="/assets/img/ragequit.jpg"
    caption="A legitimate strategy"
%}

Winning a war doesn't necessarily means winning every single battle. For some teams, it's better to focus on other challenges or just get some rest before the next day's challenges (this challenge was given quite late in the evening, when many were possibly burned out).

Also, the basic client given to the competitors already send random answers back to the server. So, who knows, you might get lucky (but you probably won't).

With that being said, let's go over some real tips for the ones ~~crazy~~ brave enough to step up to the challenge.

## Tip 1: Get some data

Quick question: can you solve a problem without any data?

<div style="color:Black; background-color:Black; display:inline-block">
   Protip: You can't
</div>

So with this in mind, the first thing you should do is capture some problems of each type that the server can send. With this you'll able to easily verify that your solutions are, in fact, correct.

## Tip 2: Solve the problem sequentially first

Given that this was a parallelism challenge, some might be tempted to go all out with a parallel algorithm right from the start. If this is your case, please calm yourself. It is far easier to solve a problem sequentially. Remember, you not only have to produce answers to the problems sent fast, but these answers also have to be correct.

To illustrate this, let's take the problem of finding a value in an array. If we choose to implement a parallel solution first, we might end up with something like this.

```cpp
template <size_t N>
std::array<bool, N>
  SolveArrayProblems(const std::array<std::vector<int>, N>& arrays,
                   const std::array<int, N>& values)
{
  std::array<bool, N> answers = { };
  const int nbThreads = std::thread::hardware_concurrency();
  std::thread> workers[nbThreads];

  // For each problem
  for (size_t iProb = 0; iProb < N; ++iProb)
  {
    auto begin = arrays[iProb].begin();
    size_t nbElemPerRange = arrays[iProb].size() / nbThreads;
    std::vector<int> partialAnswers(nbThreads, 0);

    // We'll start as many threads as possible
    for (int iThread = 0; iThread < nbThreads - 1; ++iThread)
    {
      // Each thread will have an equal part of
      // the array to scan through...
      workers[iThread] = std::thread{ [&, iProb, iThread]()
      {
        partialAnswers[iThread] =
            std::count(begin + (nbElemPerRange * iThread),
                       begin + (nbElemPerRange * (iThread + 1)),
                       values[iProb]);
      } };
    }

    // ...except for the last one that will take the remainder
    workers[nbThreads - 1] = std::thread{ [&, iProb]()
    {
      partialAnswers[nbThreads - 1] =
          std::count(begin + (nbElemPerRange * (nbThreads - 1)),
                     arrays[iProb].end(),
                     values[iProb]);
    } };

    // Make sure all threads are done
    for (auto& w : workers)
    {
      w.join();
    }

    // Answer true if the value was found at least once
    answers[iProb] =
         std::accumulate(partialAnswers.begin(),
                         partialAnswers.end(), 0) != 0;
  }

  return answers;
}
```

In short, we want to start as many threads as possible. To do so, we split an array so as to give each thread an (approximatively) equal share of the work to be done. Then, we start the threads by giving each of them a piece of the array. Then, we join them. Then, we give an answer based on the what the threads found. Finally, we start it all over for every other array that was part of the problem.

Phew!

It'll work, but this is seriously not a solution that you should consider during the competition for two big reasons. First, its size makes it error prone (especially in a high stress environment like the CS Games). So you'll waste a lot of time getting it right. Second, creating and destroying threads, which this solution does a lot of, takes time, which you don't really have in this context.

Instead, you should concentrate on first solving the problem in a sequential fashion. For example, a simple sequential solution can be written in way less lines of code using the [```std::find```](http://en.cppreference.com/w/cpp/algorithm/find) algorithm.

```cpp
template <size_t N>
std::array<bool, N>  
  SolveArrayProblemsSeq(const std::array<std::vector<int>, N>& arrays,
                        const std::array<int, N>& values)
{
  std::array<bool, N> answers = { };
  for (size_t iProb = 0; iProb < N; ++iProb)
  {
    answers[iProb] =
        std::find(arrays[iProb].begin(),
                  arrays[iProb].end(),
                  values[iProb]) != arrays[iProb].end();
  }

    return answers;
}
```

And not only is it shorter to write, it is easier to see that it is correct. So there you have it, a correct solution that you can then speed up with multiple threads.

## Tip 3: Parallelize at the coarsest level possible

For the few that got the idea to write a sequential solution first, it seemed that the tentation to write a solution akin to the first one shown in the tip above was too great to resist. If this is your case, calm your tits.

As I've already mentionned, there is a time penalty associated with creating and destroying threads. So when you have to parallelize a time-constrained application, be sure to keep this overhead in mind. Especially when some challenge's authors might reduce the time available for computation when evaluating the competitors' solutions (tough they would be real assholes to do it).

One way to minimize the number of threads used is to parallelize at the coarsest level possible like in the code below.

```cpp
template <size_t N>
std::array<bool, N>
  SolveArrayProblemsPar(const std::array<std::vector<int>, N>& arrays,
                      const std::array<int, N>& values)
{
  std::array<bool, N> answers = { };
  std::thread workers[N]; // Start only as many threads
                          // as there is problems to solve

  for (int iProb = 0; iProb < N; ++iProb)
  {
    workers[iProb] = std::thread{ [&, iProb]()
    {
      answers[iProb] =
          std::find(arrays[iProb].begin(),
                    arrays[iProb].end(),
                    values[iProb]) != arrays[iProb].end();
    } };
  }

  for (auto& w : workers)
  {
    w.join();
  }

  return answers;
}
```

As you can see, changing the granularity of the threads made us use 4 times less threads than in the previous parallel solution. It also gave us a easier to reason about solution which could probably be programmed in less than half the time it'd likely take to program the other parallel solution.

**tl;dr**: Be smart when designing a multithreaded application.

**Coming up next month**: An even more efficient way to crush the competition.
