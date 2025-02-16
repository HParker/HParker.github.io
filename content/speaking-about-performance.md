# Speaking about Performance

I get confused reading about performance differences. When someone says some software is **"twice as fast"** I understand it can now do two tasks in the time it took to do one. However, There are very similar sounding phrases with vastly different meanings.

Consider,

> *This system is 40% faster than the previous system*

compared with,

> *This system is 40% the speed of the previous system*

The first phrase is talking about a system that is slightly faster, the second is talking about a system that is significantly slower.

interestingly,

> *The system is 1.3x the speed*

and,

> *The system is 1.3x faster*

the first phrase indicates a slightly faster machine, but the second is more then twice as fast! **Or does it?** The really insidious problem with the words we use about performance is that often the person saying/writing them doesn't have a clear idea how the numbers might be misinterpreted. **"3 times as fast"** might be super clear in my head, but depending on how the reader interprets it, they might misunderstand it!

# The two main formats

Some of these refer only to the "faster"-ness of the system, while the other refers the the overall change of the system. Both are valuable ways to think about a system I don't think it is reasonable to expect people to fully standardize their language. Instead I think as readers and writers **we must be aware of the system we are currently operating with and try not to mix and match things that readers are likely to compare.**

# Generally talk about speedups on benchmarks

If you can, a good format for performance changes is,

> *This change causes a **xx.xx** times performance change on benchmark *Y**

where **xx.xx** is the performance difference in terms of "times faster" or "times slower". I like this format for a number of reasons:

- the difference is easy to parse. (over 1 is faster, under 1 is slower)

- It doesn't sound like marketing. It is much easier to over hype a change by trying to make the numbers as large as possible. A "230% as fast" sounds a lot cooler then "2.3x performance change".

# Talk explicitly about latency and throughput when you can.

It is important to remember that "speed" of a program can refer to *latency*, i.e. time to complete a single task, or **throughput** i.e. how many tasks you can complete in a second.

I previously said that, "this code is twice as fast" is clear, however being very pedantic, it is also ambiguous. There are 2x *latency* changes which do not result in a 2x *throughput* change. For example, you could speed up a web server to cut its response time in half, but if you are out of available I/O resources, it might not result in any throughput changes! Latency is important and often has some relationship with throughput, but it is at least good to keep the difference clear in your head.

When talking about latency changes talk about "amount faster then before" so, a 1.4x speedup is talking about latency. If you can talk about throughput for micro benchmarks, operations per second can be a useful unit of measure to include along with the **xx.xx** performance difference.

# Lastly,

There are reasons to use other verbiage and it is possible to convert between percentages and "times faster" metrics. If you take nothing else from this, when writing consider which one you are using and make sure it is unambiguous. When you are reading be sure that you understand which system is being used and do you best to evaluate and compare them carefully.