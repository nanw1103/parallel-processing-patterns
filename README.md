# Parallel Processing Patterns (PP-Patterns)

**Table of contents**

- [Parallel Processing Patterns (PP-Patterns)](#parallel-processing-patterns-pp-patterns)
  - [Introduction](#introduction)
  - [Understanding Parallelism and Synchronization](#understanding-parallelism-and-synchronization)
    - [Basics of Parallel Processing in Modern Systems](#basics-of-parallel-processing-in-modern-systems)
    - [Where Sequential Programming Comes From](#where-sequential-programming-comes-from)
    - [Why We Need Parallelism and Asynchronous Programming](#why-we-need-parallelism-and-asynchronous-programming)
    - [How Programming Models Have Evolved](#how-programming-models-have-evolved)
  - [Principles of Parallel Processing](#principles-of-parallel-processing)
    - [1. Separate Independent Work Early](#1-separate-independent-work-early)
    - [2. Use Batch Operations When Possible](#2-use-batch-operations-when-possible)
    - [3. Decouple Subsystems Instead of Chaining Operations](#3-decouple-subsystems-instead-of-chaining-operations)
    - [4. Design with System Limits in Mind](#4-design-with-system-limits-in-mind)
  - [PP-Patterns](#pp-patterns)
    - [1. Batch Stage Chain](#1-batch-stage-chain)
      - [The Problem](#the-problem)
      - [The Solution](#the-solution)
      - [Benefits and Tradeoffs](#benefits-and-tradeoffs)
    - [2. Request Aggregator](#2-request-aggregator)
      - [The Problem](#the-problem-1)
      - [The Solution](#the-solution-1)
      - [Benefits and Tradeoffs](#benefits-and-tradeoffs-1)
    - [3. Rolling Poller Window](#3-rolling-poller-window)
      - [The Problem](#the-problem-2)
      - [The Solution](#the-solution-2)
      - [Benefits and Tradeoffs](#benefits-and-tradeoffs-2)
    - [4. Sparse Task](#4-sparse-task)
      - [The Problem](#the-problem-3)
      - [The Solution](#the-solution-3)
      - [Benefits and Tradeoffs](#benefits-and-tradeoffs-3)
    - [5. Marker and Sweeper](#5-marker-and-sweeper)
      - [The Problem](#the-problem-4)
      - [The Solution](#the-solution-4)
      - [Benefits and Tradeoffs](#benefits-and-tradeoffs-4)
      - [Examples](#examples)
  - [Epilogue](#epilogue)

## Introduction
When building services, especially those that need to scale well and handle high traffic, we often run into common challenges related to parallel processing. To solve these, we’ve identified a set of repeatable solutions, which we refer to as patterns. These patterns mainly deal with running tasks at the same time, controlling how much work happens in parallel, and handling operations that are slow because of input/output (I/O).

These patterns may not be as widely known as the classic "Gang of Four" design patterns, but they are very useful in specific situations. Learning about them early in the design and development process can be a big help for building fast and efficient services.

These patterns come from real-world experience and were named during development. Others may have found similar solutions and given them different names. If you find a match elsewhere, feel free to open an issue or pull request in this repository to share it.

_[Source](https://github.com/nanw1103/parallel-processing-patterns/blob/main/README.md)_


## Understanding Parallelism and Synchronization

### Basics of Parallel Processing in Modern Systems
Modern computers often use an interrupt system for handling input and output (I/O). When an interrupt happens, the CPU temporarily stops what it's doing and handles something else. Even though the CPU still runs instructions one at a time, this behavior makes the system seem like it’s doing many things at once. With multiple CPUs and cores, real parallel execution is also possible, which fits well with asynchronous programming.

However, asynchronous models can be hard for people to follow, since we tend to think in steps, one thing after another. Because of this, it’s helpful to have tools and methods that make it easier to write code in a more straightforward, step-by-step (synchronous) way.


### Where Sequential Programming Comes From
Programming sits at the intersection of how computers work and how humans think. Over the past 60 years, programming languages have changed a lot. Early languages were designed around how machines worked (like using punch cards), but today’s languages, like C++, Python, and others, are designed to be easier for humans to read and write. The heavy lifting of optimizing for the computer is handled by compilers, while each language offers its own way of writing code and interacting with the system.

One key goal of modern programming languages is to be human-friendly. Writing code in a step-by-step (sequential) way is easy to understand and feels natural. That’s why most modern high-level languages support sequential programming by default, even if they can also handle tasks running in parallel.

Here’s a simple example that shows how sequential logic works:

    1. Make a phone call to R2D2.
    2. Ask R2D2 to translate an article (this takes time), and wait for the reply.
    3. Print the result.
    4. Make a phone call to 3-PO.
    5. Ask 3-PO to translate an article (this takes time), and wait for the reply.
    6. Print the result.

This is easy to write and easy to understand because it follows how we naturally think. Even though the translation work might happen on different systems (the robots), we write the code in a clear, ordered way. This approach is often preferred because it’s simple and clear.

The key technique here is to turn parallel or background work into a "wait until done" step using patterns like “ask and wait.” This style is widely used across systems, libraries, and apps. Many languages—including C, Java, and Python—support this kind of blocking behavior natively.

### Why We Need Parallelism and Asynchronous Programming

Sequential code, where tasks run one after another, can be slow, especially when some steps take time, like waiting for input/output. Modern computers are built to handle many things at once, so to take full advantage of that, we need ways to run tasks in parallel or asynchronously.

To solve this, developers use a few key techniques:

* **Threads**: These allow tasks to run at the same time, either managed by the operating system or the programming language. They let you keep writing in a step-by-step style while still doing multiple things at once.

* **Callbacks**: These are functions that get called when something finishes. You tell the system, “When this is done, run this code.” The runtime takes care of running things in the background and calling your code later, so you don’t have to manage all the details.

Here are two examples showing how to run tasks in parallel:


**Using Threads:**

```
Define thread 1:
    Make a phone call to R2D2.
    Ask R2D2 to translate an article (this takes time), and wait for the reply.
    Print the result
   
Define thread 2:
    Make a phone call to 3-PO.
    Ask 3-PO to translate an article (this takes time), and wait for the reply.
    Print the result
   
Then, run both threads at the same time.
```


**Using Callbacks:**

```
Make a phone call to R2D2.
Ask R2D2 to translate an article (non-blocking). When the result comes back:
    Print the result
   
Make a phone call to 3-PO.
Ask 3-PO to translate an article (non-blocking). When the result comes back:
    Print the result
```

Both approaches let you speed things up by not waiting for one task to finish before starting the next. Threads let you keep the code looking sequential. Callbacks make your code respond when something is ready. Each has its place, depending on what you're building.

### How Programming Models Have Evolved

It's interesting to see how programming languages have changed, especially with the rise of asynchronous programming as a built-in feature. But before we dive into that, let’s look back at how things used to work—with threads.

Threading works well because operating systems support it directly. It lets us write code in a clear, step-by-step way, which is easy to follow. But when we need to do lots of things at once (asynchronous tasks), the traditional sequential model starts to fall short. So, is there a better way to handle async tasks?

Yes—some languages were designed from the ground up for asynchronous programming. JavaScript is a good example. In JavaScript, almost everything runs asynchronously by default. The language and its runtime take care of the lower-level details, like I/O and waiting for events, so programmers don’t have to manage that complexity themselves. This makes it easy to write asynchronous code that’s clean and expressive.

Languages like C/C++, Java, C#, and Python were built to handle sequential tasks well. In contrast, JavaScript is naturally good at describing tasks that run in parallel or respond to events.

Interestingly, no single style has completely taken over. Instead, sequential-style languages have added tools to better support asynchronous tasks, like _Future/Rx/Reactive framework_ in Java, or _async/await_ in Python and C#. JavaScript, even though it’s asynchronous by nature, later added _await_ to make writing sequential logic easier.

There are also other models that help simplify parallel programming. For example:

* Java Executors and Parallel Streams

* Erlang message passing

* Go’s channels and goroutines

These tools hide some of the messy details and make it easier to describe common concurrency problems.

But no matter how programming models change, one thing stays the same—performance-critical services still need to identify and optimize expensive operations, especially those that cross boundaries (like network or disk I/O). Using parallelism, batching, and other smart techniques to reduce these slow operations can lead to faster execution, higher throughput, and lower costs.

## Principles of Parallel Processing

Engineers design based on experience, while architects rely on methodologies.

### 1. Separate Independent Work Early
At the top level, find tasks that don’t depend on each other and run them separately. This reduces interference between tasks and is a fundamental way to improve performance in parallel systems. If your overall design is too sequential, there won’t be many chances to speed things up later.


### 2. Use Batch Operations When Possible
Think of building software like building a castle—you choose blocks that shape the final structure. The tools and functions you use affect how efficiently you can build. Batch operations—doing many things at once—are often provided by lower system layers (like databases or APIs) and are usually optimized to perform better than handling one item at a time. Use them when you can.

### 3. Decouple Subsystems Instead of Chaining Operations
In an "operation cascade," each task triggers the next directly, like a chain reaction. While this approach is simple and natural, it can create bottlenecks in large systems, especially if some parts are slower than others. One slow component can hold back the whole system.

Instead, decouple your system into stages. Let each part do its job independently and at its own pace. This allows you to fine-tune and scale each piece separately. While this design is more complex, it avoids being limited by the slowest part of your system. Common patterns that follow this approach include:

* Producer-Consumer

* Batch Stage Chains

* Mark-and-Sweep

### 4. Design with System Limits in Mind
Every system has its limits—how much traffic it can handle, how much data it can process, and so on. Knowing where these limits are likely to appear helps you make better design choices up front and prepare for future scaling.


## PP-Patterns

### 1. Batch Stage Chain

_Break down complex workflows into stages to achieve more efficient and controlled parallelism._

#### The Problem
In many programming styles—like traditional, functional, or object-oriented—we usually start by writing self-contained functions that handle one complete task. This works well for sequential programs, where each step runs independently, one after another.


![self-contained unit](images/self-contained-unit.png?raw=true)

When we need better performance, the common approach is to run these existing functions in parallel. This is called the Parallel Sequential Units pattern, and it often works fine for simple cases.


![Parallel Sequential Units](images/parallel-sequential-units.png?raw=true)


However, for more complex workflows, especially ones that include heavy operations like file I/O or calling other services, running everything in parallel can lead to problems:
1. Batching is usually more efficient. For example, calling an API once with 100 items is faster than calling it 100 times.
2. Concurrency limits exist. If too many tasks run at once, you might hit API rate limits. But if too few run, you waste available capacity.

#### The Solution
Instead of running everything in parallel at once, break the work into stages. Each stage handles part of the job, either in batches or with a controlled level of concurrency.

This forms a Batch Stage Chain:
* Each stage processes its input in groups (batches) or parallel chunks.
* The output of one stage becomes the input for the next.
* The stages are connected like a producer-consumer pipeline.

This setup allows you to:
* Use batch operations efficiently.
* Control how much work runs in parallel at each step.
* Avoid overloading downstream systems.

Although this design can be more complex and less intuitive than just running existing functions in parallel, it usually performs better for heavy workloads.

![Batch Stage Chain](images/batch-stage-chain.png?raw=true)

#### Benefits and Tradeoffs
* ✅ Better performance through batching and controlled parallelism.
* ✅ Scales well even with heavy or rate-limited operations.
* ⚠️ Harder to implement than simple parallel execution.


### 2. Request Aggregator

_Group requests and process them in batches, while still offering a simple, single-request interface to callers._

#### The Problem
Working with one item at a time is usually easier than handling many. That’s why many software systems and libraries are built around single-resource operations—like "get one book" instead of "get a list of books." This keeps the interface clean and simple.

Because of layered software design, this single-item style tends to be used throughout the entire system, from the API to the backend.

But sometimes, we need to work with multiple resources at once for performance reasons. Unfortunately, we’re often stuck with interfaces that only support one item at a time. For example, a cloud API might only let you manage one VM per request, and you can’t change that.

In these cases, the typical solution is to run multiple single-resource operations in parallel. This works fine and has real benefits—it's easy to implement and still improves performance.

However, when you need even more speed, things get tricky.

Take network socket operations as an example:

* The OS (like Linux with epoll, or Windows with I/O completion ports) can handle sockets in batches for better performance.
* But high-level libraries often give you simple functions that only deal with one socket at a time.
* So, developers use threads or advanced techniques to handle multiple sockets in parallel—but it takes more effort to do this efficiently without breaking the simple interface.


#### The Solution

The Request Aggregator pattern allows you to:

* Keep exposing a simple interface—like "process one request."
* Internally, group multiple requests together and handle them in batches, especially when calling external services or doing I/O-heavy work.

It combines the simplicity of single-resource APIs with the performance benefits of batch operations.


![Request Aggregator](images/request-aggregator.png?raw=true)

#### Benefits and Tradeoffs
* ✅ Keeps your API clean and easy to use.
* ✅ Reduces the number of heavy or costly cross-system calls.
* ✅ Improves performance through internal batching.
* ⚠️ Requires careful design to manage timing, buffering, and batching without breaking the single-resource interface.

### 3. Rolling Poller Window

_Efficiently poll the status of many items by dividing them into manageable batches._

#### The Problem
When you need to check the status (or "poll") of a large number of items—like jobs, devices, or tasks—polling them all at once might overload the system. Polling them one by one is too slow and inefficient.

You may also face limits from the system you're polling:

* Maximum batch size
* Maximum response size
* API rate limits or quotas


#### The Solution
Use a rolling window—a fixed-size batch that moves across your list of items.

Here's how it works:
1. Set a window size based on system limits or performance goals.
2. On each polling cycle, poll only the items in the current window—ideally in a single batch request.
3. Cache the results locally.
4. Move the window forward to poll the next group of items.
5. After each full pass, filter out completed items, so only unfinished ones are polled in future rounds.
6. Repeat until everything is done.

This approach lets you poll large numbers of items in small, controlled batches without overwhelming the system.

 
![Rolling Poller Window](images/rolling-poller-window.png?raw=true)

#### Benefits and Tradeoffs
* ✅ Prevents API overload and respects system constraints
* ✅ Efficient for large-scale polling tasks
* ✅ Supports dynamic item updates (add/remove items during runtime)
* ✅ Works well with Request Aggregator for further performance gains
* ⚠️ Requires careful tracking of which items are done vs. still active


### 4. Sparse Task
_Break up long-running tasks into smaller, short-lived ones to improve scalability and reduce resource usage._

#### The Problem
In many distributed systems, a [task scheduler](https://en.wikipedia.org/wiki/Job_scheduler) handles jobs submitted by applications. This is common in systems built on [Shared-nothing Architecture](https://en.wikipedia.org/wiki/Shared-nothing_architecture), where each component works independently.

A typical long-running task might look like this:

1. Do some work.
2. Wait for something to finish (often involves polling an external service).
3. Mark the task as completed, failed, or canceled.

![Conventional Long-run Task](images/sparse-task-conventional-long-run-task.png?raw=true)

This setup is easy to understand, but it comes with key downsides:
1. Each task holds onto a worker thread, even while it's just waiting. This limits how many tasks can run in parallel.
2. Polling creates extra I/O traffic.
3. It’s hard to batch I/O across systems if tasks are running on different nodes.


#### The Solution
The Sparse Task pattern transforms a long-running task into multiple small, independent pieces, changing the overall structure from a polling model to an event-driven model. The pattern comprises the following components:
1. Submitter
    * Starts the main task.
    * Also schedules a timeout monitor task to check if it finishes in time.
    * May or may not run as a distributed task, depending on your system.
2. Timeout Monitor Task
    * This is a lightweight scheduled task that will alert the system if the main task doesn’t complete in time.
    * It runs reliably in a distributed way and survives restarts.
    * All state is stored externally (in a DB, for example), not in memory.
3. Event Handler
    * Listens for success or failure events (via callback or message).
    * Cleans up the monitor task when the operation is finished.


![Sparse Task Pattern](images/sparse-task-pattern-sequence.png?raw=true)

The system gets notified through a callback, such as a message bus event, webhook, or similar. If you don’t have messaging infrastructure, you can still simulate callbacks by using a batch poller that checks task status in intervals.

![Notification by Batch Poller](images/notification-by-batch-polling.png)


#### Benefits and Tradeoffs

* ✅ Much better scalability: Tasks no longer block worker threads just to wait. You can run many more tasks than you have workers.
* ✅ Lower I/O usage: Moving from polling to event-driven reduces system traffic.
* ✅ Simpler task logic: Each small piece can be idempotent (safe to retry), which is easier to manage.
* ⚠️ More moving parts: Requires callbacks, messaging, or polling infrastructure.
* ⚠️ Harder to track tasks: One logical task is now split across two or more parts. You need a reliable way to correlate them.
* ⚠️ Monitor task must match the callback: When a task finishes, the system must correctly identify and stop its timeout monitor.

### 5. Marker and Sweeper
_Flag and process tasks separately to decouple high-speed and low-speed systems._

#### The Problem

In many systems, events or triggers happen that require follow-up work on a specific item. For example, in an event-driven system, you might get an event saying:
"Room 1 needs cleanup."

The simple approach is to do the cleanup right away when the event happens. This is fine for basic cases—but in more complex systems, this causes problems:

1. The event rate might be faster than the system can handle, creating a backlog.
2. You might process the same item multiple times unnecessarily.
3. The action might trigger another event, leading to cascading operations that are hard to control as the system grows.


#### The Solution

Use the Marker and Sweeper pattern to separate what needs to be done from when it's actually done.

This pattern has three parts:

1. **Marker**: When an event happens, it quickly marks the item (e.g., “Room 1 needs cleanup”) and adds it to a flag set. No processing is done yet.

2. **Flag Set**: This is a lightweight data structure (e.g. a set) that keeps track of which items need processing. It avoids duplication: if Room 1 is already flagged, it won’t be flagged again.

3. **Sweeper**: A separate process that runs at its own pace, picks up items from the flag set, and does the actual work—such as cleaning up Room 1.

This setup is similar to the Producer-Consumer pattern, but it’s optimized for:

1. Deduplication
2. Decoupling fast and slow components
3. Batch processing


![Marker and Sweeper](images/marker-and-sweeper.png)

The Marker and Sweeper pattern is specific variation to the Producer and Consumer pattern.

#### Benefits and Tradeoffs
* ✅ Separation of concerns: Marking is fast and doesn’t block; sweeping can be slower and more thorough.
* ✅ High throughput: The marker is lightweight and can handle a high rate of events.
* ✅ No duplicate work: The flag set avoids reprocessing the same item.
* ✅ Batch-friendly: The sweeper can group items for efficient processing.
* ⚠️ Requires coordination: You need logic to manage the flag set and know when items are done.
* ⚠️ Delayed action: There may be a small delay between the event and when it’s handled.


#### Examples
Real-world examples of this pattern include:
* Network socket APIs like select() or Linux epoll, where sockets are flagged for readiness and then processed separately.
* Garbage collectors, which mark objects for cleanup and sweep them later.


## Epilogue
Patterns come from real-world experience, and they’re meant to be used in real-world systems. This back-and-forth between practice and design often makes me reflect on how we build software.

There’s no single pattern that fits every situation. But for each problem, there’s usually a pattern that fits well enough. What really guides good design isn’t the pattern itself, but the principles behind it—principles we've learned through experience. These are what we should stay true to.

And maybe, when we look back one day at our work in engineering, what stands out the most won’t be any specific pattern—but rather a balanced mindset. A steady, thoughtful way of thinking. A kind of [Doctrine of the Mean](https://en.wikipedia.org/wiki/Doctrine_of_the_Mean) in how we build and decide.