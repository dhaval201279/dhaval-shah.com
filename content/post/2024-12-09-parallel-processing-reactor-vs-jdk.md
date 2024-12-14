---
title: Parallel processing
author: Dhaval Shah
type: post
date: 2024-12-09T02:00:50+00:00
url: /parallel-processing-reactor-vs-jdk/
categories:
  - <>>
tags:
  - <>
thumbnail: "images/wp-content/uploads/2024/10/RN-FO-Quote-Merged-Photo-1.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2024/10/RN-FO-Quote-Merged-Photo-1.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/10/RN-FO-Quote-Merged-Photo-1.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

I am sure all of us would have implemented some kind of business logic, which is primarily tasked with heavy IO operations. Recently I got an opportunity to implement similar use case. Considering my obsession with [Performance Engineering](), I was able to discover an interesting finding about [parallel]() execution within [Java]() ecosystem

Cutting across the noise, to make this post more concise and succinct, I will just be focussing on following areas :
1. Understanding **What** part of requirements
2. High level overview of available solutions
3. Comparative analysis of available solutions from software & performance engineering standpoint 

## 1. Understanding what part of overall requirements
At a high level requirement definition comprised of -
1. Implement a business logic that reads object from list
2. For each object retrieved from list, perform IO intensive operations i.e. invoke bunch of downstream system APIs
3. Each object present in list and its corresponding business logic to be executed  is independent
4. Maximum number of downstream APIs to be invoked for each object in list is 10

## 2. High level overview of available solutions
Based on above requirement, solution seems to be pretty simple and straightforward - Iterate list and execute business logic in parallel for each of the objects in list. I being a typical [Java]() / [Spring]() engineer, obvious choice that I had considered -
- JDK based implementation
- Spring Core Reactor based implementation

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
   - _`Schedulers.boundedElastic()`_: Creates a bounded thread pool that can grow and shrink based on demand, preventing resource exhaustion

## 3. Comparative analysis of available solutions from software & performance engineering standpoint
By looking at above solutions, one of the obvious difference that one can make out - 
_`Flux`_ based implementation looks more cleaner and elegant, as it is devoid of lot of boiler plate code which is abstracted out very neatly by fluent APIs exposed by [Spring Core Reactor](). For all clean code aficionados this itself can be one of the compelling reasons to go with 2nd option. However, of late I have been a strong proponent of having more tangible criterion viz. Performance Engineering, Scalability, High Availability, Resiliency etc to compare available options and than conclude with acceptable tradeoffs.

So in quest of determining more tangible factors - I started delving bit deeper to compare both these options through the lens of performance engineering. As a result I captured metrics for processing entire list with -
1. 1,00,000 objects
2. 2,50,000 objects
3. 5,00,000 objects

### 3.1 Total time spent in processing entire list
[![ Time taken to process entire list  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/processing-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/processing-time-comparison.png)

As we can see in above graph - Spring Reactor takes relatively less amount of time to process entire list. Looking at the readings, JDK based implementation does not seem to exponentially increase with increase in size of list.

### 3.2 Memory footprint
[![ Memory Footprint  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/memory-footprint-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/memory-footprint-comparison.png)

From above graph one can clearly infer -
- For 5 lac objects, JVM is required to allocate 3 times more memory for JDK based implementation as compared to Spring Reactor based implementation
- Percentage increase w.r.t Peak Memory is exponentially increasing as the number of objects in the list

### 3.3 GC Metrics
As we have seen in my previous blogs, GC has a significant impact on performance of an application. Hence lets compare its key metrics

#### 3.3.1 GC Pauses

[![ GC Pause Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/gc-pause-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/gc-pause-time-comparison.png)

This mainly indicates amount of time taken by STW GCs. Comparatively speaking, in most of the cases GC pauses are higher for JDK based implementation.

#### 3.3.2 CPU Time

[![ CPU Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/cpu-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/cpu-time-comparison.png)

This shows total CPU time taken by garbage collector. Its evident from above graph that JDK based implementation is requiring higher CPU time for its GC activities 

#### 3.3.3 Object Metrics
[![ Object Creation and Promotion Rate  ](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/object-metrics-comparison.png.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2024/12/object-metrics-comparison.png.png)

This shows rate at which objects are created within JVM heap and rate at which they are promoted from Young to Old region. An interesting behavior that can be inferred - Even though object creation rate has been higher for Spring Reactor based implementation, object promotion rate is way less when compared with JDK based implementation

# Conclusion
After objectively comparing both the solutions using above critera, it is quite apparent that Spring Reactor based implementation is not only clean and elegant but also relatively better from performance considerations.

P.S - All the graphs shown above are prepared by using data from GC report generated by [GCEasy](https://gceasy.io/)

