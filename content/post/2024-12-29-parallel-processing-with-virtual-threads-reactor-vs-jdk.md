---
title: Parallel processing with Virtual Threads - A comparative analysis
author: Dhaval Shah
type: post
date: 2025-01-01T02:00:50+00:00
url: /parallel-processing-virtual-threads-reactor-vs-jdk/
categories:
  - performance
  - jdk
  - spring
tags:
  - performance
  - jdk
  - spring
thumbnail: "images/wp-content/uploads/2025/01/vt-java-vs-reactor.jpg"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-java-vs-reactor.jpg)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-java-vs-reactor.jpg)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
My [previous article](https://www.dhaval-shah.com/parallel-processing-reactor-vs-jdk/) focussed on comparing solutions for performing parallel execution using  [Spring Core Reactor](https://spring.io/reactive) and [JDK 21](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html). This article will follow my previous article where I will provide comparative analysis of [Virtual Threads](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html) based execution for [Spring Core Reactor](https://spring.io/reactive) and [JDK 21](https://www.oracle.com/java/technologies/javase/jdk21-archive-downloads.html) based implementation.

Keeping the same use case that we referred to for comparing Spring Core Reactor & JDK based implementation, this article will be focussing on:
1. Limitations of [OS](https://en.wikipedia.org/wiki/Operating_system) / Platform threads
2. What are Virtual Threads
3. When to use Virtual Threads
4. Solutions with Virtual Thread based implementation
5. Comparative analysis of Virtual Thread based implementations

# 1. Limitations of OS / Platform threads
Constraints of [OS](https://en.wikipedia.org/wiki/Operating_system) / Platform threads:
- Lifecycle of Platform threads is **resource intensive**
- They can handle only limited number of concurrent threads without degrading performance, as they rely on underlying OS capabilities
- Blocking operations hold up OS threads, **limiting the scalability** of application
- Platform thread management incurs expensive context-switching which in turn reduces the efficiency of the application

Contemporary applications that are throughput and latency sensitive, demand high concurrency without overwhelming system resources

# 2. What are Virtual Threads
Virtual Threads, a feature of [Project Loom](https://openjdk.org/projects/loom/) in JDK 21 offers lightweight, scalable threads that are managed by [JVM](https://en.wikipedia.org/wiki/Java_virtual_machine). Unlike OS threads, they are cheap to create, block, and manage, making them ideal for high-concurrency applications.

Salient Features from performance engineering standpoint:
- Low Cost : Millions of virtual threads can be created without significant memory / CPU overhead
- Blocking friendly - Blocking a virtual thread doesn't block OS threads, allowing more efficient resource utilization

# 3. When to use Virtual Threads
For any thread within an application, its primary task is to perform CPU or I/O bound operations. While virtual threads can be used for both type of operations, one should be mindful of the fact that Virtual Threads are more appropriate for I/O intensive operations. Having said that, it can still be used for CPU intensive operations, but the benefits might not be proportionate to the gain we may see in I/O bound operations.

# 4. Solutions with Virtual Threads based implementation
## 4.1 JDK Based implementation with Virtual Threads
Excerpt from [actual code](https://github.com/dhaval201279/reactivespring/blob/master/src/main/java/com/its/reactivespring/mc/ethbatch/VirtualThreadJdk21ApplicationWithIsolatedExceptionForEachParallelThread.java)

```java
  public static void withFlatMapUsingJDK() {
    ...
    var virtualThreadExecutor = Executors.newThreadPerTaskExecutor(
        Thread
          .ofVirtual()
          .name("jdk21-vt-", 0)
          .factory()
    );

    try (virtualThreadExecutor) {
        // Submit tasks for parallel processing
        List<CompletableFuture<Void>> futures =
          users
            .stream()
            .map(user -> CompletableFuture.runAsync(() -> {
                try {
                    log.info("Processing user: {}", user);
                    processSomeBizLogic(user);
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    log.error("Error occurred while processing user {}: {}", user, e.getMessage());
                    failureCount.incrementAndGet();
                }
            }, virtualThreadExecutor))
            .toList(); // Collect CompletableFuture<Void> for each user

        // Wait for all tasks to complete
        CompletableFuture<Void> allOf = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));
        try {
            allOf.join();
        } catch (Exception e) {
            log.error("Error waiting for all tasks to complete: {}", e.getMessage());
        }
    }
    ...
  }
```

## 4.2 Spring Core Reactor Based implementation with Virtual Threads
Excerpt from [actual code](https://github.com/dhaval201279/reactivespring/blob/master/src/main/java/com/its/reactivespring/mc/ethbatch/VirtualThreadReactiveApplicationWithIsolatedExceptionForEachParallelThread.java)

```java
  public static void withFlatMapUsingJDK() {
    ...
    // Custom executor with virtual threads
    var virtualThreadExecutor = Executors.newThreadPerTaskExecutor(
      Thread
        .ofVirtual()
        .name("rx-vt-", 0)
        .factory()
    );

    try (virtualThreadExecutor) {
      Flux
          .fromIterable(objectList)
          .flatMap(obj ->
              Mono
                  .fromCallable(() -> {
                      log.info("Entering processUser in virtual thread: {}", obj);
                      processSomeBizLogic(obj);
                      log.info("Leaving processUser in virtual thread: {}", obj);
                      successCount.incrementAndGet();
                      return obj;
                  })
                  .doOnError(error -> {
                      log.error("Error occurred while processing user {}: {}", obj, error.getMessage());
                      failureCount.incrementAndGet();
                  })
                  .onErrorResume(error -> {
                      log.info("Skipping user due to error: {}", obj);
                      return Mono.empty(); // Skip errored objects
                  })
                  .subscribeOn(Schedulers.fromExecutor(virtualThreadExecutor)) // Use virtual threads
          )
          .doOnComplete(() -> {
              log.info("Processing completed");
              log.info("Success count: {}", successCount.get());
              log.info("Failure count: {}", failureCount.get());
          })
          .blockLast();
    }
    ...
  }
```

# 5. Comparative analysis of Virtual Thread based implementations
To ensure that the comparative analysis of Virtual Thread based solutions Platform thread based solutions is fair and wholistic, we will be keeping the same criteria as we had in my [previous article](https://www.dhaval-shah.com/parallel-processing-reactor-vs-jdk/) :
Concurrent processing of below number  of objects from a list
1. 1,00,000 objects
2. 2,50,000 objects
3. 5,00,000 objects

## 5.1 Comparing Virtual Thread based solution with Spring Core Reactor & JDK constructs

### 5.1.1 Total time spent in processing the entire list

[![ Time taken to process entire list  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-processing-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-processing-time-comparison.png)

As we can see in the above graph - Virtual Threads with JDK based implementation are super fast when compared with Spring Core Reactor. Also, the percentage increase in time for processing the entire list within Spring Core Reactor based application is exponentially increasing with an increase in no. of items in list.

### 5.1.2 Memory footprint
[![ Memory Footprint  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-memory-footprint-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-memory-footprint-comparison.png)

From the above graph one can infer -
- For 5 lac objects, JVM is required to allocate 33 times more memory in Old Gen for JDK based implementation when compared with Spring Reactor based implementation
- For 5 lac objects, peak memory utilized within Old Gen is 81 times more for JDK based implementation when compared with Spring Reactor based implementation

### 5.1.3 GC Metrics
As we have seen in my [previous blogs](https://deploy-preview-26--dhaval-shah.netlify.app/categories/jvm/), GC has a significant impact on the performance of an application. Here's how they compare:

#### 5.1.4 GC Pauses
This mainly indicates the amount of time consumed by STW GCs.

[![ GC Pause Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-gc-pause-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-gc-pause-time-comparison.png)

Comparatively speaking, in most of the cases GC pauses are higher for JDK based implementation. In spite of longer GC pauses for JDK based implementation, it does not have any significant impact on the latency of application.

#### 5.1.5 CPU Time
This shows the total CPU time consumed by the garbage collector.

[![ CPU Time  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-cpu-time-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-cpu-time-comparison.png)

It is evident from the above graph that even though JDK based implementation requires higher CPU time for its GC activities, it does not  have any negative effect on application performance.

#### 5.1.6 Object Metrics
This shows rate at which objects are created within JVM heap and rate at which they are promoted from Young to Old region.

[![ Object Creation and Promotion Rate  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-object-metrics-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-object-metrics-comparison.png)

 An interesting behavior that can be inferred - Even though object creation rate and promotion rate is significantly higher for JDK based implementation, it has minimal impact on performance of application.

 ## 5.2 Comparing Spring Core Reactor & JDK based solutions with / without Virtual Threads

 With JDK 21, developers have 2 options viz. Platform / Virtual threads when it comes to implementing asynchronous / concurrent workflows. Hence it becomes very important to understand - which would fare better when it comes to performance engineering. Logically it is quite evident, but below graph factually shows that Virtual threads based implementation are much more faster than Platform threads based implementation.

[![ Processing Time - Platform Vs Virtual Threads with JDK / Spring Core Reactor  ](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-vs-platform-comparison.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2025/01/vt-vs-platform-comparison.png)

# Conclusion

After objectively comparing both the solutions from this and the [previous](https://www.dhaval-shah.com/parallel-processing-reactor-vs-jdk/) article, it is quite apparent that :

- For Virtual threads based implementation, JDK should be one's obvious choice as they are **significantly** faster than Spring Core Reactor.
- For Platform threads based implementation, Spring Core Reactor is **relatively** faster than JDK based implementation

P.S - All the memory and GC related graphs shown above are prepared by using data from GC report generated by [GCEasy](https://gceasy.io/)