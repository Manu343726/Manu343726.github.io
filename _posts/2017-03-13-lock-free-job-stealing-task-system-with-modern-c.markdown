---
title: Lock-free job stealing task system with modern c++
date: 2017-03-13T15:58:13+01:00
---

In [my previous post]() I (re)introduced you to my main personal project,
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
    }
}
```

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

private:
    std::atomic_size_t _allocatedJobs;
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
        return _storage[_allocatedJobs++];
    }
    else
    {
        return nullptr;
    }
}
```

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

this works since non capturing lambdas are implicitly convertible to a function
pointer. But what happens if we want to use a capturing lambda in a job?

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

There's another option: Our job allocator uses a preallocated array of `Job`
objects, an array that will not be re-allocated again during its lifetime.
This means our jobs will not be "moved" nor copied along the use of the engine,
which make their padding bytes perfect candidates for raw object storage.

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
at the end of its lifetime. Since we used placement new to initialize an object
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

*Remember a job is considered as finished not after the job is run but also when all
its child jobs are considered finished too.*

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
special syntax, just capturing variables in a lambda.*

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

A worker first tries to get a job from its work queue. If there are no jobs
left in the the queue, the worker asks its engine to pick a random worker.
Then the worker proceeds to steal work from that worker queue, taking care to
not steal work from itself first (It could be that the engine returned this same
worker).

In case neither our worker or the worker returned by the engine have more work
to do, we yield the worker and return no job.

## Job queues

One of the most interesting (imho) posts in the molecular series was the implementation
of a lock-free job queue.  
The job queue is basically a double-ended queue where:

 - A worker inserts and removes jobs from its queue in a LIFO fashion,
   using `JobQueue::push()` and `JobQueue::pop()`. This means no concurrent calls
   to `pop()` and `push()` could happen.

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


``` cpp class Engine { public: Engine(std::size_t n, std::size_t m);

    Worker* randomWorker();
    Worker* threadWorker();

private:
    std::vector<std::unique_ptr<Worker>> _workers;
};
```

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
a feature.*

### 
