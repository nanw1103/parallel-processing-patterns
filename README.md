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


## Introduction
While developing services with scalability, throughput, and performance as major concerns, 
some common problems in parallel processing are solved in similar ways. The solution for 
each problem is common so can be summarized as a pattern. The target problems are most related
to concurrent operation, parallel processing, throttle control, and I/O sensitive operations.

These patterns are not as common as the GoF Design Patterns, but are effective and common 
in specific scenarios. Being aware of these patterns at an early design and implementation phase will 
be beneficial for performance-critical services.

_These patterns are summarized from my own development, so are named per my understanding. 
Some of them may already be summarized by others with a different name. If there's an existing 
reference, please let me know._

This list is a work in progress.

## Consideration on Parallel Style and Synchronization Models

### Computer system fundamentally supports concurrent execution 
Modern computer system normally adopts the interrupt mechanism for I/O. Upon signaled, it triggers the 
CPU to suspend the current routine and run a different routine. Though the per-CPU execution is 
still sequential at micro level, the execution model exposed is logically parallel. Considering multiple 
CPUs and/or cores, a computer system fundamentally supports concurrent execution, and suitable for 
asynchronous model.

However, such asynchronous model is not intuitive to human mind, which is more 
comfortable with _sequential operations_. It's also non-intuitive to write synchronous routines down. 

### Sequential programming is the origin 
One could debate whether programming is for the computer to execute, or for human to write and read.
If we look back upon the evolution of programming languages, we can see that in the last 60 years,
programming language is evolving from computer friendly (punched card) to human friendly (assembly, 
C++, Python). The programming languages aims to optimize for human: make read and write easier. The 
computer oriented optimization is taken care by compiler, programming model by each language, and OS
stacks.

So human-friendly is a key for programming languages, and sequential operation is easy to be understood
and written down. Most of high-level programming languages today are _concurrent programming languages_,
that has sequential model as native support.    

The following is an example of sequential operations:


    Make a phone call to R2D2

    Ask R2D2 to translate an article (which takes some time), and wait for response
    
    Print the translation

    Make a phone call to 3-PO

    Ask 3-PO to translate an article (which takes some time), and wait for response

    Print the translation


The example is easy to write and understand because human mind is suitable for this style. The
tasks (translation process) run on separate systems (the robots), and is logically parallel from the
run of the main logic above. However, in many cases, we model the process like shown above, because
it has an undeniable advantage: easy to create and read for human. The key point in achieving this, 
is that the parallel execution is converted to a blocking operation: "ask and wait". This style is commonly 
adopted, at OS level, in libraries, and at application layers. And this blocking style is commonly 
supported natively by many programming languages, C, Java, Python, etc.

### Parallelism and asynchronous model are required

The cost of sequential operation is obvious: the parallelism and asynchronous capability provided by the
system is not utilized. So, the execution could take longer time and the CPU/IO is not fully utilized. Then
it comes to multiple technologies for this:

* Thread: OS level thread and the programming language side counterpart. Threading nicely retains the 
sequential logic style, while support parallel operation.
* Callback: describe what to do upon an event 
  

Example of parallel execution using threads 


    Thread 1:
    
        Make a phone call to R2D2

        Ask R2D2 to translate an article (which takes some time), and wait for response

        Print the translation
        
    Thread 2:

        Make a phone call to 3-PO

        Ask 3-PO to translate an article (which takes some time), and wait for response

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

One interesting observation I had is about asynchronous pattern as first-class support in programming
languages. Before talking about that, let's look back upon the sequential programming model: 
threading.

Threading model has its own advantages, derived from OS native implementation. It helps to 
sequentially describe the steps to complete a task. That is based on the synchronous nature:
synchronous operations are easier to describe and understand. But when asynchronous is the main theme,
the sequential model is hard to optimize. This difficulty is caused by the model provided by programming
language itself. You may think, is there a different model, that can describe asynchronous problems easier? 
Then genius invented asynchronous-native languages, like JavaScript, which do everything asynchronously by 
default, and the runtime handles the asynchronous operation to lower-level calls (OS, I/O), hiding the 
complexity from programmers. Such languages are fluent to describe asynchronous tasks.

Examples of sequential-native languages are C++, Java, and Python. By their nature, it's easy to describe
sequential problems.
Examples of asynchronous-native languages are JavaScript family (CoffeeScript, TypeScript). By their nature, 
it's easy to describe parallel routine.

The interesting thing is, for these two styles, none of them dominates. Later, sequential-native languages
added features to support describing in the asynchronous way, by introducing concepts like Future in Java 
and C++, _async_ and _await_ in C#, Rx framework, etc. These are easy to achieve in asynchronous-native 
languages like JavaScript. While JavaScript also introduced _await_ later to improve the capability to 
describe sequential operations.

There are other models and concepts which fundamentally simplify how parallelism problems are described or
concurrency achieved. For example, Java Executor, Java Parallel Streaming, Erlang messaging, 
Golang channel, Golang goroutine. More details are being hidden from developers, and the models
and concepts are better in abstraction for describing common problems.

No matter how the building block evolves, one of the core problems to solve for a I/O centric and performance
sensitive service remains unchanged: to identify the cross-boundary heavyweight operations, utilize the 
building blocks to optimize these operations via parallelism or batch, reduce number of calls, to achieve
shorter execution time, higher throughput, and lower system cost.   
 
 
## PP-Patterns

### 1. Batch Stage Chain

_Fine granular parallel execution based on stages, for better parallelism._

#### Problem
Traditional programming languages, functional programming, object-oriented programming, and school 
education normally leads programmer to write programs as function units from the beginning. 
That's normally a good practice for implementing sequential scenarios: a self-contained function unit. 

![self-contained unit](images/self-contained-unit.png?raw=true)

When parallelism is needed, a natural process is to reuse the existing function unit as a building block 
and run it in parallel to achieve speed and performance. Such an easy and straightforward approach normally 
works well at the beginning. Let's call this pattern as "Parallel Sequential Units".

![Parallel Sequential Units](images/parallel-sequential-units.png?raw=true)


However, when the sequence is complex, especially multiple heavyweight operations (I/O or inter-service calls)
are involved, parallel run of the function unit, in most of the cases, is not optimized. Because of two factors:
1. Normally there are more efficient ways to do batch I/O or inter-service call. They should be effectively 
grouped together.
2. Heavyweight operations could have concurrency limit, or API quota limit. Parallel execution of the existing
unit, logically doing the same operation concurrently, could either exceed the heavyweight operation limit (if 
the parallel number is high), or not fully utilizing the concurrency capability (if the parallel number is low).   

#### Solution
The solution for such a problem is to break the sequential operation into multiple stages. Within each stage, 
heavyweight operations can be done in a batch manner, or properly handling the concurrency. In ideal cases, such 
operation could also be optimized using batch operations and concurrency happen at this level. The stages 
are chained together by producer-consumer pattern. The output of a previous stage is fed as the input of the 
next stage in the chain. Let's call this pattern as "Batch Stage Chain". Such a pattern in many cases has better 
performance than the Parallel Sequential Units pattern, at the cost of non-intuitive implementation.

![Batch Stage Chain](images/batch-stage-chain.png?raw=true)

#### Consequences
* Since multiple calls of the same operation are handled together in a stage, optimization can be applied within 
each stage. It enables the possibility to achieve better concurrency and performance.
* The solution is non-intuitive than Parallel Sequential Units.  


### 2. Request Aggregator
_Aggregate requests and perform in a batch manner, while keep the simple style for callers._

#### Problem
WIP

#### Solution
WIP

#### Consequences
WIP

### 3. Rolling Poller Window
_Polling states of large number of items with throttle control._

#### Problem
WIP

#### Solution
WIP

#### Consequences
WIP

