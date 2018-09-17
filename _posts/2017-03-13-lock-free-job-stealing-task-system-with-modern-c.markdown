---
title: Lock-free job stealing with modern c++
date: 2017-03-13T15:58:13+01:00
tags: [c++,datatypes,lock-free]
---

In [my previous post](http://manu343726.github.io/2017/02/11/writing-ast-matchers-for-libclang.html) I (re)introduced you to my main personal project,
[siplasplas](https://github.com/Manu343726/siplasplas), a library and tool
to implement static and dynamic reflection with C++14. *Also runtime C++
compilation on top of it, I'm a bit masochist...*

As I told in the intro of my previous post, after [Meeting C++
conference](https://github.com/Manu343726/meetingcpp2016) my main work has
been to port the reflection parser from Python to a reusable C++ API.

One of the core aspects of the API is to be able to parse all the
dependencies of header files, if changed. Basically:

 1. Parse all the header files that have changed.

 2. Get a list of all the files included by each header (This is really
    simple [with libclang](http://clang.llvm.org/doxygen/group__CINDEX__MISC.html#ga4363bd8c203ca2b5dfc23c5765695d60)).

 3. Repeat from 1 recursively for each included header.

*Of course there are some extra details involved, like checking if headers
have been processed by this parsing job itself (Maybe as a dependency of
another header), ignoring system and out-of-sourcetree includes in
general, etc.*

I want to parallelize the parsing process as much as possible, since it
may involve parsing (And, later, metadata processing and code generation)
of tens to hundreds of files depending on the target project.  
Despite the usual advises like *"Never write your own task engine"*,
a task engine is something I didn't have written myself before, so
I thought this will be a great time to read [this awesome job stealing
series
again](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)

***Disclaimer:*** *This post is highly based on the Job System 2.0 series
on Molecular Matters blog, you may notice a lot of similarities between
this post snippets and explanations and the original molecular posts. All
the code shown here is real working code, the result from my work
following that great post series.*

## Basics

Job stealing is a scheduling strategy where instead of having a common
work pool for n workers, we have **per-worker pools**, and workers have the
chance to **steal work from others pool** if there is no stuff left in their
own pool. *It's basically like me eating pasta with the family.*

From the programming perspective, this translates to having n worker
threads with one m jobs pool each. By default each worker picks work from
its own pool, but if there's no work left they try to find more work by
randomly choosing another working and getting work from its pool. This
strategy works very well since it reduces pool access contention and
also helps to auto-balance work by aiding busy threads to complete their
assigned work.

## Jobs

But, what is a job? *I've used the word "job" at least 7 times and we
still don't know what the f... a job is or how it is modeled.*

A job is basically a unit of work, in its very basic form a plain free
function that has to be run in order to complete a task. To keep things
simple and fast, a job can be represented by a POD type as follows:

``` cpp
class Job
{
public:
    Job() = default;
    Job(void(*jobFunction)(Job&), Job* parent = nullptr);

    void run();
    bool finished() const;

private:
    void(*_jobFunction)(Job&);
    Job* _parent;
    std::atomic_size_t _unfinishedJobs;

    void finish();
};
```

Here, `Job` objects encapsulate a function pointer, a pointer to a parent
job, and an atomic counter. The pointer points to the function to execute
when the job is processed, and the parent pointer and the counter keep
track of parent-child relationships of jobs.

### Running jobs

A job is run by calling `Job::run()`, which is defined as follows:

``` cpp
void Job::run()
{
    _jobFunction(*this);
    finish();
}
```

First the job function is called with the job itself as parameter, and
then the job executes its "teardown" code, by calling `Job::finish()`:

``` cpp
void Job::finish()
{
    _unfinishedJobs--;

    if(finished())
    {
        if(_parent != nullptr)
        {
            _parent->finish();
        }
    }
}
```

In the description of the `Job` class we said that `_unfinishedJobs` and
the parent pointer keep track of parent-child relationships. What this
means is that a given job is considered finished if and only if its job
function has been run (its `Job::run()` method has been called by the
scheduler) **and if all its children jobs are finished**.  During job
initialization, the unfinished jobs counter of the parent (if any) is
incremented, and decremented when one of its children finishes.

``` cpp
Job::Job(void(*jobFunction)(Job&), Job* parent) :
    _jobFunction{jobFunction},
    _parent{parent},
    _unfinishedJobs{1} // 1 means **this** job has not been run
{
    if(_parent != nullptr)
    {
        _parent->_unfinishedJobs++;
    } } ```
}
```

*Note I don't care about memory ordering in the `unFInishedJobs` counter,
because there are no memory operations areound related to the counter.
This could be a use case for
[`std::memory_order_relaxed`](http://en.cppreference.com/w/cpp/atomic/memory_order)*.

This relationship makes possible to write fork-join parallelism patterns
by simply spawning multiple isolated jobs linked to a common parent job,
and make the current thread wait until the parent is finished. Something
like:

``` cpp
Job* root = allocateJob([](Job&){});

for(std::size_t = 0; i < 42; ++i)
{
    Job* job = allocateJob([](Job& job)
    {
        // do stuff
    }, root);

    submit(job);
}

submit(root);
wait(root); // join here
```

Finally, `Job::finished()` is defined as not having any unfinished job
left, i.e.:

``` cpp
bool Job::finished() const
{
    return _unfinishedJobs == 0;
}
```

### Job pools

We can allocate jobs by simply calling `new` and `delete`, but that would
not be an specially "fast" implementation, we can improve things a bit.
One alternative could be to use a pool allocator of jobs, but a simpler
(but limited) and faster alternative could be to have a preallocated array
of jobs, with an increasing counter of the number of jobs that have been
allocated. In other words, a linear allocator:

``` cpp
class Pool
{
public:
    Pool(std::size_t maxJobs);

    Job* allocate()
    bool full() const;
    void clear();

    Job* createJob(JobFunction jobFunction);
    Job* createJobAsChild(JobFunction jobFunction, Job* parent);

    template<typename Data>
    Job* createJob(JobFunction jobFunction, const Data& data);
    template<typename Data>
    Job* createJobAsChild(JobFunction jobFunction, const Data& data, Job*
    parent);
    template<typename Function>
    Job* createClosureJob(Function function);
    template<typename Function>
    Job* createClosureJobAsChild(Function function, Job* parent);

private:
    std::size_t _allocatedJobs;
    std::vector<Job> _storage;
};
```

The interface is very simple: One function to allocate a new job, a getter
to check if the pool is full (All the storage has been used), and
a function to clear the pool.  

First we pre-allocate the storage:

``` cpp
Pool::Pool(std::size_t maxJobs) :
    _allocatedJobs{0},
    _storage{maxJobs}
{
    SIPLASPLAS_ASSERT_EQ(_storage.size(), maxJobs);
}
```

To allocate a job, we simply return the address of the next available
storage element, if any:

``` cpp
Job* Pool::allocate()
{
    if(!full())
    {
        return &_storage[_allocatedJobs++];
    }
    else
    {
        return nullptr;
    }
}
```

*Note that there are no atomics involved in `Pool::allocate()`. The system
uses per-worker pools, which means Job allocations will be done from the
worker thread only. If this were not the case, `Pool::allocate()` would
have to follow a CAS strategy to increment the allocated jobs object
similar to what `JobQueue::steal()` and `JobQueue::pop()` doo (See
bellow).*

Clearing the storage (for situations such as restarting the job system for
the next frame of your game engine) is as simple as setting the counter to
0, since jobs are POD objects and no destructor calls are needed. We
simply reuse the job objects from the previous iteration:

``` cpp
void Pool::clear()
{
    _allocatedJobs = 0;
}
```

Finally, `Job::full()` definition is straightforward:

``` cpp
bool Pool::full() const
{
    return _allocatedJobs == _storage.size();
}
```

All the `createXXX()` functions are basically calls to `Pool::allocate()`
followed by a constructor call:

``` cpp
Job* Pool::createJob(JobFunction jobFunction)
{
    Job* job = allocate();

    if(job != nullptr)
    {
        new(job) Job{jobFunction};
        return job;
    }
    else
    {
        return nullptr;
    }
}
```

### Binding data to a job

One optimization you could apply to the `Job` type is to align it to the
cache line boundary to prevent false sharing. You can do so by using C++11
`alignas()`, but we can explicitly declare the padding bytes and use them
for our convenience:

``` cpp
class Job
{
    ...

private:
    void(*_jobFunction)(Job&);
    Job* _parent;
    std::atomic_size_t _unfinishedJobs;

    static constexpr std::size_t JOB_PAYLOAD_SIZE = sizeof(_jobFunction)
                                                  + sizeof(_parent)
                                                  + sizeof(_unfinishedJobs);

    static constexpr std::size_t JOB_MAX_PADDING_SIZE = 
        std::hardware_destructive_interference_size;
    static constexpr std::size_t JOB_PADDING_SIZE = JOB_MAX_PADDING_SIZE - JOB_PAYLOAD_SIZE;

    std::array<unsigned char, JOB_PADDING_SIZE> _padding;
};
```

*I have to confess in my real code I have a plain 64 there... But since
using cool C++17 stuff works better for blog posts (This does not strictly
have to compile, right?), let me take my C++ DeLorean and use
[`std::hardware_destructive_interference_size`](http://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size).*

Now we have a bunch of unused bytes where we can put user defined POD
data:

``` cpp
class Job
{
public:
    template<typename Data>
    std::enable_if_t<
        std::is_pod<Data>::value &&
        (sizeof(Data) <= JOB_PADDING_SIZE)
    >
    setData(const Data& data)
    {
        std::memcpy(_padding.data(), &data, sizeof(Data));
    }

    template<typename Data>
    const Data& getData() const
    {
        return *reinterpret_cast<const Data*>(_padding.data());
    }

    template<typename Data>
    Job(void(*jobFunction)(Job&), const Data& data, Job* parent = nullptr)
        : Job{jobFunction, parent}
    {
        setData(data);
    }

    ...
};
```

Again, since bound data is of POD types no destructor calls are needed, so
we are ok binding data, running the job, and forgetting about job data
after that.

### Binding non-POD data to a job

Using PODs by default makes 99% of the simple cases fast, but there are situations
where binding non-POD data could be useful. Consider a lambda:

``` cpp
Job job{[]Job& job
{
    std::cout << "hello!\n";
}};
```

This works since non capturing lambdas are implicitly convertible to
a function pointer. But what happens if we want to use a capturing lambda
in a job?

``` cpp
std::string message = "hello!";

Job job{[message]Job& job
{
    std::cout << message << "\n";
}};
```

This no longer works since the lambda is capturing the `message` string.
When a lambda captures something (A variable, `this`, etc) it cannot be
converted to a function anymore since the lambda object could be stateful.

Let's try again:

``` cpp
std::string message = "hello!";
Job job{[](Job& job)
{
    const std::string& message = job.getData<std::string>();
    std::cout << message << "\n";
}, message};
```

Obviously this approach does not work because `std::string` is not a POD
type. We cannot `memcpy()` it into the job padding storage, and also its
destructor must be called once the job goes out of scope.

There's another option: Our job allocator uses a preallocated array of
`Job` objects, an array that is not going to be re-allocated again during
its lifetime. This means our jobs will not be "moved" nor copied along the
use of the engine, which make their padding bytes perfect candidates for
raw object storage:

``` cpp
class Job
{
public:
    template<typename T, typename... Args>
    void constructData(Args&&... args)
    {
        new(_padding.data())(std::forward<Args>(args)...);
    }
};
```

Now we can use `Job::constructData<T>()` to bind a non-POD object
into a job by constructing it in the job padding. Let's wrap everything into
a `closure()` function:

``` cpp
template<typename Function>
void closure(Job* job, Function function)
{
    auto jobFunction = [](Job& job)
    {
        const auto& function = job.getData<Function>();

        function(job);
    }

    new(job) Job{jobFunction};
    job.constructData<Function>(function);
}

closure(allocateJob(), [message](Job& job)
{
    std::cout << message << "\n";
});
```

Tah dah! Well... not exactly. We were finally able to bind a non-POD object
to a job, but non-POD objects have an associated destructor that must be called
at the end of their lifetime. Since we used placement new to initialize an object
on the job storage, we have to explicitly call the object destructor ourselves.
We can change `closure()` implementation to do that:

``` cpp
template<typename Function>
void closure(Job* job, Function function)
{
    auto jobFunction = [](Job& job)
    {
        const auto& function = job.getData<Function>();

        function(job);

        // Destroy the bound object after
        // running the job
        function.~Function();
    }

    new(job) Job{jobFunction};
    job.constructData<Function>(function);
}
```

Now, what happens with child jobs? Our `closure()` implementation above assumes
that nobody will use the bound closure after the job is run, but that may not be
true if a child job depends on data from its parent:

``` cpp
Job* root = allocateJob();
std::string message = "hello!";
closure(root, [message](Job& root)
{
    for(std::size_t i = 0; i < 42; ++i)
    {
        closure(allocateJob(&root), [&message](Job& child)
        {
            std::cout << message << "\n";
        });
    }
})
```

The code above may look right, but nobody guarantees that child jobs will be
scheduled before the `root` job run finishes. In the worst case, all jobs
are scheduled after `root` job has been run, resulting in dangling references
to the `message` variable captured by `root`.

We can patch `Job::run()` and `Job::finish()` to fix the issue: First, allow users
to install a callback during the execution of a job, callback that will be executed
when the job is finished:

``` cpp
Job job{[](Job& job)
{
    job.onFinished([](Job& job)
    {
        std::cout << "job finished!\n";
    })
}};
```

*Remember a job is considered as finished not after the job is run only
but when all its child jobs are considered finished too.*

where `Job::onFinished()` can reuse the job function pointer:

``` cpp
void Job::onFinished(JobFunction callback)
{
    _jobFunction = callback;
}
```

Now rewrite `Job::run()` to track changes in the job function during the execution
of a job:

``` cpp
void Job::run()
{
    auto jobFunction = _jobFunction;

    _jobFunction(*this);

    if(_jobFunction == jobFunction)
    {
        // The function has not changed,
        // mark the job as "no callback"
        _jobFunction = nullptr;
    }

    finish();
}
```

finally, execute the user defined callback when the job is finished:

``` cpp
void Job::finish()
{
    _unfinishedChildJobs--;

    if(finished())
    {
        if(_parent != nullptr)
        {
            _parent->finish();
        }

        if(_jobFunction != nullptr)
        {
            // Run the onFinished callback
            _jobFunction(*this);
        }
    }
}
```

*This technique assumes all non-POD data is passed as a captured variable
by the job function. I like it because passing extra job data requires no
special syntax, just capturing variables in a lambda. Also, manually
registering a on-finished callback works with POD resources too, such as
an OpenGL resource that must be released after all the job tree is
processed.

Also, my real implementation of `closure()` is a bit more complex, using
an static if to dynamically allocate/deallocate closures if they not fit
into the job padding storage*.

## Workers

At the beginning of the post I've explained that our job system would consist in
a set of N workers. A worker is basically a thread and a job pool
associated with it:

``` cpp
class Worker
{
public:
    enum class Mode
    {
        Background,
        Foreground
    };

    enum class State
    {
        Idle,
        Running,
        Stopping
    };

    Worker(Engine* engine, std::size_t maxJobs, Mode mode = Mode::Background);
    ~Worker();

    void start();
    void stop();
    bool running() const;
    Pool& pool();
    void submit(Job* job);
    void wait(Job* job);

private:
    Pool _pool;
    Engine* _engine;
    JobQueue _queue;
    std::thread::id _threadId;
    std::thread _thread;
    std::atomic<State> _state;
    std::atomic<Mode> _mode;

    Job* getJob();
    void join();
};
```

Workers can be run in two modes, `Worker::Mode::Background` and
`Worker::Mode::Foreground`, which means the worker executes jobs in its own
owned thread or if it only runs work on the caller thread (Without
fetching work in the background). This is done to have N-1 spawned worker
threads, and one worker running on the main thread.

A worker also has an associated work queue where the user submits jobs:

``` cpp
void Worker::submit(Job* job)
{
    _workQueue.push(job);
}
```

Background workers fetch work in an infinite loop run by the worker thread:

``` cpp
while(_running)
{
    Job* job = getJob();

    if(job != nullptr)
    {
        job->run();
    }
}
```

while foreground workers use `Worker::wait()` to run jobs in the caller thread
waiting until an specific job is finished:

``` cpp
void Worker::wait(Job* waitJob)
{
    while(!waitJob->finished())
    {
        Job* job = getJob();

        if(job != nullptr)
        {
            job->run();
        }
    }
}
```

All the magic behind job stealing is implemented in `Worker::getJob()`:

``` cpp
Job* Worker::getJob()
{
    Job* job = _workQueue.pop();

    if(job != nullptr)
    {
        job->run();
    }
    else
    {
        Worker* worker = _engine->randomWorker();

        if(worker != this)
        {
            Job* job = worker->_workQueue.steal();

            if(job != nullptr)
            {
                return job;
            }
            else
            {
                std::this_thread::yield();
                return nullptr;
            }
        }
        else
        {
            std::this_thread::yield();
            return nullptr;
        }
    }
}
```

A worker first tries to get a job from its work queue. If there are no
jobs left in its own queue, the worker asks the engine to pick a random
worker. Then the worker proceeds to steal work from that worker queue,
taking care to not steal work from itself first (It could be that the
engine returned this same worker).

In case neither our worker or the worker returned by the engine have more
work to do, we yield the worker and return no job.

## Job queues

One of the most interesting posts in the molecular series is the
implementation of a lock-free job queue.  The job queue is basically
a double-ended queue implemented as a pre-allocated vector, where:

 - A worker inserts and removes jobs from its queue in a LIFO fashion,
   using `JobQueue::push()` and `JobQueue::pop()`. This means no concurrent calls
   to `pop()` and `push()` could happen (Since these are only called from the worker
   thread).

 - Other workers steal jobs from the queue in a FIFO fashion,
   using `JobQueue::steal()`. `steal()` operations can be invoked concurrently while
   a `pop()` or `push()` operation is running in the queue owner thread.

``` cpp
class JobQueue
{
public:
    JobQueue(std::size_t maxJobs);

    bool push(Job* job);
    Job* pop();
    Job* steal();
    std::size_t size() const;
    bool empty() const;

private:
    std::vector<Job*> _jobs;
    std::atomic<int> _top, _bottom;
};
```

### Inserting Jobs

`JobQueue::push()` increments the `bottom` of the queue:

``` cpp
bool JobQueue::push(Job* job)
{
    int bottom = _bottom.load(std::memory_order_acquire);

    if(bottom < static_cast<int>(_jobs.size()))
    {
        _jobs[bottom] = job;
        _bottom.store(bottom + 1, std::memory_order_release);

        return true;
    }
    else
    {
        return false;
    }
}
```

Here we use acquire-release semantics to make sure the compiler will not
reorder the assignment to the array with the bottom increment. As the
molecular post notes, this could lead to concurrent `steal()` calls
returning jobs that are not in their queue slots yet, actually returning
garbage pointers.

*Besides the array asignment order, `pop()` has no more race problems with
its (potential) concurrent enemy, `steal()`, since the only thing that
could happen is that a `steal()` call is executed before the increment is
done, making steal to fail if there are no jobs in the queue from its
point of view, besides a new job was actually inserted by `push()`.*

### Extracting jobs

`JobQueue::pop()` has to decrement `bottom`, and make sure no concurrent
`steal()` calls are trying to return the same job:

``` cpp
Job* JobQueue::pop()
{
    int bottom = _bottom.load(std::memory_order_acquire);
    bottom = std::max(0, bottom - 1);
    _bottom.store(bottom, std::memory_order_release);
    int top = _top.load(std::memory_order_acquire);

    if(top <= bottom)
    {
        Job* job = _jobs[bottom];

        if(top != bottom)
        {
            // More than one job left in the queue
            return job;
        }
        else
        {
            int expectedTop = top;
            int desiredTop = top + 1;

            if(!_top.compare_exchange_weak(expectedTop, desiredTop,
                    std::memory_order_acq_rel))
            {
                // Someone already took the last item, abort
                job = nullptr;
            }

            _bottom.store(top + 1, std::memory_order_release);
            return job;
        }
    }
    else
    {
        // Queue already empty
        _bottom.store(top, std::memory_order_release);
        return nullptr; 
    }
}
```

First, `pop()` decrements `bottom` and then reads the current value of
`top`, this ordering in the reads protects `pop()` against concurrent
calls to `steal()` in the meantime (`steal()` works only if there are jobs
to steal left in the range `[top, bottom)`. If you decrement `bottom`
first you reduce the chance of concurrent `steal()`s to return jobs).

The most important part of `pop()` is the initialization of `job`: `pop()`
and `steal()` access to different ends of the queue, so the only case
where they could be fighting for the same job is when only one job is left
in the queue. In that case, both `bottom` and `top` point to the same
queue slot, and we have to make sure only one thread is returning this
last job.

`pop()` ensures this doing a nice trick: Before returning the job, it
checks if a concurrent call to  `steal()` happened after we read `top`, by
doing a CAS to increment top. If the CAS fails, it means `pop()` has lost
a race against a concurrent `steal()`call, returning nullptr `Job` in that
case. If the CAS succeeds, `pop()` won and incremented `top` preventing
further `steal()` calls to extract the last job again.


## The engine

``` cpp
class Engine
{
public:
    Engine(std::size_t threads, std::size_t jobsPerThread);

    Worker* randomWorker();
    Worker* threadWorker();

private:
    cpp::StaticVector<Worker> _workers;

    Worker* findThreadWorker(const std::thread::id threadId);
};
```

*`cpp::StaticVector<T>` is a custom container implementing a simple
pre-allocated array where elements can be added and initialized in place.
This is not possible with `std::vector` directly since it requires value
types to be moveable at least, which is not the case with `std::mutex` or
our `Worker` for example.*

As you can see the `Engine` (Or `Manager`, `System`, `Runner`, whatever
generic and non-meaningful name you choose) owns a set of n workers,
m jobs pool each. It also exposes a function to return the worker
associated with the calling thread (if any) and a function that randomly
returns one of the workers.

*The main difference from molecule is that I have a class representing the
job system that can be instantiated on demand, instead of having a global
engine API. The global approach makes more sense in the molecule use case
since the whole engine pipeline would be based on running MxN jobs in parallel
per frame. Here the parallelism is not inherent to the parsing system but
an opt-in feature.*

When initializing the engine, we allocate all the workers and start them:

``` cpp
Engine::Engine(std::size_t workerThreads, std::size_t jobsPerThread) :
    _workers{workerThreads}
{
    std::size_t jobsPerQueue = jobsPerThread;
    _workers.emplaceBack(this, jobsPerQueue, Worker::Mode::Foreground);

    for(std::size_t i = 1; i < workerThreads; ++i)
    {
        _workers.emplaceBack(this, jobsPerQueue, Worker::Mode::Background);
    }

    for(auto& worker : _workers)
    {
        worker.run();
    }
}
```

Note that the first worker is initialized in `Foreground` mode while the
rest are initialized in `Background` mode. This is done so the thread
initializing the engine (For example, the main application thread) could
contribute to the work too by waiting for jobs using `Worker::wait()`. For
a system of N cores, initializing an `Engine` with N threads is fine,
because `Engine` already cares about the initializer thread and assigns
it a worker.

`Engine::findThreadWorker()` function searches for the engine `Worker`
that is running on the given thread:

``` cpp
Worker* Engine::findThreadWorker(const std::thread::id threadId)
{
    for(auto& worker : _workers)
    {
        if(worker.threadId() == threadId)
        {
            return &worker;
        }
    }

    return nullptr;
}
```

so `Engine::threadWorker()` could be as simple as:

``` cpp
Worker* Engine::threadWorker()
{
    return findThreadWorker(std::this_thread::get_id());
}
```

On the other hand, `Engine::randomWorker()` uses a normal distribution to
return one of the currently active workers:

``` cpp
Worker* Engine::randomWorker()
{
    std::uniform_int_distribution<std::size_t> dist{0, _workers.size()};
    std::default_random_engine randomEngine{std::random_device()()};

    Worker* worker = &_workers[dist(randomEngine)];

    if(worker->running())
    {
        return worker;
    }
    else
    {
        return nullptr;
    }
}
```

*Only the currently active because it could happen that a worker continues
running and asking for worker where to stole jobs, while the other workers
are being stopped. This is the case during engine destruction.*

## A hello world example

Noe that we have completed the API, let's write a parallel-for hello world
example to compare our results with molecular:

``` cpp
// 4 threads, 60k jobs
Engine engine{4, 60*1000};
Worker* worker = engine.threadWorker();

Job* root = worker->pool().createJob([](Job& job) { // NOP });
{
    // NOP
});

for(std::size_t i = 0; i < 60*1000; ++i)
{
    Job* child = worker->pool().createClosureJobAsChild([i](Job& job)
    {
        // NOP
    }, root);

    worker->submit(child);
}

worker->submid(root);
worker->wait(root);
```

Running this example on an Intel Core i7 6560U (2.2GHz, 4 cores, no
hyperthreading) takes 25ms approx, which is **40 times slower than the
molecular example** *In my deffense, I'm running this on a linux VM...*

This could not be performance-ready **at all** for a 60 fps game engine,
but I'm glad it works ok after three weeks of segfaults and gdb sessions.
It was my first take on true lock-free stuff and I'm really happy with
what I've learned.

## My use case

Let me show a simple example of the use case I have in mind:

``` cpp
#include <siplasplas/jobs/engine.hpp>
#include <siplasplas/reflection/parser/api/core/clang/index.hpp>
#include <siplasplas/constexpr/arrayview.hpp>

using namespace cpp::jobs;
using namespace cpp::reflection::parser::api::core::clang;

class Parser
{
public:
    Parser(const std::vector<std::string>& files, std::size_t threads) :
        _files{files},
        _engine{threads, 1024}
    {}

    void run()
    {
        Job* compileFiles = _engine.threadWorker()->pool().createClosureJob(
            [this](Job& job)
            {
                for(const auto& file : _files)
                {
                    std::cout << "Parser::run(): " << file << std::endl;

                    parseFile(file, job);
                }
            }
        );

        _engine.threadWorker()->submit(compileFiles);
        _engine.threadWorker()->wait(compileFiles);
    }

    TranslationUnit parseFile(const std::string& file)
    {
        std::cout << "Parser::parseFile(\"" << file << "\")" << std::endl;

        return _index.parse(file, CompileOptions()
            .std("c++14")
            .I(SIPLASPLAS_LIBCLANG_INCLUDE_DIR)
            .I(SIPLASPLAS_LIBCLANG_SYSTEM_INCLUDE_DIR)
            .I(SIPLASPLAS_INCLUDE_DIR)
            .I(SIPLASPLAS_REFLECTION_OUTPUT_DIR)
            .I(SIPLASPLAS_EXPORTS_DIR)
            .I(CTTI_INCLUDE_DIR)
        );
    }

    void parseFile(const std::string& file, Job& parentJob)
    {
        Job* fileJob = _engine.threadWorker()->pool().createClosureJobAsChild(
            [this, file, &parentJob](Job& fileJob)
            {
                if(!alreadyParsed(file))
                {
                    auto tu = parseFile(file);
                    auto inclusions = tu.inclusions();
                    storeTu(file, tu);

                    for(const auto& inclusion : inclusions)
                    {
                        parseFile(inclusion.file().fileName().str().str(), fileJob);
                    }
                }
                else
                {
                    std::cout << "Parser::parseFile(\"" << file << "\"): Already parsed, skipping" << std::endl;
                }
            },
            &parentJob
        );

        if(fileJob)
        {
            _engine.threadWorker()->submit(fileJob);
        }
    }

    const std::unordered_map<std::string, TranslationUnit>& translationUnits() const
    {
        return _tus;
    }
private:
    Index _index;
    std::vector<std::string> _files;
    mutable std::recursive_mutex _lockTus;
    std::unordered_map<std::string, TranslationUnit> _tus;
    Engine _engine;

    bool alreadyParsed(const std::string& file) const
    {
        std::lock_guard<std::recursive_mutex> guard{_lockTus};
        return _tus.find(file) != _tus.end();
    }

    void storeTu(const std::string& file, TranslationUnit& tu)
    {
        std::lock_guard<std::recursive_mutex> guard{_lockTus};

        if(!alreadyParsed(file))
        {
            _tus.emplace(file, std::move(tu));
        }
        else
        {
            std::cout << "Parser::storeTu(\"" << file << "\"): Already parsed, will be ignored..." << std::endl;
        }
    }
};

int main()
{
    Parser parser{ {
        SIPLASPLAS_INCLUDE_DIR "/siplasplas/reflection/parser/api/core/clang/index.hpp"
    }, 4};
    parser.run();

    for(const auto& keyValue : parser.translationUnits())
    {
        std::cout << "TranslationUnit: " << keyValue.second.spelling() << std::endl;
    }
}
```

*Three weeks to write a lock-free job system and now you put mutexes
everywhere? Please don't judge me, is just a 20 loc example...*

This is a toy proof of concept of the parsing pipeline I'm working on. The
example shows a `Parser` class that recursively parses all the included
headers of a given header file, spawning one parallel job for each file
parsed. It runs damnly fast compared to doing all the work in one thread,
as we all known parsing a C++ file could take ages...

## Future work

I would like to continue working on the engine, adding continuations and
parallel programming primitives. Also, integrating boost.coroutine to be
able to yield jobs could be really interesting.
