---
title: CS Games 2015 Parallelism Challenge - A More Efficient Solution
permalink: /csgames-efficient/
categories: [CS Games 2015 Parallelism Challenge]
tags: [Parallelism]
---

![Desktop View]({{ "/assets/img/csgames-logo.png" }})

What's up people!

Last time, we went over some potential solutions for a problem given during the parallelism challenge. As you may remember (if you don't, go [here](http://faouellet.github.io/csgames-simple/)), I put a special emphasis on not trying to go overboard with threads. My reasoning was that trying to start as many threads as possible right off the bat would be counter-productive because it could yield quite the complex solution. Some might disagree and find the solution using inner parallelism to be most elegant one. But they would be wrong. Also, consider the following:

|Method|Time (ms)|
|:---|---:|
|Sequential|141.84|
|Inner parallelism|94.44|
|_**Outer parallelism**_|_**90.72**_|

What's this you ask? Well, it's just a small experiment I ran on my machine. Basically, I used all three proposed solutions to find the last element in a vector containing 100,000,000 elements (remember we're kind of in a real time setting here, so the worst case matters a lot). This is repeated 25 times to give a small sample of what was expected to happen when we evaluated the students' solution. As you can see, I wasn't kidding when I told you that you should try to avoid creating and deleting too many threads.

{%
    include figure.html
    src="/assets/img/drugs.jpg"
    caption="Threads are like drugs: use them in the right amounts. Wait, that doesn't sound right..."
%}

This then begs the question: are we still dealing with too many threads? The answer is, without any surprises since I'm asking, yes. The truly best way to reduce the threads created and destroyed is by using a thread pool i.e. a data structure that will manage a number a threads for as long as you have tasks to give it. Bonus, it also impress some challenge's organizers resulting in a better score because of a WOW factor. In fact, it was what I personally hoped that at least one of the teams would produce during the parallelism challenge. Alas, it didn't happen. Still, I'll present what I expected of the competitors.

Before we go any further, I want to say that what I'm going to present is largely inspired by something that my friend (and programming guru) Patrice Roy already did. If you can understand French, go have a look at his [website](http://h-deb.clg.qc.ca/), it's really worth it.

So without further ado, let's implement a thread pool.

The first thing a thread pool needs is threads. Thus we will start with the following:

```cpp
class ThreadPool
{
public:
  ThreadPool(size_t nbThreads = std::thread::hardware_concurrency())
  {
    mThreads.reserve(nbThreads);
  }

private:
  ThreadPool(const ThreadPool&) = delete;
  ThreadPool& operator=(const ThreadPool&) = delete;

private:
  std::vector<std::thread> mWorkers;
};
```

This first pass defines the very basis on which a thread pool can be built. By default, we will ask for as many threads as there are cores on the machine it will run through a call to [```std::thread::hardware_concurrency```](http://en.cppreference.com/w/cpp/thread/thread/hardware_concurrency). This is appropriate for us since our threads will not do any kind of I/O. If we were to use the ```ThreadPool``` for more I/O-intensive purposes, a higher number of threads would be preferable to compensate for threads busy doing things other than computing stuff. Also, the copy constructor and the assignment operator have been deleted since copying a thread pool is not a really good idea.

Next, we have to give those worker threads something to do. But first, we need to decide what IS a task. Remember that we have to deal with six different kinds of problem that have different types of inputs. A natural choice would be to involve templates and a [```std::function```](http://en.cppreference.com/w/cpp/utility/functional/function), or, more appropriatedly in this context, a [```std::packaged_task```](http://en.cppreference.com/w/cpp/thread/packaged_task). But wait! If we do this we would then need to apply this kind of change to our thread pool.

```cpp
template <typename R, typename... Args> // Humm... something doesn't
                                        // seem right around here
class ThreadPool
{
public:
  ThreadPool(size_t nbThreads = std::thread::hardware_concurrency())
  {
    mThreads.reserve(nbThreads);
  }

private:
  ThreadPool(const ThreadPool&) = delete;
  ThreadPool& operator=(const ThreadPool&) = delete;

private:
  std::vector<std::thread> mWorkers;
  // Probably because of the line below
  std::deque<std::packaged_task<R(Args...)>> mTasks;
};
```

By trying to add generic tasks to the thread pool, we forced the thread pool to be generic on the basis of the return type and the arguments' types of its tasks. This will result in having to instantiate a ```ThreadPool``` for each type of task we want to handle. In other words, we lost genericity by trying to use generic types! Ponder that one for a while.

To fix this situation, we can use a simple C++ trick: a non-generic base class. The idea here is to hide the fact that we store different kinds of generic tasks in our task queue by hiding them behind pointers to their non-generic interface. This will eliminate the need to have template at the ```ThreadPool``` level since there won't be generic member variables inside it anymore. Having said that here's our new task abstraction.

```cpp
class ThreadPool
{
private:
  class ITask
  {
  public:
    virtual ~ITask() = default;
    virtual void Run() = 0;
  };

  template <typename R>
  class Task
  {
  public:
    virtual ~Task() = default;
    void Run() override { mTask(); }
    std::future<R> GetFuture() { return mTask.get_future(); }

  private:
    std::packaged_task<R()> mTask;
  };

/* ... */

private:
  std::vector<std::thread> mWorkers;
  std::deque<std::unique_ptr<ITask>> mTasks;
};
```

Now, there are many things to explain about the task implementation. First off, notice how I've gotten rid of the ```Args...``` types. The reason why I did this is because they quite frankly weren't necessary for having tasks that can take any kinds of arguments. I'll show you how in a moment (spoilers: it involves lambda capture). Another thing to notice is the ```GetFuture``` method that I added. The reason I added it is because I want the client code to be able to get results back from the ```ThreadPool```. The method by which I do so is by giving the client a [```std::future```](http://en.cppreference.com/w/cpp/thread/future) that it can use to get the desired result when it needs to. The last thing to see here is that I've chosen to use a [```std::deque```](http://en.cppreference.com/w/cpp/container/deque) of [```std::unique_ptr```](http://en.cppreference.com/w/cpp/memory/unique_ptr) to be my task container. The reason for using a [```std::deque```](http://en.cppreference.com/w/cpp/container/deque) is because we want a FIFO container to ensure that every task will be executed while the reason for the use [```std::unique_ptr```](http://en.cppreference.com/w/cpp/memory/unique_ptr) is, as my prime minister would say, because it's 2016.

Now that we know what a task is, it's time to give some to our threads. To do so, we'll need to add some member variables to our thread pool to ensure that no race conditions occurs between threads manipulating the task queue.

```cpp
private:
  std::vector<std::thread> mWorkerThreads;
  std::deque<std::unique_ptr<ITask>> mTasks;  

  mutable std::mutex mTaskMutex;
  std::condition_variable mNewTasksCV;
  std::mutex mNewTaskMutex;

  std::atomic<bool> mStop;
```

The first thing we need to coordinate the accesses to the task queue is a way to ensure that only one thread at a time try to take away a task from it. This is the job of ```mTaskMutex```. The second thing we need is a way to alert threads that a new task to run is available. This is done through the ```mNewTaskCV``` condition variable. For those who don't know what a condition variable is a data structure used to synchronize threads waiting for some condition, hence the name, to be met. Notice how there is a second mutex associated with the condition variable. This is because a condition variable needs to changed under a mutex to correctly sends its notification(s) to waiting threads. Finally, we'll use an atomic boolean to communicate to the workers that they should stop working. This will possibly happen in either the destructor (which we'll get to) or when the client code explicitly request it through this method:

```cpp
void Stop()
{
  mStop = true;
  // Taking the mutex managing new task arrival so that our
  // message will be heard by all
  std::unique_lock<std::mutex> lock(mNewTaskMutex);
  // Notifying every thread waiting on new task that there
  // won't be any
  mNewTasksCV.notify_all();
}
```

Now that we have everything that we need to avoid race conditions within our ```ThreadPool```, we can proceed with adding task to it. The method by which this will be achieved looks like this:

```cpp
template <class F>
std::future<std::result_of_t<F()>> AddTask(F&& func)
{
  std::lock(mTaskMutex, mNewTaskMutex);
  std::unique_lock<std::mutex>(mNewTaskMutex, std::adopt_lock);
  std::lock_guard<std::mutex>(mTaskMutex, std::adopt_lock);

  auto task = std::make_unique<ConcreteTask<std::result_of_t<F()>>>([func]() { return func(); });

  auto fut = task->GetFuture();

  mTasks.emplace_back(std::move(task));

  // Notifying the worker threads that a new task is available
  mNewTasksCV.notify_one();
  return fut;
}
```

Let's go over it line by line. In the declaration, despite the template wizardry at work, we simply announce that we will return a future whose type corresponds to the type of the value returned by executing a callable of type ```F```. Then, inside the function, we take hold of both the mutexes used to manage the task queue. This is done to both avoid data races and later notify some worker threads that a new task is avalaible. To lock the mutexes using [```std::lock```](http://en.cppreference.com/w/cpp/thread/lock) because this function is written with a deadlock avoiding algorithm saving us lots of trouble. The mutexes are then given to adoption to a [```std::unique_lock```](http://en.cppreference.com/w/cpp/thread/unique_lock), because of how [```std::condition_variable```](http://en.cppreference.com/w/cpp/thread/condition_variable) works, and a [```std::lock_guard```](http://en.cppreference.com/w/cpp/thread/lock_guard), because we want to release the mutex when we're done. Afterwards, in rapid succession, the task is created, we get a future from it before moving it in the task queue, we give a shout to a thread and we're done.

After adding a task, the most important thing to do is taking a task from the queue to execute it. To this end, I propose the following method:

```cpp
std::unique_ptr<ITask> GetNextTask()
{
  if (Empty())
  {
    // No task in the queue. Waiting until the situation changes
    std::unique_lock<std::mutex> lock(mNewTaskMutex);
    mNewTasksCV.wait(lock, [this]() { return !mTasks.empty(); });
  }

  if (mTasks.empty())
    return nullptr;

  std::lock_guard<std::mutex> lg{ mTaskMutex };
  // Get a task from the queue to feed the workers
  auto next = std::move(mTasks.front());
  mTasks.pop_front();
  return next;
}
```

Looking at it, we see that we first have to check if the queue is empty. If it is then we use the condition variable to wait until we get a signal that a new task was added to the queue. When we can proceed forward, we first make sure that the signal we received was sent because, if that's the case, the mutex protecting the task queue is taken to ensure that no other worker thread steal the task we are about to take. Afterwards, we can just pop a task out of the queue to give to a thread to execute.

The only things left to complete the thread pool is to actually start it and to stop it. The way we'll start it is by having it start its worker threads right from the moment it is created. This gives us the following constructor:

```cpp
ThreadPool(size_t nbThreads = std::thread::hardware_concurrency())
    : mStop { false }
{
  mThreads.reserve(nbThreads);
  for (size_t i = 0; i < mWorkerThreads.capacity(); ++i)
  {
    mWorkerThreads.emplace_back([this]()
    {
      while (!mStop)
      {
        auto task = GetNextTask();
        task->Execute();
      }
    });
  }
}
```

Quite simply, we just start some number of threads that will constantly ask for a new that to be run. This will go on until a stop signal is given to the ```ThreadPool```.

```cpp
~ThreadPool()
{
  Stop();
  for (auto& t : mWorkerThreads)
    if (t.joinable())
      t.join();
}
```

To stop the ```ThreadPool``` activities we'll make sure that a stop signal is sent before anything. Then, all that's left to do is to join all of its workers.

How do we use it? Let's have a look:

```cpp
std::array<bool, 4>
  SolveArrayProblemsPool(const std::array<std::vector<int>, 4>& arrays,
                         const std::array<int, 4>& values)
{
  std::array<bool, 4> answers = { false, false, false, false };
  std::array<std::future<bool>, 4> futureAnswers;

  for (int iProb = 0; iProb < 4; ++iProb)
  {
    futureAnswers[iProb] = pool.AddTask([&arrays, &vals, iProb]()
    {
      return std::find(arrays[iProb].begin(),
                       arrays[iProb].end(),
                       values[iProb]) != arrays[iProb].end();
    });
  }

  transform(std::begin(futureAnswers),
            std::end(futureAnswers),
            std::begin(answers),
            [](std::future<bool>& __f__) { return f.get(); });

  return answers;
}
```

Again, we parallelize our application by having different threads solve differents problem as was the case in the outer parallelism approach demonstrated last month.

And now, for the question we're all waiting for: how does it compare to our previous solution? Let's have a look:

|Method|Time (ms)|
|:---|---:|
|Sequential|141.84|
|Inner parallelism|94.44|
|Outer parallelism|90.72|
|_**Thread Pool**_|_**91.96**_|

Damn! All that effort for nothing!

{%
    include figure.html
    src="/assets/img/trying.jpg"
    caption="Once again, Homer Simpson was proven right."
%}

While this seems like a lot of wasted efforts, look closely at the results. The ```ThreadPool```-using solution is more efficient than two other proposed approachs. Moreover, it allows for more optimizations than the outer parallelism approach. Indeed, one could, for example, chop the problems to solve it in smaller pieces using more lightweight tasks while at the same time cranking up the number of threads avalaible to the ```ThreadPool```. This wouldn't be possible with the outer parallelism approach because we would then have to face the same issues that plagued the inner parallelism approach. All in all, I'd say that this approach still has a lot a potential but I'll let you the readers decide what to think of it.

Finally, to wrap up this article and this series, to all that will take part in this year's CS Games, go have fun!

**Coming up next month**: Let's change subjects and create a programming language!
