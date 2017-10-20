---
layout: post
title:  "LLCPP - A quest to faster C++ logging (Part 1)"
date:   2017-10-19 12:56:15 +0300
categories: cpp
---

Several weeks ago a friend and I were discussing C++ logging frameworks. He had some issues with his company's home-made logger, mainly regarding call latencies, and wanted to switch to something better. After some googling around we've found a couple of interesting candidates: [g3log][g3log-github], [NanoLog][NanoLog-github], [MAL][mal-github] and [spdlog][spdlog-github]. All four options aim to be "fast" and use similar approaches to achieve that. In this post I'd like to talk about what logging frameworks do, how do they define and measure what "fast" is and showcase a different approach to C++ logging which I've been messing around with.

# Logging, what is it good for? Absolutely...
Nothing? Nah, it's really useful, right? Chances are you started logging when you wrote one of your first computer programs and it failed miserably. You've worked on that piece of code for a while, going line by line to make sure they make sense. Heck, you might've (unknowingly) had your first [duck debug][duck-debug] moments trying to fix it. And then you had an epiphany: "Lets use that Hello World! thing to see whats going on!". I vaguely remember mine, and it probably looked something like this:
```c++
...
int foo(char *str, size_t len) {
  for (int i = 0; i < len; i++) {
    if (str[i] != SOME_VALUE) {
      printf("Hello world!\n");
      // do something
    } else {
      printf("Hello world!!!!!!!!\n");
      // fail miserably
    }
  }
}
...
```
It might seem kind of silly now, but it is an important lesson for a new programmer: It's not all black-box after you leave your editor, you can get information from your code while its running! As you became a more experienced developer, you've probably started using a debugger for these kind of tasks. Yet I'm sure most developers still use printf-debugging from time to time.

But logging isn't just for finding bugs, collecting information from a running program has many uses:
- *Output*: Sometimes, the logs are the intended purpose of your program. For example: keeping a list of the times users have logged in.
- *Additional crash report information*: For those times where a simple crash report is just not enough, logs are used to help you figure out how we got in this sticky situation in the first place.
- *Monitoring*: Many applications use logs to convey their status at certain points of time, this information is then collected and checked to make sure everything is running smoothly. If too many errors appear, you can get notified without having to check your application all the time.
- And others which I've probably missed...

Okay, enough meta rambling...

# What do C++ logging frameworks do?
In a broad sense, logging frameworks are meant to be a high-level API to make it easy for the programmer to log information. "High-level" means that it abstracts most of the detail about *how* it's done, and "easy" (at least my interpretation of it) means that it won't take too much brain-power to use (we need that brain-power to be focused on the actual code). 

The "easy" part is very subjective and it seems like the C++ community has trouble deciding between a couple of approaches:
- *printf-style*: The good-ol `"Hello world!!!! %s %d %f etc...", "please work this time", 42, 1.618`
- *iostream-style*: `"we" << "want" << "chevron" << "hell!"`
- *python-style*: `"{} can be {} with {} curly braces", "everything", "solved", "more"`

I have to admit, they're all pretty "easy" and are all valid approaches. Personally, I'm kinda leaning towards the printf-style: I've grown used to its format and quirks, I don't like the weird implicit-state changes that may happen with the iostream-style and I prefer the printf-style explicit-ness to the python-style (default) implicit-ness.

The abstraction part is the heart of this post. It's both the advantage and disadvantage of (logging) frameworks everywhere. The advantage is quite clear, there won't be any "easy" use. "hiding of detail", "reasonable hack-ability" and many other good things without abstraction. The disadvantage is that it makes us *ignore* the fact that actual code is executed there, which is dangerous. Most developers just slap in more logging calls without being aware of the side-effects of those calls, which range from long periods of the program being blocked on I/O to the program crashing due to format errors.

Indeed, as is visible from the description of the candidates I mentioned above, logging frameworks try to minimize those side effects. Every single one of them (and many others I've encountered while searching) attempt to cause the least amount of run-time side effects. This is important for a pretty obvious reason: People who use C++ care about their code's performance. Some of them do it out of pure perfectionism, others work on high-load applications that can't waste much time on side-goals like logging and some work on real-time applications which must have strong guarantees about run-time performance.

It is important to note a trend that showed up on all of the candidates mentioned above: They implement an asynchronous logging paradigm. This means that when a log function is called, the framework attempts to do the minimal amount of work on the calling thread, and delegates the rest of the work to a dedicated "log-worker" thread. See [this][kjellkods-async-loggers] post from the author of g3log for some more info. Indeed, this usually decreases the run-time side effects, but with a fine print. The issue is: This does not really reduce the total amount of work the framework does, it just queues it for later. Sure, if your application has infrequent bursts of high load with enough time to clear the queue, this approach makes a lot of sense. However, if your application is constantly under load, the fast async logger they promised isn't any better than a synchronous one. Not to mention, all that extra work maintaining the data structures and another thread add a lot of complexity to these frameworks. That being said, I have to admit that this approach is probably the "right" way to go for most applications. I would only suggest that you try to understand the details behind the abstractions you use to make sure they really fit your application goals.

After going through the code of the candidates I've mentioned above, I'd like to break their behavior to the following categories:
- *API*: The C++ interface for calling the setup and log functions from the user's code
- *Serializing the information*: Turning this call `DEBUG("Hello %s!", "world")` (or the chevron equivalent) to the string "Hello world!".
- *Async support*: Implementation of a queues or other dispatch mechanism and worker-thread to collect log messages and pass them to the IO code.
- *I/O*: Sending the serialized string to some output: stdout, file, network, etc.
- *Other stuff*: Some frameworks implement some auxiliary features like: file rotation, conditional logging, crash handlers and others. I put these in their own category because, In my opinion, they're "nice-to-have" features and not part of the essentials of a logging framework.

#### API
This part usually holds the obvious debug/info/warn/... functions, some helper macros, `operator<<` implementation and initialization functions. To me, it was kind of interesting to see that none of the non-chevron approaches tried to verify the function-arguments based on the format string. More on my attempt at this later on. Some frameworks add extra information on top of the format you provide in your call. For example: File path and line of the log call, date and time of the log call and more.

#### Serializing the information
Most approaches seemed to realize that this part can cause a performance hit and therefore either implement/borrow a quick formatting implementation. Some frameworks even try to delay the formatting to the worker thread, this usually requires some extra argument serialization along the way and has some quirks with parameters passed by reference or pointer (i.e. strings).

#### Async support
In order to make the log calls asynchronous, the frameworks pass the serialized log messages to a worker thread that handles the rest of the serialization and writes the string to a selected IO method. The candidates use different methods of dispatching the logs: some use lock-free queues and some use queues that may block, some may drop log messages when the load is high and some guarantee that all messages will be written. A pretty good discussion (with some benchmark spoilers) can be found [here][g3log-vs-spdlog] and [here][g3log-vs-spdlog-cont] with inputs from the authors of our candidates.

#### I/O
Some candidates allow you to initialize them with various of IO "sinks" which allow you to write your log to different receivers (i.e. Android's logcat, syslog, file system, etc.) while others only let you specify a filename to write to. Other than that, the worker thread pretty much passes the data directly to a libc/libc++ library call - usually `std::ofstream`. I was kind of surprised that none of them implemented some kind of `select(2)` code to handle multiple sinks.

# Benchmarks, what are we measuring?
As mentioned earlier, the main selling point of most of the logging frameworks I've considered is performance. From the repositories of the candidates above, we can learn how they benchmark their performance and hopefully understand the consequences for using each candidate. The first step is to decide a test case, which involves the following variables:
- What logging framework are we benchmarking?
- Which features of the framework are we benchmarking?
- What underlying abstractions are we building upon?
- What exactly are we measuring?

The first point is pretty straight forward: pick a group of frameworks you wish to compare. Usually this includes the author's own framework. Unfortunately I have not found a single (up to date) benchmark authored by anyone that is not an author of a logging framework. This creates a very competitive atmosphere and discussions, while this is a positive thing when it leads to innovation, it often leads to biased discussion at best - and forgetting the actual customers at worst. I admit, by entering this discussion with my own framework, I'm now a part of the problem too. But I hope someone that is reading this will be encouraged to create a neutral comparison that will drive the framework authors forward.

After we chose our candidates, we decide which features we are benchmarking. This step is unfortunately, often overlooked upon, because it is crucial to understand the difference in features between the frameworks we're comparing. One key example is a benchmark I saw, which compares two frameworks that handle the log's timestamp differently: One converts it to human-readable format (with `localtime(3)`) and the other leaves it as an integer. This difference may seem subtle, but the extra work done by `localtime(3)` can be the reason for not-so-subtle differences in performance. This may lead to questionable comparisons - obviously, the author of the `localtime(3)`-formatting-framework can remove it (or make it optional) very easily, which may invalidate the results. Another example is mixing asynchronous frameworks that allow passing of non-literal strings as arguments (which means they have to copy it) with those that don't. Users may pick the framework that works faster but only later realize that they're missing a feature that's essential for them.

As I've mentioned earlier, understanding the underlying abstraction for each framework and test case is extremely important for a benchmark. While it's very hard to catch every little side-effect, we should actively try to find the ones that have the biggest impact. My key example here revolves around choosing the filesystem which stores the log files. Some benchmarks write to your current working directory, some write to `/tmp` (which may be the same filesystem as your CWD) and others write to `/dev/null`. While it may be relevant to benchmark any or all of these options, you should try to understand how this decision affects each framework. Some may seem faster on `/dev/null` because they're optimized for a "happy flow" where I/O calls don't block that much but on actual real-life workloads may not be better.

After we have all of the above figured out, we need to decide what we are measuring. The benchmarks that were used by the framework's authors measured at least one of the following scenarios:
- For a given work-load (X total number of logs, Y producer threads, async/sync method of operation), measure the total time to completion.
- For a given work-load, measure the average log call site latency.
- For a given work-load, measure the worst log call site latency.
- For a given work-load, measure a histogram of the log call site latency (and pick the higher percentiles).

While there is a strong opinion for measuring only worst case latency, it really depends on the requirements and the design of the applications. Obviously, hard real time systems must make precise predictions on this property, but most of the systems don't have these constraints. In fact, most operating systems will work **against** this measurement because of their preemptive nature. I have tried getting around this preemtion issue by playing around with the Linux Kernel scheduler parameters (`sched_setscheduler(..., SCHED_FIFO, ...)`) and by measuring with `clock_gettime(CLOCK_THREAD_CPUTIME_ID, ...)` but both showed inconsistent results.

An important thing to remember about call site latency benchmarks is that they do not reflect the **total CPU time** the framework has used. In the case of async loggers, they show relatively low call site latency, but if you measure the total time to completion (that is, including actually writing the log) you don't necessarily get better performance from them. Therefore, if your issues are overall system performance (i.e. high cpu loads over extended periods of time) and not specific call site latency (i.e. quick bursts of high computational load), you should give proper weight to the first scenario over the last three.

# A different approach...
I started the post by promising to show case a different approach, which I've been working on for the past couple of weeks. However, I think this post is getting too big - which means I'll dedicate the next post introducing my logging framework and discuss how it doing in the benchmarks.

You can read it here: [Part 2][part2].

[g3log-github]: https://github.com/KjellKod/g3log
[NanoLog-github]: https://github.com/Iyengar111/NanoLog
[spdlog-github]: https://github.com/gabime/spdlog
[mal-github]: https://github.com/RafaGago/mini-async-log
[duck-debug]: https://rubberduckdebugging.com/
[kjellkods-async-loggers]: https://kjellkod.wordpress.com/2011/11/17/kjellkods-g2log-vs-googles-glog-are-asynchronous-loggers-taking-over/
[g3log-vs-spdlog]: https://kjellkod.wordpress.com/2015/06/30/the-worlds-fastest-logger-vs-g3log/
[g3log-vs-spdlog-cont]:https://github.com/gabime/spdlog/issues/293
