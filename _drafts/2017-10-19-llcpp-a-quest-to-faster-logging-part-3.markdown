---
title:  "LLCPP - A quest to faster C++ logging (Part 3 - Benchmarks)"
date:   2017-10-19 12:56:15 +0300
categories: cpp
---

_This is part 3 of the series of post on LLCPP, if you've just started here, I suggest you see the [intro post] for some context._

# Benchmarks, what are we measuring?
As mentioned in the previous posts, the main selling point of most of the logging frameworks I've considered is performance. From the repositories of the candidates above, we can learn how they benchmark their performance and hopefully understand the consequences for using each candidate. The first step is to decide a test case, which involves the following variables:
- What logging framework are we benchmarking?
- Which features of the framework are we benchmarking?
- What underlying abstractions are we building upon?
- What exactly are we measuring?

The first point is pretty straight forward: pick a group of frameworks you wish to compare. Usually this includes the author's own framework. Unfortunately I have not found a single (up to date) benchmark authored by anyone that is not an author of a logging framework. This creates a very competitive atmosphere and discussions; and while this is a positive thing when it leads to innovation, it often leads to biased discussion at best - and forgetting the actual customers at worst. I admit, by entering this discussion with my own framework, I'm now a part of the problem too. But I hope someone that is reading this will be encouraged to create a neutral comparison that will drive the framework authors forward.

After we chose our candidates, we decide which features we are benchmarking. This step is unfortunately, often overlooked, because it is crucial to understand the difference in features between the frameworks we're comparing. One key example is a benchmark I saw, which compares two frameworks that handle the log's timestamp differently: one converts it to human-readable format (with `localtime(3)`) and the other leaves it as an integer. This difference may seem subtle, but the extra work done by `localtime(3)` can be the reason for not-so-subtle differences in performance. This may lead to questionable comparisons - obviously, the author of the `localtime(3)`-formatting-framework can remove it (or make it optional) very easily, which may invalidate the results. Another example is mixing asynchronous frameworks that allow passing of non-literal strings as arguments (which means they have to copy it) with those that don't. Users may pick the framework that works faster but only later realize that they're missing a feature that's essential for them.

As I've mentioned earlier, understanding the underlying abstraction for each framework and test case is extremely important for a benchmark. While it's very hard to catch every little side-effect, we should actively try to find the ones that have the biggest impact. My key example here revolves around choosing the filesystem which stores the log files. Some benchmarks write to your current working directory, some write to `/tmp` (which may be the same filesystem as your CWD) and others write to `/dev/null`. While it may be relevant to benchmark any or all of these options, you should try to understand how this decision affects each framework. Some may seem faster on `/dev/null` because they're optimized for a "happy flow" where I/O calls don't block that much but on actual real-life workloads may not be better.

After we have all of the above figured out, we need to decide what we are measuring. The benchmarks that were used by the framework's authors measured at least one of the following scenarios:
- For a given work-load (X total number of logs, Y producer threads, async/sync method of operation), measure the total time to completion.
- For a given work-load, measure the average log call site latency.
- For a given work-load, measure the worst log call site latency.
- For a given work-load, measure a histogram of the log call site latency (and pick the higher percentiles).

While there is a strong opinion for measuring only worst case latency, it really depends on the requirements and the design of the applications. Obviously, hard real time systems must make precise predictions on this property, but most of the systems don't have these constraints. In fact, most operating systems will work **against** this measurement because of their preemptive nature. I have tried getting around this preemtion issue by playing around with the Linux Kernel scheduler parameters (`sched_setscheduler(..., SCHED_FIFO, ...)`) and by measuring with `clock_gettime(CLOCK_THREAD_CPUTIME_ID, ...)` but both showed inconsistent results.

An important thing to remember about call site latency benchmarks is that they do not reflect the **total CPU time** the framework has used. In the case of async loggers, they show relatively low call site latency, but if you measure the total time to completion (that is, including actually writing the log) you don't necessarily get better performance from them. Therefore, if your issues are overall system performance (i.e. high CPU loads over extended periods of time) and not specific call site latency (i.e. quick bursts of high computational load), you should give proper weight to the first scenario over the last three.

# Results and Comparison

# Afterword
In the next post I'd like to share some thoughts on the future ides for LLCPP and what I've learned from this experiment.

You can read it here: [Part 4][part4].
