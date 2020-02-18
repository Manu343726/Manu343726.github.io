---
title: Always measure (A raytracing runtime post-mortem)
date: 2019-06-20 00:00:00 -07:00
tags:
- c++
comments: true
---

Most of my colleages don't know I'm a big fan of graphics programming and that
I used to write a lot of hobby projects on the topic (A DirectX 9 2d engine,
a software renderer from scratch, etc).  Life is complex and for multiple
reasons I've never worker professionally in the field, but my interest kept
there.

Some days ago I got tired of writing docs for my next
[tinyrefl](https://github.com/Manu343726/tinyrefl) release, and started to
thinkabout a new hobby project to distract myself from my main project
(*Yeah...*). So I pulled again my Amazon Kindle account and downloaded all
"Raytracing In A Weekend" series by Peter Shirley.

> For those of you new to this field, raytracing is a way of doing computer graphics where
> scene ilumination is computed by simulating light rays across the scene,
> with complex light interactions such as reflection, refraction, etc.
> This technique, in contrast with "rasterization" used in real time graphics,
> gives realistic images at the expense of long computation times.
>
> Peter Shirley books [are available online](https://github.com/petershirley/raytracinginoneweekend/) and
> provide probably the best introduction to ray tracing from the
> implementation perspective.

This was not the first time I read Shirley's books, but now I wanted to read
the books in depth while working on my own implementation.

## The implementation

I called my implementation `raytracer` (*Naming is hard, you know...*), and
hosted it [on github](https://github.com/Manu343726/raytracer).

The project is organized as three different modules:

 - **A library with all the raytracing datatypes and math routines**: vector and
   color algebra, rgb and hsv color maps, random generation, intersections,
   etc.
 
 - **A lock-free job scheduler** to parallelize the rendering. A couple of
   years ago I wrote an implementation of a lock-free work stealing task
   scheduler using C++11 atomics (blog post
   [here](https://manu343726.github.io/2017-03-13-lock-free-job-stealing-task-system-with-modern-c/)),
   and I thought that this project could be a good excuse to put such as thing
   in practice.

 - **A common runtime library** for easy setup of renderers. The idea is that
   all renderer setup is implemented by this library and you only have to write
   a `kernel()` function and link it against the runtime library to get
   a working renderer executable:
 
   ``` cpp
   void kernel(
       const float x, // [0, 1] in screen space
       const float y, // [0, 1] in screen space
       const kernel_constants& constants,
       color& pixel)
   {
        pixel = color::hsv(x);
   }
   ```

   The runtime library takes care of setting up a renderer parallelized so that
   each pixel is computed as an independent job (task). It also implements
   reflection of runtime configuration and kernel constants so that these can
   be read from a JSON config file or from command line arguments:

   ``` cpp
   struct settings
   {
     rt::kernel_constants kernel_constants;
        
     [[
       rt::short_name("t"),
       rt::description("Number of job engine parallel threads (default: all [hardware threads])")
     ]]
     std::string threads = "all";

     [[
       rt::short_name("l"),
       rt::description(
         "Logging level (TRACE, DEBUG, INFO, WARNING, ERROR, CRITICAL, OFF) (default: INFO)")
     ]]
     std::string log_level = "INFO";

     [[
       rt::short_name("f"),
       rt::description("Output image file (default: output.ppm")
     ]]
     std::string output_file = "output.ppm";

     std::unordered_map<std::string, std::string> jobs_per_thread;

     [[rt::description(
         "Default thread job pool size (default: all)")]]
     std::string default_jobs_per_thread = "all";
   };
   ```

   ``` bash
   $ ./bin/mandelbrot --help                   0,0642s
    raytracer lib client
    Usage:
      raytracer-client [OPTION...]

      -i, --iterations arg          Number of iterations (default: 1)
      -s, --samples arg             Semples per pixel (default: 1)
      -w, --width arg               Width of the output image in pixels (default:
                                    800)
      -h, --height arg              Height of the output image in pixels
                                    (default: 600)
          --aspect_ratio arg        Aspect ratio used by the renderer (default:
                                    width/height)
      -t, --threads arg             Number of job engine parallel threads
                                    (default: all [hardware threads])
      -l, --log_level arg           Logging level (TRACE, DEBUG, INFO, WARNING,
                                    ERROR, CRITICAL, OFF) (default: INFO)
      -f, --output_file arg         Output image file (default: output.ppm
          --default_jobs_per_thread arg
                                    Default thread job pool size (default: all)
          --help                    Display this help and exit
      -c, --config-file arg         Full path to the configuration JSON file
   ```

   Command line options override settings written in the configuration file, so
   testing different scenarios is really simple.  
   For example, the following config file declares an scenario where the
   renderer uses all hardware threads available, the main thread has a job pool
   big enough to allocate all the `kernel()` jobs, all background threads have
   an empty job pool (All `kernel()` jobs are issued from the main thread), and
   the renderer is configured with 4k resolution and one sample per pixel:

   ``` json
   {
       "log_level": "INFO",
       "output_file": "output.ppm",

       "threads": "all",
       "jobs_per_thread": {
           "thread-0": "100%"
       },
       "default_jobs_per_thread": "0%",

       "kernel_constants": {
           "iterations": 100,
           "samples_per_pixel": 1,
           "screen_width": 4096,
           "screen_height": 2160,
           "aspect_ratio": 1.896296
       }
   }
   ```

   If I run the renderer with the above config file I get:

   ``` bash
   $ ./bin/renderer -c ../settings.json
   [2019-06-19 03:09:09.221] [info] loading config file ../settings.json ...
   [2019-06-19 03:09:09.356] [info] runtime settings:
   [2019-06-19 03:09:09.356] [info]  - worker threads: all (4)
   [2019-06-19 03:09:09.357] [info]  - output file: output.ppm
   [2019-06-19 03:09:09.357] [info]  - max allocated jobs per thread:
   [2019-06-19 03:09:09.357] [info]     thread 0: 100% (8847360 jobs)
   [2019-06-19 03:09:09.357] [info]     thread 1: 0% (0 jobs)
   [2019-06-19 03:09:09.357] [info]     thread 2: 0% (0 jobs)
   [2019-06-19 03:09:09.357] [info]     thread 3: 0% (0 jobs)
   [2019-06-19 03:09:09.357] [info] constants:
   [2019-06-19 03:09:09.357] [info]  - iterations: 100
   [2019-06-19 03:09:09.357] [info]  - samples per pixel: 1
   [2019-06-19 03:09:09.357] [info]  - screen width: 4096 pixels
   [2019-06-19 03:09:09.357] [info]  - screen height: 2160 pixels
   [2019-06-19 03:09:09.357] [info]  - aspect ratio: 1.8963
   [2019-06-19 03:09:09.357] [info] 
   [2019-06-19 03:09:09.357] [info] Starting...
   [2019-06-19 03:09:23.727] [info] elapsed time: 13675 ms
   [2019-06-19 03:09:23.809] [info] Done. Saving rendering to file...
   ```

   But if I override some settings, say screen resolution, I get this
   instead:

   ``` bash
   $ ./bin/renderer -c ../settings.json -w 800 -h 600
   [2019-06-19 03:13:40.624] [info] loading config file ../settings.json ...
   [2019-06-19 03:13:40.641] [info] runtime settings:
   [2019-06-19 03:13:40.641] [info]  - worker threads: all (4)
   [2019-06-19 03:13:40.641] [info]  - output file: output.ppm
   [2019-06-19 03:13:40.641] [info]  - max allocated jobs per thread:
   [2019-06-19 03:13:40.641] [info]     thread 0: 100% (480000 jobs)
   [2019-06-19 03:13:40.642] [info]     thread 1: 0% (0 jobs)
   [2019-06-19 03:13:40.642] [info]     thread 2: 0% (0 jobs)
   [2019-06-19 03:13:40.642] [info]     thread 3: 0% (0 jobs)
   [2019-06-19 03:13:40.642] [info] constants:
   [2019-06-19 03:13:40.642] [info]  - iterations: 100
   [2019-06-19 03:13:40.642] [info]  - samples per pixel: 1
   [2019-06-19 03:13:40.642] [info]  - screen width: 800 pixels
   [2019-06-19 03:13:40.642] [info]  - screen height: 600 pixels
   [2019-06-19 03:13:40.642] [info]  - aspect ratio: 1.8963
   [2019-06-19 03:13:40.642] [info] 
   [2019-06-19 03:13:40.642] [info] Starting...
   [2019-06-19 03:13:41.463] [info] elapsed time: 752 ms
   [2019-06-19 03:13:41.469] [info] Done. Saving rendering to file...
   ```

## Then weird things start to happen...


If you managed to keep reading up to this point you may have noticed I've not
shown any raytracing code yet. That's because while I have been implementing
the math library with raytracing primitives I've focused my tests to get the
runtime as smooth as possible (So testing the final raytracer would be easy).

To do so, I've written a [Mandelbrot set](https://en.wikipedia.org/wiki/Mandelbrot_set) kernel to use it for profiling:

``` cpp
void kernel(
    const float x,
    const float y,
    const kernel_constants& constants,
    color& pixel)
{
    complex origin{
        (x - 0.7f) * constants.aspect_ratio * 2.0f,
        (y - 0.5f) * 2.0f};

    complex z{0.0f, 0.0f};

    std::size_t i = 0;

    while(i < constants.iterations && std::abs(z) < 2.0f)
    {
        z = z * z + origin;
        i++;
    }

    if(i < constants.iterations)
    {
        pixel = color::hsv(static_cast<float>(i) / constants.iterations);
    }
}
```

which produces an image like the following:

![](/img/mandelbrot.png)  
*Mandelbrot set 800x600 100x iterations*

This image was generated with the 4K JSON config shown before, but changing the
resolution to 800x600:

``` bash
$ ./bin/mandelbrot -c ../settings.json -w 800 -h 600
[2019-06-19 03:13:40.624] [info] loading config file ../settings.json ...
[2019-06-19 03:13:40.641] [info] runtime settings:
[2019-06-19 03:13:40.641] [info]  - worker threads: all (4)
[2019-06-19 03:13:40.641] [info]  - output file: output.ppm
[2019-06-19 03:13:40.641] [info]  - max allocated jobs per thread:
[2019-06-19 03:13:40.641] [info]     thread 0: 100% (480000 jobs)
[2019-06-19 03:13:40.642] [info]     thread 1: 0% (0 jobs)
[2019-06-19 03:13:40.642] [info]     thread 2: 0% (0 jobs)
[2019-06-19 03:13:40.642] [info]     thread 3: 0% (0 jobs)
[2019-06-19 03:13:40.642] [info] constants:
[2019-06-19 03:13:40.642] [info]  - iterations: 100
[2019-06-19 03:13:40.642] [info]  - samples per pixel: 1
[2019-06-19 03:13:40.642] [info]  - screen width: 800 pixels
[2019-06-19 03:13:40.642] [info]  - screen height: 600 pixels
[2019-06-19 03:13:40.642] [info]  - aspect ratio: 1.8963
[2019-06-19 03:13:40.642] [info] 
[2019-06-19 03:13:40.642] [info] Starting...
[2019-06-19 03:13:41.463] [info] elapsed time: 752 ms
[2019-06-19 03:13:41.469] [info] Done. Saving rendering to file...
```
*800x600 x100 iterations/pixel Mandelbrot set, Intel Core i7-6560U @2.20 GHz x4
threads*

If I did the `std::chrono` thing right the rendering took around 752
milliseconds.

> Note I'm not measuring the job scheduler setup time nor the time it takes the
> renderer to dump the buffer to a PPM file. However job creation and
> scheduling is measured since jobs are submitted and processed on the fly from
> the main thread. For reference, this is the function that schedules
> `kernel()` jobs in parallel, following a fork-join pattern:
>
> ``` cpp
> void canvas::foreach(
>   canvas::pixel_function          function,
>   const kernel_constants&         constants,
>   const std::size_t               threads,
>   const std::vector<std::size_t>& jobsPerThread)
> {
>   rt::jobs::Engine engine{threads, jobsPerThread, pixel_count()};
>
>   // Time measurement starts here, all scheduler initialization is done
>   // in the Engine constructor
>   const auto start = std::chrono::high_resolution_clock::now();
>
>   const float x_ratio = 1.0f / _width;
>   const float y_ratio = 1.0f / _height;
>
>   auto* worker = engine.threadWorker();
>
>   auto* root = worker->pool().createJob([](rt::jobs::Job&) {});
>
>   for(std::size_t row = 0; row < _height; ++row)
>   {
>     for(std::size_t column = 0; column < _width; ++column)
>     {
>       auto&       pixel = this->pixel(row, column);
>
>       auto* pixelJob = worker->pool().createClosureJobAsChild(
>         [function, row, column, x_ratio, y_ratio, &pixel, constants](rt::jobs::Job& job) {
>           for(std::size_t i = 0; i < constants.samples_per_pixel; ++i)
>           {
>             const float x     = (column + rt::random()) * x_ratio;
>             const float y     = 1.0f - (row + rt::random()) * y_ratio;
>             color local_pixel = color::rgb(0.0f, 0.0f, 0.0f);
>
>             function(x, y, constants, local_pixel);
>             pixel += local_pixel;
>           }
>
>           pixel /= constants.samples_per_pixel;
>       }, root);
>
>       worker->submit(pixelJob);
>     }
>   }
>
>   worker->submit(root);
>   worker->wait(root); // thread "blocks" here
>
>   const auto elapsed = std::chrono::high_resolution_clock::now() - start;
>
>   spdlog::info(
>     "elapsed time: {} ms",
>     std::chrono::duration_cast<std::chrono::milliseconds>(elapsed).count());
> }
> ```

I'm sure things can be optimized further, but it was a good start.

But when I switched to my desktop PC and started experimenting with other
scenarios I noticed timings were a bit weird. For example:

``` bash
./build/bin/mandelbrot -c settings.json --iterations 100
[2019-06-19 11:30:32.471] [info] loading config file settings.json ...
[2019-06-19 11:30:32.508] [info] runtime settings:
[2019-06-19 11:30:32.508] [info]  - worker threads: all (16)
[2019-06-19 11:30:32.508] [info]  - output file: output.ppm
[2019-06-19 11:30:32.508] [info]  - max allocated jobs per thread:
[2019-06-19 11:30:32.508] [info]     thread 0: 100% (8847360 jobs)
[2019-06-19 11:30:32.508] [info]     thread 1: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 2: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 3: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 4: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 5: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 6: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 7: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 8: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 9: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 10: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 11: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 12: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 13: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 14: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info]     thread 15: 0% (0 jobs)
[2019-06-19 11:30:32.508] [info] constants:
[2019-06-19 11:30:32.508] [info]  - iterations: 100
[2019-06-19 11:30:32.508] [info]  - samples per pixel: 1
[2019-06-19 11:30:32.508] [info]  - screen width: 4096 pixels
[2019-06-19 11:30:32.508] [info]  - screen height: 2160 pixels
[2019-06-19 11:30:32.508] [info]  - aspect ratio: 1.8963
[2019-06-19 11:30:32.508] [info] 
[2019-06-19 11:30:32.508] [info] Starting...
[2019-06-19 11:30:39.944] [info] elapsed time: 7137 ms
[2019-06-19 11:30:40.043] [info] Done. Saving rendering to file...
```
*4K x100 iterations/pixel Mandelbrot set, Ryzen 7 1700X 3.4GHz x16 threads*

Okay, 7.137 seconds. Now look at this:

``` bash
 ./build/bin/mandelbrot -c settings.json --iterations 100 --threads 50%
[2019-06-19 11:30:47.393] [info] loading config file settings.json ...
[2019-06-19 11:30:47.430] [info] runtime settings:
[2019-06-19 11:30:47.430] [info]  - worker threads: 50% (8)
[2019-06-19 11:30:47.430] [info]  - output file: output.ppm
[2019-06-19 11:30:47.430] [info]  - max allocated jobs per thread:
[2019-06-19 11:30:47.430] [info]     thread 0: 100% (8847360 jobs)
[2019-06-19 11:30:47.430] [info]     thread 1: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 2: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 3: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 4: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 5: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 6: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info]     thread 7: 0% (0 jobs)
[2019-06-19 11:30:47.430] [info] constants:
[2019-06-19 11:30:47.430] [info]  - iterations: 100
[2019-06-19 11:30:47.430] [info]  - samples per pixel: 1
[2019-06-19 11:30:47.430] [info]  - screen width: 4096 pixels
[2019-06-19 11:30:47.430] [info]  - screen height: 2160 pixels
[2019-06-19 11:30:47.430] [info]  - aspect ratio: 1.8963
[2019-06-19 11:30:47.430] [info] 
[2019-06-19 11:30:47.430] [info] Starting...
[2019-06-19 11:30:52.981] [info] elapsed time: 5252 ms
[2019-06-19 11:30:53.020] [info] Done. Saving rendering to file...
```
*4K x100 iterations/pixel Mandelbrot set, Ryzen 7 1700X 3.4GHz x8 threads*

Whaaaat? Half the threads and it gets almost 2s down?

## Profile first, cry later

I kept running those tests for a couple of hours and the numbers were
consistently weird. If you have experience with concurrent systems
something could have been a bit suspicious from the very start of this
post: **Why I'm allocating one task per pixel?**

I know things could be simpler if I just subdivide the framebuffer in 16
(*Number of hardware threads*) separated clusters and issue just one task
per cluster, but that goes against my philosophy when doing concurrency:
**Focus on declaring your tasks and their dependencies, and leave
allocation and scheduling to more experienced people**.

> IMHO I'm nowhere an expert of lock-free data structures, if that's even
> possible. So I will never put something like this job scheduler in production
> but search for alternatives online.

> Corollary: Threads and their associated control structures (Mutexes,
> condition variables, etc) **are not the right abstraction for
> concurrency but implementation details.** [*Pedantic rant off*]

In this specific case the goal of the system is to render multiple independent
pixels, so from a pure logic point of view each pixel is an independent task.
So, one task per pixel, throwing all the hard problems to the scheduler.

Going back to the real world of *Implementing Things That Work (tm)* this
approach is not optimal if for some reason your system takes more time playing
ping-pong in the task scheduler than actually doing userful work. That happens
if **the time to run each task is similar (or less) to the time it takes the
scheduler to grab a pending task and posting it on a worker thread**.

If you haven't read my post lock-free job stealing (or [the original
post](https://blog.molecular-matters.com/2015/08/24/job-system-2-0-lock-free-work-stealing-part-1-basics/)
by Stefan Reinalter) let me give you a brief explanation of how the job
scheduler works:

 - The engine creates N - 1 background threads, and manages the current
 thread (The thread that creates the engine) as implicitly created.

 - Each thread has a memory pool to allocate jobs from that thread and
   a lock-free queue to submit jobs to that thread. Allocating and posting
   a job is done in two different steps: 

   ``` cpp
   Job* job = current_thread.pool.create(); // allocates a job
   current_thread.submit(job) // puts the job in the thread work queue
   ```

 - Background threads pull jobs in a loop from their correspondent work
   queues. If the queue is empty, the thread tries to "steal" a job from
   the queue of one of the other threads (randomnly).

 - If both the thread job queue the the other random job queue return nothing,
   the thread yields (To avoid busy waits).

 - The foreground thread, the thread that created the engine, can execute
   jobs when doing waits for job completion by calling `wait(Job* job)`.
   `wait()` works similar to the background thread loops. Background
   threads can also `wait()` for a job completion as part of a job
   execution.

After building the Mandelbrot renderer with profiling enabled (`-pg` with
GCC), I run the example again through
[`uftrace`](https://github.com/namhyung/uftrace), a function call tracer
for C and C++:

``` bash
$ uftrace record ./build/bin/mandelbrot -c settings.json -w 1024 -h 768
...
Starting...
elapsed time: 2483 ms
Done. Saving rendering to file...

$ ufrace report

Total time   Self time       Calls  Function
==========  ==========  ==========  ====================
  43.046 s    6.379  s          15  std::thread::_State_impl::_M_run
  10.768 s  307.555 ms      786432  rt::jobs::Job::run
  10.219 s  373.328 ms      786431  rt::jobs::closure::$_0::_FUN
   9.724 s  157.002 ms      786432  rt::canvas::foreach::$_0::_FUN
   9.567 s    5.912  s      786431  kernel
   7.691 s    4.413  s    11215993  rt::jobs::Engine::randomWorker
   7.533 s    7.533  s     1322551  sched_yield
   6.532 s    0.360 us           1  main
   6.532 s   27.350 us           1  run
   6.531 s    4.750 us           1  kernel_runner
   6.509 s    4.964  s    11399823  rt::jobs::JobQueue::pop
   5.048 s    3.722  s     9893442  rt::jobs::JobQueue::steal
   3.624 s  221.279 ms           1  rt::canvas::dump_to_file
   3.123 s   98.572 ms      787242  fmt::v5::vformat_to
   3.024 s    1.078  s      787242  fmt::v5::internal::parse_format_string
   2.901 s  534.607 ms           1  rt::canvas::foreach
   2.285 s    2.285  s    11215993  std::uniform_int_distribution::operator()
   2.141 s    2.141  s    31130753  spdlog::logger::log
   1.765 s    1.765  s    18906517  cabsf
   1.671 s    1.671  s    19066979  __mulsc3
   1.666 s   91.735 ms           1  rt::jobs::Worker::wait
   1.489 s  307.648 ms     2359372  fmt::v5::visit_format_arg
   1.354 s    1.354  s    20450839  spdlog::details::registry::instance
   1.181 s  617.921 ms     2359297  fmt::v5::basic_writer::write_padded
```

Looking at the `Self time` column of the uftrace report we see that the
example wasted around 7 seconds yielding (See
[`sched_yield`](http://man7.org/linux/man-pages/man2/sched_yield.2.html)
row), and just around 6 seconds doing actual work in `kernel()` (Note one
`kernel()` call is issued per pixel). That means some scheduler threads
suffer from starvation because other threads are taking all the work.
Having scheduler related functions at the top of the timing report is
a clear indicator that threads are idle most of the time.

> Note this behavior is balanced between all engine threads over time,
> it's not always the same threads that take all the work but sometimes
> it's one thread which manages to get the work and others that get stuck.
>
> If you look at engine execution statistics, job execution is balanced
> across background threads but the full execution time gets longer than
> expected:
>
> ```
> [info] elapsed time: 31866 ms
> [debug] Stopping worker 0...
> [debug] Worker 0 stopped (8847360 jobs were allocated [100%], 2091173 jobs were run, 0 jobs discarded, 0 max idle cycles)
> [debug] Stopping worker 1...
> [debug] Worker 1 stopped (0 jobs were allocated [100%], 454260 jobs were run, 0 jobs discarded, 2898 max idle cycles)
> [debug] Stopping worker 2...
> [debug] Worker 2 stopped (0 jobs were allocated [100%], 441886 jobs were run, 0 jobs discarded, 4003 max idle cycles)
> [debug] Stopping worker 3...
> [debug] Worker 3 stopped (0 jobs were allocated [100%], 444041 jobs were run, 0 jobs discarded, 4264 max idle cycles)
> [debug] Stopping worker 4...
> [debug] Worker 4 stopped (0 jobs were allocated [100%], 454229 jobs were run, 0 jobs discarded, 4203 max idle cycles)
> [debug] Stopping worker 5...
> [debug] Worker 5 stopped (0 jobs were allocated [100%], 454127 jobs were run, 0 jobs discarded, 4252 max idle cycles)
> [debug] Stopping worker 6...
> [debug] Worker 6 stopped (0 jobs were allocated [100%], 446256 jobs were run, 0 jobs discarded, 4357 max idle cycles)
> [debug] Stopping worker 7...
> [debug] Worker 7 stopped (0 jobs were allocated [100%], 448462 jobs were run, 0 jobs discarded, 4404 max idle cycles)
> [debug] Stopping worker 8...
> [debug] Worker 8 stopped (0 jobs were allocated [100%], 438851 jobs were run, 0 jobs discarded, 4225 max idle cycles)
> [debug] Stopping worker 9...
> [debug] Worker 9 stopped (0 jobs were allocated [100%], 437758 jobs were run, 0 jobs discarded, 4203 max idle cycles)
> [debug] Stopping worker 10...
> [debug] Worker 10 stopped (0 jobs were allocated [100%], 453287 jobs were run, 0 jobs discarded, 4447 max idle cycles)
> [debug] Stopping worker 11...
> [debug] Worker 11 stopped (0 jobs were allocated [100%], 449126 jobs were run, 0 jobs discarded, 4489 max idle cycles)
> [debug] Stopping worker 12...
> [debug] Worker 12 stopped (0 jobs were allocated [100%], 497281 jobs were run, 0 jobs discarded, 4752 max idle cycles)
> [debug] Stopping worker 13...
> [debug] Worker 13 stopped (0 jobs were allocated [100%], 445606 jobs were run, 0 jobs discarded, 4373 max idle cycles)
> [debug] Stopping worker 14...
> [debug] Worker 14 stopped (0 jobs were allocated [100%], 444109 jobs were run, 0 jobs discarded, 4280 max idle cycles)
> [debug] Stopping worker 15...
> [debug] Worker 15 stopped (0 jobs were allocated [100%], 446908 jobs were run, 0 jobs discarded, 4541 max idle cycles)
> [info] Done. Saving rendering to file...
> ```

Running the same example with half the threads gives almost the same execution time:

``` cpp
$ uftrace record ./build/bin/mandelbrot -c settings.json -w 1024 -h 768 --threads 50%
...
Starting...
elapsed time: 2861 ms
Done. Saving rendering to file...

$ ufrace report

Total time   Self time       Calls  Function
==========  ==========  ==========  ====================
 16.705  s    2.473  s           7  std::thread::_State_impl::_M_run
  7.902  s  205.519 ms      786432  rt::jobs::Job::run
  7.535  s  271.779 ms      786431  rt::jobs::closure::$_0::_FUN
  7.169  s  112.403 ms      786432  rt::canvas::foreach::$_0::_FUN
  7.057  s    4.247  s      786431  kernel
  6.150  s    0.680 us           1  main
  6.150  s   34.670 us           1  run
  6.148  s    4.040 us           1  kernel_runner
  3.727  s  234.821 ms           1  rt::canvas::dump_to_file
  3.210  s   99.347 ms      787226  fmt::v5::vformat_to
  3.111  s    1.127  s      787226  fmt::v5::internal::parse_format_string
  2.875  s    1.778  s     5410145  rt::jobs::Engine::randomWorker
  2.539  s    1.946  s     5627477  rt::jobs::JobQueue::pop
  2.417  s  447.145 ms           1  rt::canvas::foreach
  1.671  s    1.227  s     4205707  rt::jobs::JobQueue::steal
  1.531  s  317.070 ms     2359340  fmt::v5::visit_format_arg
  1.436  s   82.958 ms           1  rt::jobs::Worker::wait
  1.370  s    1.370  s    18906520  cabsf
  1.272  s    1.272  s    19066979  __mulsc3
  1.214  s  641.059 ms     2359297  fmt::v5::basic_writer::write_padded
745.856 ms  745.697 ms    14008537  spdlog::logger::log
689.986 ms  689.986 ms     5410145  std::uniform_int_distribution::operator()
471.963 ms  471.876 ms     9016382  spdlog::details::registry::instance
419.634 ms  309.172 ms     1643366  fmt::v5::basic_writer::padded_int_writer::operator()
407.030 ms  407.030 ms     5410145  rt::jobs::Worker::running
362.867 ms  362.867 ms     1204438  sched_yield
```

but note how `sched_yield()` only takes 362 milliseconds now, and other
scheduler internal functions such as `JobQueue::steal()` or
`JobQueue::pop()` take around 4 times less time running.

To confirm my theory I rerun the all threads scenario but with 1000 iterations per
pixel instead of 100. Since the mandelbrot kernel uses this iterations
config for the escape time of the mandelbrot set check, having more
iterations per pixel means each pixel takes longer to run:

``` shell
$ uftrace record ./build/bin/mandelbrot -c settings.json -w 1024 -h 768 --iterations 1000
...
Starting...
elapsed time: 2861 ms
Done. Saving rendering to file...

$ ufrace report

  Total time   Self time       Calls  Function
  ==========  ==========  ==========  ====================
    1.034  m   20.173  s          15  std::thread::_State_impl::_M_run
    1.018  m  376.804 ms      786431  rt::jobs::Job::run
    1.018  m  963.867 ms      786431  rt::jobs::closure::$_0::_FUN
    1.017  m    1.016  m      780761  kernel
    8.542  s    0.581 us           1  main
    8.542  s   42.429 us           1  run
    8.539  s    7.524 us           1  kernel_runner
    6.325  s  851.467 ms           1  rt::canvas::foreach
    5.454  s  132.997 ms           1  rt::jobs::Worker::wait
    2.211  s  173.843 ms           1  rt::canvas::dump_to_file
    2.028  s    2.015  s      786504  fmt::v5::internal::parse_format_string
    1.235  s    1.235  s       10721  linux:schedule
  326.466 ms    8.798 ms        6879  sched_yield
  238.994 ms  176.710 ms      158243  rt::jobs::Engine::randomWorker
  170.134 ms   28.633 ms        9929  __mulsc3
  158.411 ms   29.409 ms       10205  cabsf
   71.590 ms    9.769 ms        4946  rt::jobs::JobQueue::pop
   50.050 ms    5.375 ms        2584  rt::jobs::JobQueue::steal
```

This makes more sense, scheduler functions are at the bottom of the report and `kernel()` sits at the top.

## Lessons learned

 - Always setup a new project so that it can easily debugged and profiled
 from day one.
 - When writing code it's good to put quality over performance hacks, but
 never forget about profiling. In my experience it's easier to fall into
 the trap of early pessimization than on the devil of early optimization.
 - Having hobby projects that are technically challenging is quite fun!

## Some more notes

 - `uftrace` includes a lot of options, my personal favorites being
 a ncurses-based TUI and a Google Chrome compatible timeline graph.

 - In all the profiling examples run for this post `uftrace` was run with
 a time filer to ignore functions that take less than one microsecond
 (`--time-filter 1us`). This was neccessary to avoid blocking `uftrace`:
 By default `uftrace` works by launching the profiled application as
 a child process and spawning multiple write-profile-data-to-disk threads
 that communicate with the profiled process through a socket. Whenever an
 instrumented function is left, the profiled child process sends the
 profile data to one of the write-to-disk `uftrace` threads using
 [`writev()`](eage://linux.die.net/man/2/writev). If there's a high
 workload sometimes all the profiled process threads get blocked in
 `writev()`, I guess the write-to-disk threads of `uftrace` could not cope
 with the load. Also be careful because if you force kill the process in
 that situation you may end up with a memory leak even after killing the
 process, which makes your ram usage to increase over time. 

   > I know how it sounds, that should not be possible. I'm not a linux kernel
   > expert, but I can tell you that's exactly what I see whenever that
   > happens. I have 32GB of RAM in both of my desktop PCs and I've to restart
   > them after running, blocking, and killing a uftrace session multiple times
   > if I forgot to add time filters.

 - The raytracer runtime implements antialiasing through the
 `--samples-per-pixel` option, so I could have used it to increase the per
 pixel work instead of `--iterations`. I think `--iterations` makes more
 sense for the Mandelbrot example, antialiasing is really implemented with
 raytracing in mind.

 - The first weeks of the raytracer implementation were used to fix bugs in my
 implementation of the job stealing scheduler. Looking at the post, the main
 issue was that I completely forgot to add a memory barrier after touching one
 of the markers of the queue, something that Stefan Reinalter explicitly
 explains as being *"The most important part of this implementation"* [in his
 blog
 post](https://blog.molecular-matters.com/2015/09/25/job-system-2-0-lock-free-work-stealing-part-3-going-lock-free/).
 With C++11, a `MEMORY_BARRIER` could be implemented as
 a [`std::atomic_thread_fence()`](https://en.cppreference.com/w/cpp/atomic/atomic_thread_fence).
 Check [my
 implementation](https://github.com/Manu343726/raytracer/blob/master/src/jobs/jobqueue.cpp#L23)
 and compare it with Stephan's.

   > **"Raytracing In One Weekend"**... More like "Being Wrong For Two Years
   > And A Couple Of Weeks, And Writing A Post About It". Lock-free is hard...

 - I don't expect the weird thread starvation issues to happen with the real
 raytracing scenes since raytracing is inherently expensive at the pixel
 level (Antialiasing, recursive scene marching, Montecarlo based importance
 sampling, etc).

 - All the examples shown are from the same `mandelbrot` executable
 compiled in release mode with GCC 9. Note when enabling intrumentation
 for profiling (`-pg`) timings change compared to a normal release build,
 and also running the program through the profiler changes timings even
 more. Never compare release and profiled runs, but measure diffs between
 profile sessions.

 - Timings in this post do not explicitly include the time to write the
 output image down to disk, but you can check it in the `uftrace` reports
 shown, search for the `canvas::dump_to_file()` function. As the
 resolution of the image increases the time it takes to write the `PPM`
 file becomes an issue. Right now I'm using fmt's [`fmt::print()` with
 a `std::ofstream`](https://fmt.dev/latest/api.html#std-ostream-support),
 so maybe I can be lazy this time and blame the standard streams library
 debug performance.

 - The raytracer runtime uses the work-in-progress 0.5.0 release of
 [tinyrefl](https://github.com/Manu343726/tinyrefl) to implement settings
 and scene reflection. In the following weeks (months?) I will write some
 posts about the new features of tinyrefl and how they're used in the
 raytracer runtime.

## Some cool raytracing pictures! (Work in progress)

At the time of writing this post I have completed Chapter 8 ("Diffuse
Materials") of Raytracing In One Weekend, and I'm working on the
implementation of the next chapter (Dielectrics). Here's the scene I've
configured for Chapter 8:

``` json
{
    "log_level": "INFO",
    "output_file": "output.ppm",

    "threads": "all",
    "jobs_per_thread": {
        "thread-0": "100%"
    },
    "default_jobs_per_thread": "0%",

    "canvas_follows_screen_resolution": true,
    "use_custom_aspect_ratio": false,

    "kernel_constants": {
        "iterations": 100,
        "samples_per_pixel": 100,
        "screen_width": 4096,
        "screen_height": 2160,
        "aspect_ratio": 1.896296,

        "scene": {
            "camera": {
                "position": {"x": 0.0, "y": 800.0, "z": 5000.0},
                "look_at": {"x": 0.0, "y": 0.0, "z": 1000.0},
                "up": {"x": 1.0, "y": 0.0, "z": 0.0},
                "viewport_width": 4096.0,
                "viewport_height": 2160.0,
                "fov": 1.0
            },

            "objects": [
                {"type": "rt::hitables::sphere", "args": {
                    "center": {"x": -1100.0, "y": 0.0, "z": 0.0},
                    "radious": 500.0,
                    "material": {
                        "type": "rt::materials::metal",
                        "args": {
                            "albedo": {"r": 0.9, "g": 0.9, "b": 0.9},
                            "fuzziness": 0.0
                        }
                    }
                }},
                {"type": "rt::hitables::sphere", "args": {
                    "center": {"x": 0.0, "y": 0.0, "z": 0.0},
                    "radious": 500.0,
                    "material": {
                        "type": "rt::materials::metal",
                        "args": {
                            "albedo": {"r": 0.9, "g": 0.9, "b": 0.9},
                            "fuzziness": 0.3
                        }
                    }
                }},
                {"type": "rt::hitables::sphere", "args": {
                    "center": {"x": 1100.0, "y": 0.0, "z": 0.0},
                    "radious": 500.0,
                    "material": {
                        "type": "rt::materials::metal",
                        "args": {
                            "albedo": {"r": 0.9, "g": 0.9, "b": 0.9},
                            "fuzziness": 0.0
                        }
                    }
                }},
                {"type": "rt::hitables::sphere", "args": {
                    "center": {"x": 0.0, "y": -20500.0, "z": 0.0},
                    "radious": 20000.0,
                    "material": {
                        "type": "rt::materials::lambertian",
                        "args": {
                            "albedo": {"r": 1.0, "g": 1.0, "b": 0.5}
                        }
                    }
                }}
            ]
        }
    }
}
```

This is the (current) `raytracer` kernel:

``` cpp
color trace(const ray& ray, const hitable& world, std::size_t depth_left)
{
    rt::hit_record hit_record;

    if(world.hit(ray, 0.0f, FLT_MAX, hit_record))
    {
        rt::ray     scattered;
        vector      attenuation;
        const auto* material = hit_record.material;

        if(depth_left > 0 &&
           material->scatter(ray, hit_record, attenuation, scattered))
        {
            return attenuation * trace(scattered, world, depth_left - 1);
        }
        else
        {
            return color::rgb(0.0f, 0.0f, 0.0f);
        }
    }
    else
    {
        vector unit_direction = ray.direction().normalized();

        const float t = 0.5f * (unit_direction.y + 1.0f);

        return (1.0f - t) * color::rgb(1.0f, 1.0f, 1.0f) +
               t * color::rgb(0.5f, 0.7f, 1.0f);
    }
}

void kernel(
    const float             x,
    const float             y,
    const kernel_constants& constants,
    color&                  pixel)
{
    ray eye_to_pixel = constants.scene.camera.ray(x, y);

    pixel = trace(eye_to_pixel, constants.scene, constants.iterations);
}
```

The renderer running on my Ryzen 1700X:

``` shell
$ ./bin/raytracer -c ../settings.json                                                                      0,0156s
[info] loading config file ../settings.json ...
[info] runtime settings:
[info]  - worker threads: all (16)
[info]  - output file: output.ppm
[info]  - max allocated jobs per thread:
[info]     thread 0: 100% (8847360 job(s))
[info]     thread 1: 0% (0 job(s))
[info]     thread 2: 0% (0 job(s))
[info]     thread 3: 0% (0 job(s))
[info]     thread 4: 0% (0 job(s))
[info]     thread 5: 0% (0 job(s))
[info]     thread 6: 0% (0 job(s))
[info]     thread 7: 0% (0 job(s))
[info]     thread 8: 0% (0 job(s))
[info]     thread 9: 0% (0 job(s))
[info]     thread 10: 0% (0 job(s))
[info]     thread 11: 0% (0 job(s))
[info]     thread 12: 0% (0 job(s))
[info]     thread 13: 0% (0 job(s))
[info]     thread 14: 0% (0 job(s))
[info]     thread 15: 0% (0 job(s))
[info] constants:
[info]  - iterations: 100
[info]  - samples per pixel: 100
[info]  - screen width: 4096 pixels
[info]  - screen height: 2160 pixels
[info]  - aspect ratio: 1.8963
[info]  - scene:
[info]    - camera:
[info]       - position: (0, 800, 5000)
[info]       - look at: (0, 0, 1000)
[info]       - up: (1, 0, 0)
[info]       - viewport width: 4096
[info]       - viewport height: 2160
[info]       - field of view: 1
[info]    - objects: 4 object(s)
[info]      - rt::sphere{center: (-1100, 0, 0), radious: 500, material: rt::materials::metal{albedo: (0.9, 0.9, 0.9), fuzziness: 0}}
[info]      - rt::sphere{center: (0, 0, 0), radious: 500, material: rt::materials::metal{albedo: (0.9, 0.9, 0.9), fuzziness: 0.3}}
[info]      - rt::sphere{center: (1100, 0, 0), radious: 500, material: rt::materials::metal{albedo: (0.9, 0.9, 0.9), fuzziness: 0}}
[info]      - rt::sphere{center: (0, -20500, 0), radious: 20000, material: rt::materials::lambertian{albedo: (1, 1, 0.5)}}
[info] 
[info] Starting...
[info] elapsed time: 39436 ms
[info] Done. Saving rendering to file...
```

And the resulting image:

![](/img/raytracer.png)

I will update the post with new images while completing the rest of the chapters of the books.
