---
title: "Review of Thriving in a Crowded and Changing World"
date: 2022-01-09
---

[Thriving in a Crowded and Changing World][0] is a bulky paper written by
C++ author Bjarne Stroustrup for the _History of Programming Languages_
ACM conference. I collected a few quotes below that I found interesting.

[0]: https://dl.acm.org/doi/pdf/10.1145/3386320

> I wanted that tool to write a version of the Unix kernel that could work on
multiple processors connected through a local area network or a shared memory.

What happened to that vision?

> In the late 1980s, the power of computers increased dramatically, I added Templates.

This solved the too much computer power issue for a very long time.

> iostreams -- an elaboration by Jerry Schwartz and the standards committee of my simple 1984
streams library to handle a wide variety of character types, locales, and buffering strategies.

I wonder what he means by "elaboration"...

> I was highly amused to find that representatives from Intel and IBM effectively
vetoed that idea by pointing out that by adopting the Java memory model for C++
we would slow down all JVMs by a factor of at least two. Consequently, to
preserve the performance of Java, we had to adopt a far more complex model for
C++. Ironically and predictably, C++ was then criticized for having a more
complicated memory model than Java.

Unfortunately, I couldn't find a reference for that.
I admire tough that the committee has such high ethics.

> This is not just "design by committee;" it is "design by a confederation of committees."

> I consider the threads-and-locks level of concurrency the worst model for application
use of concurrency

> Why did we require that programmers should use `constexpr` to mark functions that can be
executed at compile time? In principle, the compiler can figure out what can be computed at
compile time, but without an annotation, users would be at the mercy of variations in cleverness of
compilers and a compiler would need to keep bodies of all functions around "forever" in case they
were needed for constant expression evaluation

I always wondered, didn't know that. Unfortunately, I still do not really understand why.

> Unfortunately, beginners flock to examine the most horrendously specialized
code and take great pride in explaining it (often erroneously) to others.
Bloggers and speakers enhance their reputations by showing off hair-raising
examples. This is a major source of C++’s reputation for complexity.

I don't get what's wrong with `using swallow = int[]; (void)swallow{0, (f(t), int{})...};`.
Rolls off the tongue.

On exception specifications:

> As I feared, that led to maintenance problems, run-time overhead as an
exception was repeatedly checked along the unwinding path, and bloated source
code. For C++11, exception specifications were deprecated and for C++17, we
finally removed them (unanimously).

Thank you guys for using `using namespace std;`, forcing the committee to chose
the technically correct names.

> They are called unordered (e.g., `unordered_map`) to distinguish them from
the older, ordered, standard containers (e.g., `map`) and because the obvious
names (e.g., `hash_map`) had been used extensively in other libraries before
C++11.

On digit separators:

> Many, including me, argued for using the underscore as a separator (as in
several other languages). Unfortunately, people were using the underscore
as part of user-defined literal suffixes

Past forced mistakes:

> The prefix `template<...>` syntax was not my first choice when I designed
templates. It was forced upon me by people worried that templates would be
misused by less competent programmers, leading to confusion and errors. The
heavy syntax for exception handling, `try { ... } catch( ... ) { ... }`, was a
similar story.

The bashing of C++17 is a recurring theme:

> I have not been able to identify any unifying themes for the C++17 features. These features
look to be merely a set of "bright ideas" thrown into the language and standard library as voting
majorities could be found.

Now I understand why:

```
auto {x ,y ,z} = f (); // proposed
auto [x ,y ,z] = f (); // accepted
```
> This is a minor change, but I think a mistake.

Regarding optional/variant/any:

> I see these three variants of the idea of a discriminated union as a stop-gap measure. Functional-
programming-style pattern matching is a far more elegant, general, and
potentially more efficient solution to the problems with unions.

> I consider operator dot a prime example of the members of the committee lacking
a shared view of what C++ is and what it should become. Thirty years, six proposals, many
discussions, much design and implementation work, and we have nothing.

> In retrospect, I don’t think that the object-oriented notation (e.g., x.f(y)) should ever have
been introduced. The traditional mathematical notation f(x,y) is sufficient. As a side benefit, the
mathematical notation would naturally have given us multi-methods, thereby saving us from the
visitor pattern workaround.

Regarding the spaceship operator `<=>`:

> So, instead of a facility that made trivial examples work as expected for novices, we got an
elaborate facility that allows experts to carefully craft subtle comparisons.

Expectations:

> Coroutines - should be very fast and simple.

At this point in the paper, you just know the final result:

> a complete explanation of the proposals
written, presented, and discussed is beyond the scope of this paper. Here, I
present only an outline. There are simply too many complex details for anything
else; the papers alone run into many hundreds of pages

> The conclusion was that it might be
possible to get the best of both approaches, but that would require serious study. That study took
years, yielded no clear results, and in the meantime more proposals emerged.

How dare you, Dennis:

> At the time, I heard the priceless comment: "Dennis isn’t a C
expert; he never comes to the meetings".

> The only thing that increases faster than hardware performance is human expectation.

I don't think this is true. The perceived latency of computers just got worse
in general in the past decades. I blame bloated software.
Humans expect a fluent user interface.
