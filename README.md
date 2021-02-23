# Parallel Processing Patterns (PP-Patterns)

**Table of contents**

- [Introduction](#introduction)

- [PP-Patterns](#pp-patterns)

  - [Batch Stage Chain](#batch-stage-chain)

  - [Request Aggregator](#request-aggregator)

  - [Rolling Poller Window](#rolling-poller-window)


## Introduction
While developing services with scalability, throughput, and performance as major concerns, 
some common problems in parallel processing are solved in similar ways. The solution for 
each problem is common so can be summarized as a pattern. The target problems are most related
to concurrent operation, parallel processing, throttle control, and I/O sensitive.  

These patterns are not as common as the GoF Design Patterns, but are effective and common 
in specific scenarios. Being aware of these patterns at early design and implementation phase will 
be beneficial for performance-critical services.

These patterns are summarized from my own development, so are named per my understanding. 
Some of them may already be summarized by others with a different name. If there's an existing 
reference, please let me know.

This list is a work in progress.

## PP-Patterns

### Batch Stage Chain

_Fine granular parallel execution based on stages, to achieve better parallelism._

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


### Request Aggregator
_Aggregate requests and perform in a batch manner._

#### Problem
WIP

#### Solution
WIP

#### Consequences
WIP

### Rolling Poller Window
_Polling states of large number of items with throttle control._

#### Problem
WIP

#### Solution
WIP

#### Consequences
WIP

