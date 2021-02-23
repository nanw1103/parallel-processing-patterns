# Parallel Processing Patterns (PP Patterns)

## Introduction
While developing services with scalability and performance as a major concern, some common problems in parallel processing are solved in similar ways. The solution for each problem is common so can be summarized as pattern. These patterns are not as common as the GOF Design Patterns, but are effective and common in specific scenarios.

These patterns are summarized from my own development, so are named per my understanding. Some of them may already be summarized by others with a different name. If there's an existing reference, please let me know.

This list is a work in progress.

## Patterns

### Batch Stage Chain Pattern

Traditional programming languages, functional programming, object-oriented programming, and school education normally leads programmer to write programs as function units from the beginning. That's normally a good practice for implementing sequential scenarios: a self-contained function unit. When parallelism is needed, a natural process is to reuse the existing function unit as a building block and run it in parallel to achieve speed and performance. Such an easy and straightforward approach normally works well at the beginning. Let's call this pattern as "Parallel Sequential Unit".

However, when the sequence is complex, especially multiple IO or inter-service calls are involved, parallelly run of the function unit, in most of the cases, is not optimized. Because normally there are more efficient ways to do batch IO or inter-service call. But doing this will break the existing self-contained units.

The solution for such a problem is to break the sequential operation into multiple stages. Within each stage, batch IO or inter-service call is utilized as optimization and concurrency happen at this level. The stages are chained together by producer-consumer pattern. The output of a previous stage is fed as the input of the next stage in the chain. Let's call this pattern as "Batch Stage Chain". Such a pattern in many cases has better scalability, throughput, and parallelism than the Parallel Sequential Unit pattern.

Variety implementation of the Batch Stage Chain pattern already exist nowadays, but programmers may not be aware of it from a generic point of view. This booth tries to summarize this pattern, and visualize its behavior in comparison with the Parallel Sequential Unit pattern. So that programmers have an explicit impression of the pattern, and could adopt it at an early phase of performance-critical projects.

### I/O Aggregator Pattern
WIP

### Rolling Poller Window Pattern
WIP

### Systematic Timeout Pattern
WIP
