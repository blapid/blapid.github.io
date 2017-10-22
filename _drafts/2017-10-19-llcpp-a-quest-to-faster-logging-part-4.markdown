---
title:  "llcpp - A quest to faster C++ logging (Part 4 - Afterword)"
date:   2017-10-22 12:56:15 +0300
categories: cpp
---

_This is part 4 of the series of post on llcpp, if you've just started here, I suggest you see the [intro post] for some context._

In this post I'd like to share extra thoughts I've had that after completing the series or that I could not fit into another post.

# The Future Of `llcpp`
I started this framework because I was curious about how binary logging would score in the benchmarks. After some continued hacking it actually got to perform much better than I had anticipated, so I decided to write this series of post to get this idea to a wider audience. While I'll keep using this framework for my personal projects, I'm interested in receiving feedback from the C++ community. I'd love to see objective benchmarks that compare it to the other frameworks and to continue to improve it according to the feedback.

Here's a quick list of areas that still need work:
- Documentation! Examples! Tutorials!
- A branch that uses "python style" (braces) instead of "printf style". It could be interesting to make a format parser that can parse the [fmt syntax][fmt-syntax] and it shouldn't be too difficult. One of the advantages of the `fmt` syntax is that it will be easier to add types and modifiers without having to account for all of the other types you're using. I haven't thought it through yet, but it should also be possible to implement positional arguments and named arguments too, but it would require some deep changes to the way argument serialization happens.
- Auxiliar features: Log rotation is a very important feature that is currently missing. Other features that are available on the rest of the leading frameworks should be considered too.
- Async log: While the benchmarks show that the framework is doing just fine without one, implementing an async queue and a worker could make this framework perform even better.
- Improve the parsing script.
- Make the binary log format more robust?

# Some More Thoughts
- C++ meta programming and compile time magic has a lot of potential. Unfortunately, many things are still very difficult to achieve; even worse, its **very** difficult to test and review. If you're interested in what's coming in the future, check [this][meta-cppcon] out. Oh man, I wish I had those tools when making this framework!
- Benchmarking worst case performance is very tricky. I should probably dig deeper into the issue and try to come up with better results.
- When trying to solve a problem, or improving a given solution, try to look for those hidden assumptions/constrains. You may find that relaxing them will result in something interesting. And if it didn't, it's a nice thought exercise regardless! :)

# Fin
Thanks for reading! I'd love to hear your feedback :) 

[meta-cppcon]: https://www.youtube.com/watch?v=4AfRAVcThyA
[fmt-syntax]: http://fmtlib.net/4.0.0/syntax.html#formatspec