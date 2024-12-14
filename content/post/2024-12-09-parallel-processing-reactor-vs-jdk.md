---
title: Parallel processing - A comparative analysis
author: Dhaval Shah
type: post
date: 2024-12-13T02:00:50+00:00
url: /parallel-processing-reactor-vs-jdk/
categories:
  - performance
  - jdk
tags:
  - performance
  - jdk
thumbnail: "images/wp-content/uploads/2024/12/industrial_pipes.jpg"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/industrial_pipes.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/industrial_pipes.jpg)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

I am sure all of us would have implemented some kind of business logic, which is primarily tasked with heavy IO operations. Recently I got an opportunity to implement similar use case. Considering my obsession with [Performance Engineering](https://en.wikipedia.org/wiki/Performance_engineering), I was able to discover an interesting finding about [parallel](https://en.wikipedia.org/wiki/Parallel_computing) execution within [Java](https://www.java.com/en/) ecosystem

I'll keep this concise and focus on three main areas:
1. Understanding **What** part of requirements
2. High level overview of available solutions
3. Comparative analysis of available solutions from software & performance engineering standpoint 

## 1. Understanding what part of overall requirements

At a high level, the requirement were:
1. Implement a business logic that reads objects from a list.
2. For each object, perform IO-intensive operations by invoking multiple downstream system APIs.
3. Each object's logic is independent of the others.
4. A maximum of 10 downstream APIs to be invoked per object.

## 2. High level overview of available solutions

The solution seemed straightforward - iterate the list and execute the business logic in parallel for each object. 
As a [Java](https://www.java.com/en/) / [Spring](https://spring.io/) engineer, I considered two options:
- [JDK 21](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html)based implementation
- [Spring Core Reactor](https://spring.io/reactive) based implementation

### 2.1 JDK Based implementation
Excerpt from [actual code](https://github.com/dhaval201279/reactivespring/blob/master/src/main/java/com/its/reactivespring/mc/ethoca/Jdk17ApplicationWithIsolatedExceptionForEachParallelThread.java)
```java
  public static void withFlatMapUsingJDK() {
    ...
    // Define thread pool
    ExecutorService executorService = Executors.newFixedThreadPool(AppConstants.PARALLELISM);

    // Submit tasks for parallel processing
    List<CompletableFuture<Void>> futures =
            objectList
                .stream()
                .map(anObject -> CompletableFuture.runAsync(() -> {
                    try {
                        log.info("Processing object: {}", anObject);
                        processSomeBizLogic(anObject);
                        successCount.incrementAndGet();
                    } catch (Exception e) {
                        log.error("Error occurred while processing object {} : {}", anObject, e.getMessage());
                        failureCount.incrementAndGet();
                    }
                }, executorService))
                .toList(); // Collect CompletableFuture<Void> for each object

    // Wait for all tasks to complete
    CompletableFuture<Void> allOf = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
    try {
        allOf.join();
    } catch (Exception e) {
        log.error("Error waiting for all tasks to complete: {}", e.getMessage());
    }

    // Shut down the executor
    executorService.shutdown();

    // Log results
    log.info("Success count: {}", successCount.get());
    log.info("Failure count: {}", failureCount.get());
    log.info("## Processing completed for object list size : {} ", objectList.size());
    ...
  }
```

**Overview of parallel execution using [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)**

1. Thread Pool (_`ExecutorService`_): Used _`Executors.newFixedThreadPool(4)`_ to process objects (read from list) in parallel with a fixed number of threads
2. Parallel Processing (_`CompletableFuture`_): Each object is processed in parallel using _`CompletableFuture.runAsync`_
3. Error Handling: Exceptions thrown during processing are caught within the _`CompletableFuture.runAsync`_ block, and failure counts are incremented. Errors are isolated to individual tasks and hence do not affect other threads.
4. Wait for Completion: Used _`CompletableFuture.allOf`_ to wait for all tasks to complete.

### 2.2 Spring Core Reactor based implementation
Excerpt from [actual code](https://github.com/dhaval201279/reactivespring/blob/master/src/main/java/com/its/reactivespring/mc/ethoca/ReactiveApplicationWithIsolatedExceptionForEachParallelThread.java)

``` java
  ...
  Flux
    .fromIterable(objectList)
    .parallel(AppConstants.PARALLELISM)
    .runOn(Schedulers.boundedElastic())
    .flatMap(anObject ->
            Mono.fromCallable(() -> {
                        log.info("Entering processSomeBizLogic from Callable : {} ", anObject);
                        processSomeBizLogic(anObject);
                        log.info("Leaving processSomeBizLogic from Callable  : {}", anObject);
                        successCount.incrementAndGet();
                        return anObject;

                    })
                    .doOnError(error -> {
                        log.error("Error occurred while processing object {} : {}", anObject, error.getMessage());
                        failureCount.incrementAndGet();
                    })
                    .onErrorResume(error -> {
                        log.info("Entering onErrorResume");
                        return Mono.empty();
                    }) // Skip the errored object
    )
    .sequential()
    .doOnComplete(() -> {

        log.info("Success count: {}", successCount.get());
        log.info("Failure count: {}", failureCount.get());
        log.info("$$ Processing completed for object list size : {} ", objectList.size());
    })
    .blockLast();
  ...
```

**Overview of parallel execution using [Flux](https://javadoc.io/static/io.projectreactor/reactor-core/3.5.3/reactor/core/publisher/Flux.html)**
1. With _`flatMap`_: This operator allows us to handle errors on per-item basis. Each item is wrapped in _`Mono.fromCallable`_ to enable the error handling flow.
2. Error Handling with _`doOnError`_ and _`onErrorResume`_:
    - _`doOnError`_: Logs the error details for the specific item
    - _`onErrorResume`_: Ensures that errors are skipped by returning an empty Mono. Errors in one thread will not affect the processing of other thread
3. Parallel Execution: The _`parallel(4).runOn(Schedulers.boundedElastic())`_ ensures that processing happens in parallel
   - _`Schedulers.boundedElastic()`_: Creates a bounded thread pool that can adjust based on demand, preventing resource exhaustion

## 3. Comparative analysis of available solutions from software & performance engineering standpoint

The main difference between these solutions is the [cleanliness](https://www.amazon.in/Clean-Code-Handbook-Software-Craftsmanship-ebook/dp/B001GSTOAM) and elegance of the _`Flux`_ based implementation, which avoids boilerplate code with its fluent APIs. However, it’s important to have tangible criteria to compare alternatives, such as performance, scalability, resilience etc.

So in quest of determining more tangible factors to compare alternatives under considerations - I started delving bit deeper to evaluate available options through the lens of performance engineering. Hence I measured performance of processing entire list in parallel with -

1. 1,00,000 objects
2. 2,50,000 objects
3. 5,00,000 objects

### 3.1 Total time spent in processing entire list

[![ Time taken to process entire list  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/processing-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/processing-time-comparison.png)

As we can see in above graph - Spring Reactor processes the list faster. JDK's performance doesn’t exponentially degrade with an increasing list size.

### 3.2 Memory footprint
[![ Memory Footprint  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/memory-footprint-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/memory-footprint-comparison.png)

From above graph one can clearly infer -
- For 5 lac objects, JVM is required to allocate 3 times more memory for JDK based implementation as compared to Spring Reactor based implementation
- Percentage increase w.r.t Peak Memory is exponentially increasing the list size in JDK

### 3.3 GC Metrics
As we have seen in my previous blogs, GC has a significant impact on performance of an application. Here's how they compare:

#### 3.3.1 GC Pauses

[![ GC Pause Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/gc-pause-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/gc-pause-time-comparison.png)

This mainly indicates amount of time taken by STW GCs. Comparatively speaking, in most of the cases GC pauses are higher for JDK based implementation.

#### 3.3.2 CPU Time

[![ CPU Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/cpu-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/cpu-time-comparison.png)

This shows total CPU time taken by garbage collector. Its evident from above graph that JDK based implementation is requiring higher CPU time for its GC activities.

#### 3.3.3 Object Metrics
[![ Object Creation and Promotion Rate  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/object-metrics-comparison.png.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/object-metrics-comparison.png.png)

This shows rate at which objects are created within JVM heap and rate at which they are promoted from Young to Old region. An interesting behavior that can be inferred - Even though object creation rate has been higher for Spring Reactor based implementation, object promotion rate is way less when compared with JDK based implementation

# Conclusion

After objectively comparing both the solutions, it is quite apparent that Spring Reactor based implementation is not only clean and elegant but also performs better.

P.S - All the graphs shown above are prepared by using data from GC report generated by [GCEasy](https://gceasy.io/)

