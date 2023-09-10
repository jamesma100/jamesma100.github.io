---
layout: post
title: "Python multithreading in 10 minutes"
---

It's usually the case that having more compute resources makes our jobs faster, as we can distribute our work onto multiple cores. In Python, however, this is complicated by the presence of the global interpreter lock, or GIL for short. On the one hand, using multiple threads may not always boost performance and can starve certain threads or even increase latency, due to the added overhead of context switching. On the other hand, not using multithreading at all could waste valuable time and resources. So how does the GIL affect our programs, and when should or shouldn't we use multithreading?

#### Old GIL
Before Python 3, the GIL works as sort of a cooperative multitasking mechanism. An executing thread will enter into a "check" every 100 ticks, with each tick loosely mapping to a bytecode instruction. During each one of these checks, the thread releases and then reacquires the GIL. This periodic check allows other threads a chance to acquire the GIL and run.

In the easy case, a running thread performs an I/O request, at which point it voluntarily releases the GIL and another waiting thread acquires it. A more interesting scenario arises when the running thread runs until the check. If there are other threads waiting, one will then acquire the GIL. Which one gets to run depends on the OS's priority queue (it may even be the original running thread).

What if there are multiple cores? In this case, threads battle over the GIL, and usually it is the I/O-bound tasks that lose. This is especially true during heavy load - an I/O thread wants to reacquire the GIL so it keeps trying, but every time it is signaled, another CPU-bound thread has already acquired it. A CPU-bound task can continuously run - even though it is periodically releasing the GIL, it is able to get it right back. This ultimately starves I/O bound threads, while degrading the performance of CPU-bound threads by introducing overhead.

#### New GIL
The new GIL released in Python 3.2 attempts to address this starvation problem. Under this new mechanism, when a waiting thread wants to run, it does a timed `cv_wait()` on the GIL, allowing the running thread to release the GIL voluntarily. If it does, the waiting thread acquires it. If not, the waiting thread waits for 5ms, known as the check interval, then sets a global variable `gil_drop_request`. The running thread is then forced to suspend execution and pass control of the GIL to the waiting thread.

This introduces a new problem known as the convoy effect. I/O functions, even those like `read()`ing from a socket that return almost immediately, have to relinquish the GIL to another CPU thread that wants to run, attempt to reacquire it, then go through the lengthy timeout process once again. Only then will it be able to continue with the I/O operation. This stalls non-blocking I/O tasks.

#### How to multithread in Python?
Luckily, there are still ways to multithread in Python. Since the GIL only operates on Python objects, anything that isn't pure Python is exempt from its constraints. For example, Python code that calls C functions can be parallelized with the help of libraries like `multiprocessing`. Operations like calculating a SHA-256 hash on a large message blob, for example, can be sped up this way. Many data analysis packages like `numpy` take advantage of this fact. On a single core, an I/O task that is interacting with a client by reading and writing to a network socket, for example, can allow another thread to run in the meantime.

#### Solutions?
There have been many attempts to solve the problems associated with the GIL, but so far none have been merged. We cannot just remove the GIL, since many libraries and extensions have been built on top of it. Decreasing the check interval may alleviate wait times for I/O operations, but at a cost to other multithreaded tasks due to increased number of context switches. Ultimately, what Python needs is a scheduling mechanism, such as a [multilevel feedback queue](https://en.wikipedia.org/wiki/Multilevel_feedback_queue), to assign priorities to different threads and ensure fair access to the interpreter.

For more details, read Victor Skvortsov's [deep dive](https://tenthousandmeters.com/blog/python-behind-the-scenes-13-the-gil-and-its-effects-on-python-multithreading/) into the the GIL and
David Beazley's [excellent talk](https://www.youtube.com/watch?v=Obt-vMVdM8s) on the convoy problem.
