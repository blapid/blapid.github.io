---
title:  "LLCPP - A quest to faster C++ logging (Intro)"
date:   2017-10-19 12:56:15 +0300
categories: cpp
comments: true
---

Several weeks ago a friend and I were discussing C++ logging frameworks. He had some issues with his company's home-made logger, mainly regarding call latencies, and wanted to switch to something better. After some googling around we've found a couple of interesting candidates: [g3log][g3log-github], [NanoLog][NanoLog-github], [MAL][mal-github] and [spdlog][spdlog-github]. All four options aim to be "fast" and use similar approaches to achieve that. In the following posts I'd like to talk about what logging frameworks do, how do they define and measure what "fast" is and showcase a different approach to C++ logging which I've been messing around with.

# TL;DR
If the requirement of outputting a textual log from the C++ application is relaxed, we can create a faster logging framework. This framework uses C++17 template (and constexpr) meta-programming magic (which also adds type safety) to reduce the number of allocations needed to zero and eliminates the need to format the data to human readable form. Later, a simple script can be used to transform the log into human readable form.
Intrigued? skeptical? excited? angry? Read on!

# Post Index
- Part 1: Why are we logging and what modern logging frameworks in C++ do
- Part 2: LLCPP - A different approach
- Part 3: Benchmarks, discussion and comparison
- Part 4: Afterword

[g3log-github]: https://github.com/KjellKod/g3log
[NanoLog-github]: https://github.com/Iyengar111/NanoLog
[spdlog-github]: https://github.com/gabime/spdlog
[mal-github]: https://github.com/RafaGago/mini-async-log
[duck-debug]: https://rubberduckdebugging.com/
[kjellkods-async-loggers]: https://kjellkod.wordpress.com/2011/11/17/kjellkods-g2log-vs-googles-glog-are-asynchronous-loggers-taking-over/
[g3log-vs-spdlog]: https://kjellkod.wordpress.com/2015/06/30/the-worlds-fastest-logger-vs-g3log/
[g3log-vs-spdlog-cont]:https://github.com/gabime/spdlog/issues/293
