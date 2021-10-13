# Parallel Processing Patterns (PP-Patterns)

**Table of contents**

- [Introduction](#introduction)
- [Consideration on Parallel Style and Synchronization Models](#consideration-on-parallel-style-and-synchronization-models)
  - [Computer system fundamentally supports concurrent execution](#computer-system-fundamentally-supports-concurrent-execution)
  - [Sequential programming is the origin](#sequential-programming-is-the-origin)
  - [Parallelism and asynchronous model are required](#parallelism-and-asynchronous-model-are-required)
  - [Programming model evolved](#programming-model-evolved)
- [PP-Patterns](#pp-patterns)

  - [1. Batch Stage Chain](#1-batch-stage-chain)

  - [2. Request Aggregator](#2-request-aggregator)

  - [3. Rolling Poller Window](#3-rolling-poller-window)

  - [4. Sparse Task] (#4-sparse-task)

  - [5. Ledger] (#5-ledger)

## Introduction
While developing services with scalability, throughput, and performance as major concerns, 
some common problems in parallel processing are solved in similar ways. The solution for 
each problem is common so can be summarized as a pattern. The target problems are mostly related
to concurrent operations, parallel processing, throttle control, and I/O sensitive operations.

These patterns are not as common as the GoF Design Patterns but are effective and common 
in specific scenarios. Being aware of these patterns at an early design and implementation phase will 
be beneficial for performance-critical services.

_These patterns are summarized from my own development, so are named per my understanding. 
Some of them may already be summarized by others with a different name. If there's an existing 
reference, please let me know._

This list is a work in progress.

## Consideration on Parallel Style and Synchronization Models

### Computer system fundamentally supports concurrent execution 
Modern computer system normally adopts the interrupt mechanism for I/O. Upon signal, it triggers the 
CPU to suspend the current routine and run a different routine. Though the per-CPU execution is 
still sequential at the micro-level, the execution model exposed is logically parallel. Considering multiple 
CPUs and/or cores, a computer system fundamentally supports concurrent execution and is suitable for the 
asynchronous model.

However, such an asynchronous model is not intuitive to the human mind, which is more 
comfortable with _sequential operations_. It's also non-intuitive to write synchronous routines down. 

### Sequential programming is the origin 
One could debate whether programming is for the computer to execute, or for humans to write and read.
If we look back upon the evolution of programming languages, we can see that in the last 60 years,
programming languages are evolving from computer-friendly (punched card) to human-friendly (assembly, 
C++, Python). The programming languages aims to optimize for human: make read and write easier. The 
computer-oriented optimization is taken care of by the compiler, programming model by each language, and OS
stacks.

So human-friendly is a key for programming languages, and sequential operation is easy to be understood
and written down. Most high-level programming languages today are _concurrent programming languages_,
that has sequential model as native support.    

The following is an example of sequential operations:


    Make a phone call to R2D2

    Ask R2D2 to translate an article (which takes some time), and wait for a response
    
    Print the translation

    Make a phone call to 3-PO

    Ask 3-PO to translate an article (which takes some time), and wait for a response

    Print the translation


The example is easy to write and understand because the human mind is suitable for this style. The
tasks (translation process) run on separate systems (the robots) and are logically parallel from the
run of the main logic above. However, in many cases, we model the process like shown above, because
it has an undeniable advantage: easy to create and read for humans. The key point in achieving this 
is that the parallel execution is converted to a blocking operation: "ask and wait". This style is commonly 
adopted, at the OS level, in libraries, and at application layers. And this blocking style is commonly 
supported natively by many programming languages, C, Java, Python, etc.

### Parallelism and asynchronous model are required

The cost of sequential operation is obvious: the parallelism and asynchronous capability provided by the
system are not utilized. So, the execution takes a long time and the CPU/IO is not fully utilized. Then
it comes to multiple technologies for this:

* Thread: OS-level thread and the programming language side counterpart. Threading nicely retains the 
sequential logic style, while support parallel operation.
* Callback: describe what to do upon an event 
  

Example of parallel execution using threads 


    Thread 1:
    
        Make a phone call to R2D2

        Ask R2D2 to translate an article (which takes some time), and wait for a response

        Print the translation
        
    Thread 2:

        Make a phone call to 3-PO

        Ask 3-PO to translate an article (which takes some time), and wait for a response

        Print the translation
        
    Run thread 1 and thread 2 concurrently


Example of parallel execution using callbacks 


     Make a phone call to R2D2

     Ask R2D2 to translate an article (non-blocking), upon on response do:
     
        Print the translation
        
     Make a phone call to 3-PO

     Ask 3-PO to translate an article (non-blocking), upon on response do:

        Print the translation


### Programming model evolved

One interesting observation I had is about the asynchronous patterns as first-class support in programming
languages. Before talking about that, let's look back upon the sequential programming model: 
threading.

The threading model has its own advantages, derived from OS native implementation. It helps to 
sequentially describe the steps to complete a task. That is based on the synchronous nature:
synchronous operations are easier to describe and understand. But when asynchronous is the main theme,
the sequential model is hard to optimize. This difficulty is caused by the model provided by programming
languages themselves. You may think, is there a different model, that can describe asynchronous problems easier? 
Then genius invented asynchronous-native languages, like JavaScript, which do everything asynchronously by 
default, and the runtime handles the asynchronous operation to lower-level calls (OS, I/O), hiding the 
complexity from programmers. Such languages are fluent to describe asynchronous tasks.

Examples of sequential-native languages are C++, Java, and Python. By their nature, it's easy to describe
sequential problems.
Examples of asynchronous-native languages are the JavaScript family (CoffeeScript, TypeScript). By their nature, 
it's easy to describe a parallel routine.

The interesting thing is, for these two styles, none of them dominates. Later, sequential-native languages
added features to support describing in an asynchronous way, by introducing concepts like Future in Java 
and C++, _async_ and _await_ in C#, Rx framework, etc. These are easy to achieve in asynchronous-native 
languages like JavaScript. While JavaScript also introduced _await_ later to improve the capability to 
describe sequential operations.

There are other models and concepts which fundamentally simplify how parallelism problems are described or
concurrency achieved. For example, Java Executor, Java Parallel Streaming, Erlang messaging, 
Golang channel, Golang goroutine. More details are being hidden from developers, and the models
and concepts are better in abstraction for describing common problems.

No matter how the building block evolves, one of the core problems to solve for I/O centric and performance
sensitive service remains unchanged: to identify the cross-boundary heavyweight operations, utilize the 
building blocks to optimize these operations via parallelism or batch, reduce the number of calls, to achieve
shorter execution time, higher throughput, and lower system cost.   
 
 
## PP-Patterns

### 1. Batch Stage Chain

_Fine granular parallel execution based on stages, for better parallelism._

#### Problem
Traditional programming languages, functional programming, object-oriented programming, and school 
education normally leads programmers to write programs as function units from the beginning. 
That's normally a good practice for implementing sequential scenarios: a self-contained function unit. 

![self-contained unit](images/self-contained-unit.png?raw=true)

When parallelism is needed, a natural process is to reuse the existing function unit as a building block 
and run it in parallel to achieve speed and performance. Such an easy and straightforward approach normally 
works well at the beginning. Let's call this pattern "Parallel Sequential Units".

![Parallel Sequential Units](images/parallel-sequential-units.png?raw=true)


However, when the sequence is complex, especially multiple heavyweight operations (I/O or inter-service calls)
are involved, parallel run of the function unit, in most of the cases, is not optimized. Because of two factors:
1. Normally there are more efficient ways to do batch I/O or inter-service calls. They should be effectively 
grouped together.
2. Heavyweight operations could have a concurrency limit, or API quota limit. Parallel execution of the existing
unit, logically doing the same operation concurrently, could either exceed the heavyweight operation limit (if 
the parallel number is high), or not fully utilizing the concurrency capability (if the parallel number is low).   

#### Solution
The solution for such a problem is to break the sequential operation into multiple stages. Within each stage, 
heavyweight operations can be done in a batch manner, or properly handle the concurrency. In ideal cases, such 
operation could also be optimized using batch operations and concurrency happen at this level. The stages 
are chained together by the producer-consumer pattern. The output of a previous stage is fed as the input of the 
next stage in the chain. Let's call this pattern "Batch Stage Chain". Such a pattern in many cases has better 
performance than the Parallel Sequential Units pattern, at the cost of non-intuitive implementation.

![Batch Stage Chain](images/batch-stage-chain.png?raw=true)

#### Consequences
* Since multiple calls of the same operation are handled together in a stage, optimization can be applied within 
each stage. It enables the possibility to achieve better concurrency and performance.
* The solution is non-intuitive than Parallel Sequential Units.  


### 2. Request Aggregator
_Aggregate requests and perform in a batch manner, while keeping the simple style for callers._

#### Problem
Operating a single resource is normally easier than operating multiple resources at the same time. Single-resource 
operation is easy to understand, easy to design, and easy to implement: in general, more human mind friendly. 
Exposing a single-resource operation interface is a common style, chosen by software systems and libraries. For 
example, in a book store management system, the interface to get a single book versus the interface to get 
multiple books.

Due to the layering in software design, such a single-resource operation interface style is inherited and spread to 
multiple layers and related systems. 

In such a context, when an operation on multiple resources is needed, due to the software layering and existing 
interfaces, it may leave developers no choice but to stick to the single-resource operations. For example, a 
third-party cloud service exposes only single-resource operation, and it's out of our control. 

For performance-related scenarios, when operating on multiple resources, making concurrent calls using the single 
resource operation is a natural choice. This is similar to the aforementioned self-contained function unit, 
in a previous section. Such an approach has lots of advantages in practice, and in many cases may be the best choice. 
However, when additional performance is needed, it will be hard.

One example is the network socket I/O library provided by OS. At the OS level, reflecting the underlying hardware, 
the socket presentation and operation are both handled as a batch, examples are Linux epoll, and Windows completion
port. However, high-level libraries tend to expose the interfaces to operate a single socket, e.g. read from a socket. 
Then when multiple sockets are to be read concurrently, we see lots of implementation use threads to achieve 
concurrent operations. One of the advantages of such a style is easy-to-read: for the function unit, it's still a 
single socket that is being operated, in some sequence. But in the layers under the API, it's either not optimized,
or requires good design and implementation to achieve the easy-to-use single-resource interface, while not losing
the batch operation advantages. 

#### Solution

The aggregator pattern aims to preserve the easy-to-use single-resource operation interfaces while utilizing 
batch operations to optimize cross-system calls, which are normally heavy. 

![Request Aggregator](images/request-aggregator.png?raw=true)

#### Consequences
The single-resource operation interface is reserved, while batch operation can be applied to optimize cross-system
heavy calls. With careful design, the number of cross-system calls can be reduced.

### 3. Rolling Poller Window
_Poll states of a large number of items with controlled batches._

#### Problem
When a large number of items needs to be polled, sometimes it may not be possible to poll them all together, and 
polling each item individually is normally inefficient.
So it is needed to divide the operation into small batches, to meet the target service requirements. For example,
batch size limit, response size limit, quota limit of the number of API calls, etc.

#### Solution
In the array of the target items, a rolling window is used to select the items to poll. The window has a size that is
selected according to the target system requirement for optimization. In each turn, only the items in the poller 
window are polled from the target system, ideally as a single batch. The result is cached locally. The poller window 
moves on inside the array until it reaches the end. Then the items are reorganized, either in the array or 
using separate data structures so that only the unfinished items are remaining in the array as the polling target. 
The rolling window continues. This process continues until all items reach to desired completion state. 
 
![Rolling Poller Window](images/rolling-poller-window.png?raw=true)

#### Consequences
Requests to the target system are fired in a controlled manner, within constraints of the target system.
This mechanism also works well with dynamic adding or removing items during the run. 
The Rolling Poller Window could be used together with the Request Aggregator.

### 4. Sparse Task
_Break long-run tasks into multiple short-run tasks for scalability._

#### Problem
Distributed job, provided by [job scheduler](https://en.wikipedia.org/wiki/Job_scheduler), is a common concept used in service systems, for decoupling, reliability, and scalability, in [Shared-nothing Architecture](https://en.wikipedia.org/wiki/Shared-nothing_architecture). The application submits jobs to a job scheduler, and the jobs are executed in the background in a reliable and distributed manner. 
A conventional and intuitive implementation of a long-run job normally has the following sequence:

1. Perform some operation
2. Wait for the completion, which normally involves repeated polling from a target service.
3. Complete the task, as successful, canceled, or error.

While such a long-run pattern is good because of intuitive in many cases, it has the following drawbacks:
1. Each long-run job occupies an execution capacity from the job scheduler, normally a thread. Thus the total long-run jobs that can be processed at the same time are limited by the capacity of the scheduler. The system throughput of jobs is limited. 
2. The polling model incurs additional I/O.
3. Hard to aggregate inter-system I/O as batches for optimization, since distributed jobs could run on different nodes in a cluster.

![Conventional Long-run Task](images/sparse-task-conventional-long-run-task.png?raw=true)

#### Solution
A sparse task breaks the long-run task into multiple pieces. Change the overall structure from a polling model to an event-driven model. Sparse Task pattern consists of the following components:
1. **Submitter**: performs the actual task operation, and schedules a _timeout monitor task_ on _task scheduler_, per task.
2. **Event Handler**: handler to task completion events.
3. **Timeout monitor task**: a scheduled task, upon running, notifies the _event handler_ that the task is timed out.

![Sparse Task Pattern](images/sparse-task-pattern-sequence.png?raw=true)

This pattern relies on a callback to notify the event handler about task completion. Example callbacks are message bus events, Webhook callbacks. If the messaging infrastructure does not exist, the pattern can still be achieved by a batch poller based callback:

![Notification by Batch Poller](images/notification-by-batch-polling.png)


#### Consequences

1. Scalability increases since the task scheduler capacity (threads) are not occupied by the blocking logic per long-run task. Tasks that more than the number of scheduler worker capacity can run logically in parallel.
2. Task implementation is simplified. With long-run tasks, in a high-available environment, the full task logic with all internal steps should be implemented as idempotent. With sparse tasks, each small piece needs to be idempotent, which is easier.
3. System I/O is reduced, by moving from polling pattern to event-driven pattern.
4. Task tracking of sparse tasks is more complex and may require additional consideration to track the logical task.
5. System dependency increase. Normally polling is simpler and callback or messaging requires additional infrastructure.

### 5. Ledger

#### Problem
WIP

#### Solution
WIP

#### Consequences
WIP