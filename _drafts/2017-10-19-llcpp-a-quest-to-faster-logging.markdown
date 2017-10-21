---
title:  "LLCPP - A quest to faster C++ logging (Part 1)"
date:   2017-10-19 12:56:15 +0300
categories: cpp
---

_This is the first part of the series of post on LLCPP, to put things in perspective, make sure you read the [intro post]._

# Logging, what is it good for? Absolutely...
Nothing? Nah, it's really useful, right? Chances are you started logging when you wrote one of your first computer programs and it failed miserably. You've worked on that piece of code for a while, going line by line to make sure they make sense. Heck, you might've (unknowingly) had your first [duck debug][duck-debug] moments trying to fix it. And then you had an epiphany: "lets use that Hello World! thing to see whats going on!". I vaguely remember mine, and it probably looked something like this:
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
It might seem kind of silly now, but it is an important lesson for a new programmer: it's not all black-box after you leave your editor, you can get information from your code while its running! As you became a more experienced developer, you've probably started using a debugger for these kind of tasks. Yet I'm sure most developers still use printf-debugging from time to time.

But logging isn't just for finding bugs, collecting information from a running program has many uses:
- *Output*: Sometimes, the logs are the intended purpose of your program. For example: keeping a list of the times users have logged in.
- *Additional crash report information*: For those times where a simple crash report is just not enough, logs are used to help you figure out how we got in this sticky situation in the first place.
- *Monitoring*: Many applications use logs to convey their status at certain points of time, this information is then collected and checked to make sure everything is running smoothly. If too many errors appear, you can get notified without having to check your application all the time.
- And others which I've probably missed...

Okay, enough meta rambling...

# What do C++ logging frameworks do?
In a broad sense, logging frameworks are meant to be a high-level API to make it easy for the programmer to log information. "High-level" means that it abstracts most of the detail about *how* it's done, and "easy" (at least my interpretation of it) means that it won't take too much brain-power to use (we need that brain-power to be focused on the actual code). 

The "easy" part is very subjective and it seems like the C++ community has trouble deciding between a couple of approaches:
- *printf-style*: The good-ol'  `"Hello world!!!! %s %d %f etc...", "please work this time", 42, 1.618`
- *iostream-style*: `"we" << "want" << "chevron" << "hell!"`
- *python-style*: `"{} can be {} with {} curly braces", "everything", "solved", "more"`

I have to admit, they're all pretty "easy" and are all valid approaches. Personally, I'm kinda leaning towards the printf-style: I've grown used to its format and quirks, I don't like the weird implicit-state changes that may happen with the iostream-style, and I prefer the printf-style explicitness to the python-style (default) implicitness.

The abstraction part is the heart of this post. It's both the advantage and disadvantage of (logging) frameworks everywhere. The advantage is quite clear, there won't be any "easy" use, "hiding of detail", "reasonable hack-ability" and many other good things without abstraction. The disadvantage is that it makes us *ignore* the fact that actual code is executed there, which is dangerous. Most developers just slap in more logging calls without being aware of the side-effects of those calls, which range from long periods of the program being blocked on I/O to the program crashing due to format errors.

Indeed, as is visible from the description of the candidates I mentioned above, logging frameworks try to minimize those side effects. Every single one of them (and many others I've encountered while searching) attempt to cause the least amount of run-time side effects. This is important for a pretty obvious reason: people who use C++ care about their code's performance. Some of them do it out of pure perfectionism, others work on high-load applications that can't waste much time on side-goals like logging, and some work on real-time applications which must have strong guarantees about run-time performance.

It is important to note a trend that showed up on all of the frameworks mentioned above: they implement an asynchronous logging paradigm. This means that when a log function is called, the framework attempts to do the minimal amount of work on the calling thread, and delegates the rest of the work to a dedicated "log-worker" thread. See [this][kjellkods-async-loggers] post from the author of g3log for some more info. True, this usually decreases the run-time side effects, but with a fine print. The issue is: this does not really reduce the total amount of work the framework does, it just queues it for later. Sure, if your application has infrequent bursts of high load with enough time to clear the queue, this approach makes a lot of sense. However, if your application is constantly under load, the fast async logger they promised isn't any better than a synchronous one. Not to mention, all that extra work maintaining the data structures and another thread add a lot of complexity to these frameworks. That being said, I have to admit that this approach is probably the "right" way to go for most applications. I would only suggest that you try to understand the details behind the abstractions you use to make sure they really fit your application goals.

After going through the code of the candidates I've mentioned above, I'd like to break their behavior to the following categories:
- *API*: The C++ interface for calling the setup and log functions from the user's code
- *Serializing the information*: Turning this call `DEBUG("Hello %s!", "world")` (or the chevron equivalent) to the string "Hello world!".
- *Async support*: Implementation of a queue or other dispatch mechanism and worker-thread to collect log messages and pass them to the I/O code.
- *I/O*: Sending the serialized string to some output: stdout, file, network, etc.
- *Other stuff*: Some frameworks implement some auxiliary features like: file rotation, conditional logging, crash handlers and others. I put these in their own category because, In my opinion, they're "nice-to-have" features and not part of the essentials of a logging framework.

#### API
This part usually holds the obvious debug/info/warn/... functions, some helper macros, `operator<<` implementation and initialization functions. To me, it was kind of interesting to see that none of the non-chevron approaches tried to verify the function-arguments based on the format string. More on my attempt at this later on. Some frameworks add extra information on top of the format you provide in your call. For example: File path and line of the log call, date and time of the log call and more.

#### Serializing the information
Most approaches seemed to realize that this part can cause a performance hit and therefore either implement/borrow a quick formatting implementation. Some frameworks even try to delay the formatting to the worker thread; this usually requires some extra argument serialization along the way and has some quirks with parameters passed by reference or pointer (i.e. strings).

#### Async support
In order to make the log calls asynchronous, the frameworks pass the serialized log messages to a worker thread that handles the rest of the serialization and writes the string to a selected I/O method. The candidates use different methods of dispatching the logs: some use lock-free queues and some use queues that may block, some may drop log messages when the load is high and some guarantee that all messages will be written. A pretty good discussion (with some benchmark spoilers) can be found [here][g3log-vs-spdlog] and [here][g3log-vs-spdlog-cont] with inputs from the authors of our candidates.

#### I/O
Some candidates allow you to initialize them with various of I/O "sinks" which allow you to write your log to different receivers (i.e. file system, syslog, android's logcat, etc.) while others only let you specify a filename to write to. Other than that, the worker thread pretty much passes the data directly to a libc/libc++ library call - usually `std::ofstream`. I was kind of surprised that none of them implemented some kind of `select(2)` code to handle multiple sinks.


# A different approach...
While going over the repositories of the different frameworks, I've had a strange itch in the back of my head. It took me a couple of days, but then I realized: they're all conforming to a constraint that is taken for granted. Which constraint, you ask? The one that dictates that the output log file must be human-readable. What happens when we ignore this constraint? That will be the subject of the next post.

You can read it here: [Part 2][part2].

[g3log-github]: https://github.com/KjellKod/g3log
[NanoLog-github]: https://github.com/Iyengar111/NanoLog
[spdlog-github]: https://github.com/gabime/spdlog
[mal-github]: https://github.com/RafaGago/mini-async-log
[duck-debug]: https://rubberduckdebugging.com/
[kjellkods-async-loggers]: https://kjellkod.wordpress.com/2011/11/17/kjellkods-g2log-vs-googles-glog-are-asynchronous-loggers-taking-over/
[g3log-vs-spdlog]: https://kjellkod.wordpress.com/2015/06/30/the-worlds-fastest-logger-vs-g3log/
[g3log-vs-spdlog-cont]:https://github.com/gabime/spdlog/issues/293
