---
title:  "llcpp - A quest to faster C++ logging (Part 2 - llcpp)"
date:   2017-10-22 12:56:15 +0300
categories: cpp
---

_This is part 2 of the series of post on llcpp, if you've just started here, I suggest you see the [intro post][intro-post] for some context._

_This is a relatively technical post, if you're looking for quick how-to's, see the [github project page][llcpp-gh]._

# The Textual Output Constraint
In this post I'd like to discuss the hidden constraint that every logging framework conform to but no one acknowledges: textual output. I'm pretty sure that even bringing this up will get some people riled up. "What do you mean non-textual logs? What are logs good for if I can't read them?!" Obviously, they are not that helpful if you can't read them, but that's not the point. Remember, the point of logging is getting information out of your application; making it easy to consume doesn't have to burden it. If you think about it, in many modern applications, you don't even directly view the logs. Instead, they're pulled and aggregated from your servers, go through other applications (monitoring, auditing, etc.), stored in a certain way and displayed through a dedicated user interface. In this workflow, adding another step, which transforms the non-textual log to a textual log, is quite straightforward.

Basically, what I propose is designing the framework such that it wouldn't have to format the data during run-time; and hopefully, in the process, create a framework that performs better. The next step is to create another application/script which parses the non-textual log file and outputs a regular textual log that you'd expect from any other logging framework.

You may call this a *bluff*: "Formatting still happens *somewhere*, you aren't reducing the total CPU time too!". And you'd be almost right! The total CPU time isn't necessarily less; but while the previous approaches delay the work for the **same machine**, this approach delegates the work to **other machines!**. Then, if parsing is your bottleneck, just spin up more parsing nodes; this is, usually, way-easier than spinning up more application nodes (which may not even work in a distributed manner). Decoupling is a key ingredient in any software architecture, lets apply it to our logs too!

# Binary Log Format
The keen reader is probably asking: if you don't apply formatting, how do you extract argument information? The answer is: I serialize it in the quickest manner possible, while relying on the format string itself to represent the serialization structure. This binary format is pretty straightforward: write the null-terminated format string, then write the in-memory binary representation of each of the arguments. For arithmetic types (ints, floats, etc.), the size of each binary representation is derived from the format string (`%d` will be 4 bytes, `%lld` will be 8 bytes, etc.). For strings, we have two options: writing the length and then the string or passing information about the maximum length in the format string and then copying the minimum between the given maximum and the string's actual length.

Examples:
- `LOG("Hello world\n")` - would simply be written as the null terminated string, as you'd expect.
- `LOG("Hello world %d\n", 42)` - would write the null terminated string, followed by 4 bytes representing `42` (little endian).
- `LOG("Hello world %lld\n", 42)` - would write the null terminated string, followed by 8 bytes representing `42` (little endian).
- `LOG("Hello world %s\n", "42")` - would write the null terminated string, followed by 4 bytes representing `2` (the length of `"42"`) and 2 bytes with the actual string `"42"`
- `LOG("Hello world %8s\n", "42")` - would write the null terminated string, followed by 8 bytes, of which the first two would hold the string `"42"`. If the string given in the argument is bigger, it would be truncated.

I admit that this format is not too robust and parts of it could become corrupted if used wrong. Keep in mind that the format itself is merely used to prove that this approach is plausible, and could be hardened as needed.

# High-Level Design
I'd like to break down the design of my approach using the categories from the last post:
- *API*: This framework uses new meta-programming features from C++14 and C++17 to create, at compile time, a type safe API which looks like regular `printf` but outputs our binary format. I'm not completely sure, but I think it will be possible to port this approach to earlier C++ version; I'm sure, however, that it'll require quite a lot of work.
- *Serializing the information*: The format string and arguments will be serialized as described in the binary format above.
- *Async support*: Every log will be written synchronously. I would not like add the complexity of async logging right now. Interestingly, as you'll see in the 3rd part, this will not necessarily mean worse performance. In the future, this feature will be implemented.
- *I/O*: Logs will be written to an `std::FILE *`, such as a file on the filesystem or `stdout` and the likes.
- *Other stuff*: For now, there will be no auxiliary features such as log rotation and crash handling. Although they may be added later quite easily.

# Implementation Overview
The way I decided to implement the design was to parse the given format string at compile-time. This allows us to generate specific code to serialize each log line as quickly as possible (through template meta-programming). This serialization would use an "output interface" which lets it write the format parts and specify when a complete line format have been written.

In the following section I'd like to outline my implementation. This is not a complete explanation, but it should be enough to allow you to start exploring the code yourself.

#### Format String Parsing
First, we need a way to collect the format string in a way that allows us to implement a compile-time parser. The obvious approach is to use C++14 [user-defined-string-literals][user-defined-string-literals]:
```c++
template<typename CharT, CharT... Chars>
constexpr auto operator"" _log() noexcept
{
    using format_string_t = typename format_string<CharT, Chars...>;
    return format_string_t();
}
```
Using the following format_string type:
```c++
template<typename CharT, CharT... Chars>
struct format_string {
    //...
    using format_tuple = std::tuple<typename std::integral_constant<CharT, Chars>::type...>;
    inline static constexpr CharT  _chars[string_format::fmt_size()] = {Chars..., '\0'};
};
```

Means that the following expression `"hello world"_log` will return an instance of `format_string`, specialized with the characters of `"hello world"`. This specialization holds the format as a static constexpr character array - it makes sense because we don't need a copy of it for each log call - which means practically no work is done in runtime for this expression. We also convert the `Chars...` variadic template parameters to an `std::tuple<std::integral_constant<...>>` to make it easier to handle the format later.

Now, we need to implement the actual parsing. If you're new to C++ compile time string parsing, you may wish to read [Andrzej's posts][parsing-compile-time] which helped me a lot along the way.

The parser receives the format string tuple and starts going over the characters. In order to properly detect the different escape sequences, a state is kept and passed through the iterations. This state lets the parser know whether it is in an "escaped" state, which means a `'%'` character was encountered but not terminated, or in an "unescape" state, which means we're looking for the following `'%'`. The state also stores the index of the `'%'` character that cause the current escape (if in "escaped" state); this allows us to collect the start and end indexes of the escaped sequences.

Let's go over a simple parser implementation. It will not work for the entire `printf` specification; in fact, it would only work for a subset of the `'%d'` escapes. However, I hope it will give you a good sense of how we may create a complete parser. This parser will receive the information I described above, and will create an tuple of escape sequences that it extracted along the way. Each escape sequence will store its start and end indexes. Storing the start/end indexes will allow us to delegate checking further identifiers (think of `'%lld'`) to a later time and keep the code cleaner.
```c++
template<std::size_t BeginIdx, std::size_t EndIdx>
struct escape_sequence {
    static constexpr std::size_t begin = BeginIdx;
    static constexpr std::size_t end = EndIdx;
}

template<typename CharTuple, std::size_t CharTupleSize, std::size_t Idx, 
    bool IsEscaped, std::size_t EscapeIdx>
struct format_parser {
    ...
    using escape_sequences = std::tuple<>;
};
```
The implementation would check whether the current character is `'%'`
```c++
using cur_char = typename std::tuple_element<Idx, CharTuple>::type;
using is_escape_char = typename std::conditional_t<
    cur_char::value == '%',
    std::true_type,
    std::false_type
    >;
```
It would also check, whether the character is a terminator (only `'d'` for this example):
```c++
using is_int_terminator = typename std::conditional_t<
    cur_char::value == 'd',
    std::true_type,
    std::false_type
    >;
```
Now we have to handle the following matrix of options: escaped/not-escaped, encountered `'d'`/encountered `'%'`/encountered-something-else.
```c++
// If we're terminated, create a tuple with an escape_sequence, 
//   otherwise create an empty tuple
using current_escape_sequences = typename std::conditional_t<
    IsEscaped && is_int_terminator::value,
    std::tuple<escape_sequence<EscapeIdx, Idx>>,
    std::tuple<>
    >;
// Next state is escaped if we're escaped but not terminated
//  or if we're not escaped but reached an escape character
using is_next_state_escaped = typename std::conditional_t<
    (IsEscaped && !is_int_terminator::value) || 
        (!IsEscaped && is_escape_char::value),
    std::true_type,
    std::false_type
    >;
// Increment the current index, pass the isEscaped state, reset or 
//  pass the EscapeIdx
using tail_sequences = format_parser<CharTuple, CharTupleSize, Idx + 1, 
    is_next_state_escaped::value, 
    (is_next_state_escaped::value) ? (Idx) : (EscapeIdx)>::escape_sequences;
// Combine the current result with the results of the next indexes
using escape_sequences = utils::tuple_cat_t<current_escape_sequences, 
    tail_sequences>;
// Recursion edge case omitted, but its pretty straightforward
```

With this implementation, the following expression:
```c++
format_parser<CharTuple, std::tuple_size_v<CharTuple>, 0, 
    false, 0>::escape_sequences;
```
will be a tuple type containing sequences which correspond to parsed escaped sequences in the format string.

Unlike the example above, which I wanted to keep relatively simple. The framework can detect multiple amount of terminators (which can be extended/configured). The format parser collects a tuple of argument parsers, each specialized based on the format part that was escaped (using the begin/end indexes). Each specialization detects the exact type that corresponds to the format part and generates a specific code to serialize it.

#### From Parsed Format To Serialization
I'd like to begin this section with an example of an argument parser which I mentioned earlier:
```c++
// Argument parser for integer types
template <typename FormatTuple, std::size_t EscapeIdx, std::size_t TerminatorIdx>
struct argument_parser
{
    ...
    // A utility that counts the number of `l` characters between indexes
    static constexpr std::size_t num_of_l = 
        tuple_counter<FormatTuple, char, 'l', EscapeIdx, TerminatorIdx>::count;
    static_assert(num_of_l < 3, "Invalid conversion specifier");
    // 0/1 `l`'s mean 32bit, 2 `l`'s mean 64bit
    static constexpr std::size_t argument_size = ((num_of_l == 0) ? 
        (sizeof(std::uint32_t)) : (sizeof(std::uint32_t) * num_of_l));
    // `%d` expects `int`, `%ld` expects `long` and `%lld` expects `long long`
    using argument_type = typename std::conditional<
        num_of_l == 0,
        int,
        typename std::conditional<
            num_of_l == 1,
            long,
            long long>::type>::type;

    template <typename Logger, typename Arg>
    inline static void apply(Logger& logger, const Arg arg)
    {
        ...
        logger.write((std::uint8_t *)&arg, argument_size);
    }
    ...
};
```
Note that it extracts both the size that this argument will take in the format and the C++ type that we expect to be provided by the caller. The `apply` function must be provided by the argument_parser and will later be called when serializing the log line. The templated Logger type allows us to configure and extend our output "sinks" in a way that doesn't require vtables (which improves performance); and since everything is provided at compile-time, there's really no need for vtables.

The serialization call can be implemented like this:
```c++
using argument_tuple = typename string_format::format_parser::argument_tuple;

template<typename Logger, typename ArgTuple, std::size_t Idx, 
    typename Arg = typename std::tuple_element<Idx, ArgTuple>::type>
inline void apply_arg(Logger& logger, const Arg arg) {
    using argument_parser = typename std::tuple_element_t<Idx, argument_tuple>;
    ...
    using = expected_arg_type_from_format = 
        typename argument_parser::argument_type;
    static_assert(std::is_convertible<Arg, expected_arg_type_from_format>::value,
        "Discrepency between argument type deduced from format and the given argument's type");
    argument_parser::apply(logger, arg);
};

// C++17 fold expression. Calls apply_arg for each one of the arguments,
//  with it's respective argument parser
template<typename Logger, typename ArgTuple, std::size_t... I>
inline void _apply_args(Logger& logger, ArgTuple args, 
    std::index_sequence<I...>) {

    (apply_arg<Logger, ArgTuple, I>(logger, std::get<I>(args)), ...);
    ...
}
// Add index sequence
template<typename Logger, typename... Args>
inline void apply_args(Logger& logger, std::tuple<Args...> args) {
    _apply_args(logger, std::move(args), std::index_sequence_for<Args...>{});
}
// Convert variadic arguments to tuple
template<typename Logger, typename... Args>
void log_serialize(Logger& logger, Args&&... args) {
    logger.write((std::uint8_t *)format_string::_chars, fmt_size());
    apply_args(logger, std::forward_as_tuple(args...));
}
```

The last three functions create the interface between the caller (which provides the format and variadic arguments) and the tuple of argument parsers discussed above. The `apply_arg` function takes a single argument and a single argument parser and applies the argument to the parser's `apply` function, after checking the validity of the argument.

#### Loggers And Writing To An Output
The templated loggers mentioned above provide the serialization with a `write` function and a `line_hint` functions. The `write` function objective is to remove the need for buffering/allocations on the serialization (argument parsers) side and delegate it to code that handles the I/O. In a naive approach, each `write` would directly call `fwrite`, and no buffering/allocation will be needed in the logger's side as well. However, from my experiments I've learned that implementing a simple buffer in the logger helps the performance tremendously. The `line_hint` function allows us to give the logger a hint that we've completed writing a single log line; this helps buffering loggers to decide when to flush.

Loggers may receive a tuple of "prefixes", these allows the user to customize their logs structure to their needs. Prefix examples include a priority indication (err/warn/info/etc.), time indicator (timestamp or date/time), PID/TID indicator and the users can easily extend and add their own. The prefixes take no arguments from the log call itself (as they are independent of any specific log line) and are written before the log line. Interestingly, the same mechanism for writing log lines (UDL+serializing) easily allows us to write the prefixes. The result is extra entries in the binary format, that are parsed just like log lines. The only difference is that we append a `'\n'` character at the end of the log lines to separate them.

#### Parsing The Binary Format
Parsing the log file in python is pretty straight forward. We have to implement the parser, which is way easier in python than in template meta-programming. We have to implement the terminators and the correct type and length deduction from each escape sequence. Finally, we need to implement the deserialization. I've hacked together a parsing script that you can find in the repository; it's not the prettiest or the fastest, but it works. If it runs too slow for you, try running it with `pypy`. If parsing performance is still an issue, the script could be improved by caching format parsing or using a faster language to implement it with.

# Decompression
I realize that this is a very technical post. I've tried to strike a balance that would fit a blog post; still I bet some will find it too verbose, while others will find it lacking. If you're looking for a simple quick start - these can be found in the [github repository][llcpp-gh]. If you're looking for a better understanding of how this works, the best way is to read the code! I'd also be more than happy to answer any questions regarding this framework on Github or in the comments below.

The next post is about benchmarking. I'll cover a couple of important points regarding logging performance comparisons and will show the results of my tests. You can read it here: [Part 3][part3].

[llcpp-gh]:
[part3]:
[parsing-compile-time]: https://akrzemi1.wordpress.com/2011/05/11/parsing-strings-at-compile-time-part-i/
[user-defined-string-literals]: http://en.cppreference.com/w/cpp/language/user_literal