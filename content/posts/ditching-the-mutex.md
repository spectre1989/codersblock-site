+++
title = 'Ditching the Mutex'
date = 2016-06-02T00:00:00Z
draft = false
+++

TL;DR - I wrote a multiple-producer multiple-consumer queue without any mutexes, it's pretty fast, GitHub.

About a year ago, a colleague was explaining why Erlang is so scalable, and it essentially comes down to a lack of shared memory. When you have two or more threads which share a section of memory, then whenever a thread is writing to that memory, no other threads can perform a read or write at the same time. This can result in code spending too much time waiting for the shared memory to be available for reading/writing, and therefore poor performance.

Communication between threads in Erlang is instead achieved using a message queue. Each process has it's own queue, and other processes can add messages to it. This sounded to me like a great model for concurrency - each thread only writes its own data, and any inter-thread communication is strictly done through a message queue. I wondered if I could create a similar system in C++. My colleague told me someone already did, and it's called Erlang, but at present I'm enjoying C/C++ a little too much to use anything else.

I figured that mainly what this needed was a high performance concurrent message queue. In theory it would be the only point of thread synchronisation, and therefore any concurrency-related bottlenecks would happen there. I knew shared memory was slow because it involved waiting for a mutex to lock, so what about lock-free concurrent data structures I'd heard vaguely about?

For a data structure to be lock-free, it needs to allow multiple operations to happen at once (e.g. one thread adding an item to a queue, while another removes an item). If one of those threads was to be suspended part-way through, it must not stop the other(s) from finishing what they're doing.

With only a single producer and a single consumer, this is a lot more straight-forward. This is the general idea: