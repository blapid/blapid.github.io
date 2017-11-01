---
title:  "llcpp - A quest to faster C++ logging (Intro)"
date:   2017-10-31 12:56:15 +0300
categories: cpp
comments: true
---

Several weeks ago a friend and I were discussing C++ logging frameworks. He had some issues with his company's home-made logger, mainly regarding call latencies, and wanted to switch to something better. After some googling around we've found a couple of interesting candidates: [g3log][g3log-github], [NanoLog][NanoLog-github], [MAL][mal-github] and [spdlog][spdlog-github]. All four options aim to be "fast" and use similar approaches to achieve that. In the following posts I'd like to talk about what logging frameworks do, how do they define and measure what "fast" is and showcase a different approach to C++ logging which I've been messing around with.

# TL;DR
If the requirement of outputting a textual log from the C++ application is relaxed, we can create a faster logging framework. Enter `llcpp` (literal logging for c++). This framework uses C++17 template (and constexpr) meta-programming magic to reduce the number of allocations needed to zero and eliminates the need to format the data to human readable form. Later, a simple script can be used to transform the log into human readable form. Oh, and we get type safety for free.

Intrigued? skeptical? excited? angry? Read on!
_(Prefer reading code? [Go ahead][llcpp-gh]!)_

# Post Index
- [Part 1][part1]: Why are we logging and what modern logging frameworks in C++ do
- [Part 2][part2]: llcpp - A different approach
- [Part 3][part3]: Benchmarks - discussion and comparison
- [Part 4][part4]: Afterword

[llcpp-gh]: https://github.com/blapid/llcpp
[g3log-github]: https://github.com/KjellKod/g3log
[NanoLog-github]: https://github.com/Iyengar111/NanoLog
[spdlog-github]: https://github.com/gabime/spdlog
[mal-github]: https://github.com/RafaGago/mini-async-log
[duck-debug]: https://rubberduckdebugging.com/
[kjellkods-async-loggers]: https://kjellkod.wordpress.com/2011/11/17/kjellkods-g2log-vs-googles-glog-are-asynchronous-loggers-taking-over/
[g3log-vs-spdlog]: https://kjellkod.wordpress.com/2015/06/30/the-worlds-fastest-logger-vs-g3log/
[g3log-vs-spdlog-cont]:https://github.com/gabime/spdlog/issues/293
[part1]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-part-1 %}
[part2]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-part-2 %}
[part3]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-part-3 %}
[part4]: {% post_url 2017-10-31-llcpp-a-quest-to-faster-logging-part-4 %}