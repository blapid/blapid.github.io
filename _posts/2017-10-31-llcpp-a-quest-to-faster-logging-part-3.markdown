---
title:  "llcpp - A quest to faster C++ logging (Part 3 - Benchmarks)"
date:   2017-10-31 12:56:15 +0300
categories: cpp
comments: true
---

_This is part 3 of the series of post on llcpp, if you've just started here, I suggest you see the [intro post][intro] for some context._

# Benchmarks, what are we measuring?
As mentioned in the previous posts, the main selling point of most of the logging frameworks I've considered is performance. From their repositories, we can learn how they benchmark their performance and hopefully understand the consequences for using each candidate. The first step is to decide a test case, which involves the following variables:
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

# My Benchmark, Results and Comparison
While going over the benchmarks in the candidate frameworks, the one implemented on NanoLog was my favorite (you can find it [here][nano-bench]). I liked it because it compared the most frameworks (NanoLog, spdlog, g3log and reckless) and showed the most information on call site latency (50th, 75th, 90th, 99th, and 99.9th pctile, worst case and average). Furthermore, it benchmark four different work loads by running the benchmark with 1, 2, 3 and 4 threads at a time. There were a couple of things missing, which I've added for my benchmarks:
- Running them with `time` to measure the total CPU time.
- Adding [MAL log][mal-gh]. 
- (Obviously) Adding llcpp.

One important thing to note about MAL log is that it prints the log's timestamp and not human-readable date/time. This means it doesn't have to call `localtime` or `gmtime` which reduces some overhead. If we're already on the subject of time-formatting, note that: NanoLog uses `gmtime` (faster version, no tz conversion), spdlog lets you choose between the two and g3log uses `localtime`. In llcpp I've implemented two prefixes: `time_format_prefix` which lets you choose between `gmtime` and `localtime` and `nanosec_time_prefix` which outputs a nanosec timestamp. In my comparison I've used llcpp with `time_format_prefix` (`localtime` mode) and with `nanosec_time_prefix`, so you can get a sense of both options.

When planning my benchmarking I wanted to see how each framework handles the NanoLog benchmark for two different filesystem scenarios: `/dev/shm/` (a tmpfs. would rather use `/dev/null` but not all frameworks let us do that), and `/tmp` (which, in Ubuntu, is mapped to your disk; in my case an SSD). I did this for two reasons: I wanted to get a better sense of how much work each framework is doing that is not filesystem related and I wanted to see whether that work actually matters when the I/O is the bottleneck. I compiled the benchmark with clang 5.0.1 and `--std=c++1z -flto -O3` flags. Unfortunately, I could not get reckless to compile. Time is in **nanoseconds**, unless stated otherwise. I'm using NanoLog's guaranteed logger.

#### 1 Thread Logging To `/tmp`

|Logger|     50th|     75th|     90th|     99th|   99.9th|    Worst|  Average |       Total Time (seconds)|
|:----:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:---------------:|
|llcpp (timestamp) |  165|      171|      211|     2058|     6313|   149432|230.360544| 0.42 |
|llcpp (date/time) |  184|      195|      216|     2153|     3718|  1099523|260.166040| 0.53 |
|mini-async-logger |       286|      335|      361|     1348|     2884|   6204357|324.418935| 1.32 |
|NanoLog |  211|      248|      291|     1099|     3809| 2493347|798.895457| 3.01 |
|spdlog | 729|      751|      780|      973|     3277|    1057879|744.825963| 0.95 |
|g3log |  2300|     2819|     3230|     5603|    28176|   2249642|2490.365190| 3.86 |

#### 4 Threads Logging To `/tmp`

|Logger|     50th|     75th|     90th|     99th|   99.9th|    Worst|  Average |       Total Time (seconds)|
|:----:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:---------------:|
| llcpp (timestamp) | 173|      191|      242|     7501|    36031|  2668978|458.101174| 1.61 |
|llcpp (date/time) | 220|      227|      270|    15515|    46522|  2162221|567.410498| 1.69 |
|mini-async-logger | 414|      581|      863|     2872|    24384|   5858754|612.069366| 5.52 |
|NanoLog |       338|      404|     1120|     1859|    24906| 1599487|845.320061| 12.81 |
|spdlog |      1100|     1193|     1238|     1373|    14022|  1312104|1032.874598| 1.91 |
|g3log |      1904|     2452|     3171|     9600|    37283|  6174296|2428.927793| 16.29 |

#### 1 Thread Logging To `/dev/shm`

|Logger|     50th|     75th|     90th|     99th|   99.9th|    Worst|  Average |       Total Time (seconds)|
|:----:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:---------------:|
|llcpp (timestamp) | 163|      168|      209|     1262|     2357|   153668|200.760008| 0.38 |
|llcpp (date/time) |    224|      234|      287|     1409|     3064|   166864|277.915158| 0.47 |
|mini-async-logger | 330|      418|      465|     1389|     4967|   115102|368.982075| 1.34 |
|NanoLog |  211|      234|      274|     1077|     1938|   296433|259.825371| 2.2 |
|spdlog | 747|      776|      882|     1081|     9157|    156415|787.803761| 1.04 |
|g3log | 2436|     2883|     3318|     5120|    28752|   240359|2590.587821| 2.84 |


#### 4 Threads Logging To `/dev/shm`

|Logger|     50th|     75th|     90th|     99th|   99.9th|    Worst|  Average |       Total Time (seconds)|
|:----:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:-------:|:---------------:|
|llcpp (timestamp) | 169|      173|      209|     2626|    31151|  6346309|340.374203| 1.01 |
|llcpp (date/time) |       346|      705|     1183|     3747|    25537|  6118893|709.675622| 1.37 |
|mini-async-logger |       434|      695|      978|     2528|    20945|  6228416|670.505850| 5.37 |
|NanoLog |       350|      410|     1048|     1806|    20061| 10900055|741.737527| 8.86 |
|spdlog |  870|     1160|     1217|     1392|    12203|  3034379|983.986584| 1.87 |
|g3log |     1967|     2674|     3273|     9856|    36719|   277528|2488.065324| 11.14 |

#### Worst Case Latency
I feel that it's important to discuss the worst case statistic. Measuring the worst case latency for log calls could be very important for many applications. However, it's a very tricky thing to measure. When I ran these benchmarks, I could not get a consistent worst case performance number. Each framework have showed between 10,000 ns and 1,000,000 ns in different runs. The numbers I present above are the worst case over several benchmark tests, but I'm not sure they mean that much.

I've thought the huge variance could occur due to some hardware or filesystem state which I can't control, so I tried benchmarking `/dev/shm` and `/dev/null`. Unfortunately, that didn't decrease the variance either! The next thing I thought of is OS preemption, which could account for the variance when the system is under load (which it is). This has led me to try running the tests with Linux's FIFO scheduler; I've set the process's priority to 99 and `sched_rt_runtime_us` to `990000` (`-1` has crashed the system). To my surprise, all of the frameworks, except llcpp, took *way* longer to complete that with the regular scheduler. I did get smaller variance of the worst case latencies for llcpp, but without anything to compare it to, it's probably counter-productive to show it.

I have to admit that I'm a very puzzled over this issue. If you think I've missed anything or have some thoughts about this - please let me know!

#### Some Thoughts On The Results
First of all, I'd like to encourage you to make your own benchmarks. My tests (and any test by a framework author) are subjective. Also, regardless of their objectivity, they may not even measure the things you care about or the scenarios your application faces.

I was actually very surprised to see how well llcpp managed to perform. Without async support I was expecting to see worse performance across the call latency board. In terms of total CPU time, I thought I'd see a difference, but I wasn't expecting such a dramatic decrease. In this sense, spdlog is doing a very good job as well. 

While llcpp, NanoLog and MAL are doing very well in the mid-upper percentiles, the 99th and 99.9th percentile aren't very promising for any candidate. I feel like the issues of the worst case latency measurements appear here as well, just more subtle.

# Afterword
In the next post I'd like to share some thoughts on the future ides for llcpp and what I've learned from this experiment.

You can read it here: [Part 4][part4].

[nano-bench]: https://github.com/Iyengar111/NanoLog/blob/master/nano_vs_spdlog_vs_g3log_vs_reckless.cpp
[mal-gh]: https://github.com/RafaGago/mini-async-log
[part4]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-part-4 %}
[intro]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-intro %}