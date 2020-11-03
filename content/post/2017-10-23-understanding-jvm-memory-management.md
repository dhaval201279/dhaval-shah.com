---
title: Understanding JVM Memory Management
author: Dhaval Shah
type: post
date: 2017-10-23T04:44:34+00:00
url: /understanding-jvm-memory-management/
categories:
  - JVM
  - Performance
tags:
  - jvm
  - performance
  - tuning

---

Everyone of us as Software Engineers would have experienced memory leaks, OOM errors in our Java/JVM applications? In order to dissect such issues it is extremely important to understand the whats' and hows' of JVM memory and its management.

# JVM - Memory Management

One of the many strengths of the JVM is that it performs automatic memory management. As we all know memory management is the process of allocating objects, determining when those objects are no longer needed; thereby de-allocating the memory used by those objects and making it available for future allocations

This automatic management of memory within JVM is performed by a system called **Garbage Collector**.

Let's see how the automatic memory management actually works?

JVM has what is called as heap - lets consider this as one rectangle for the sake of understanding. And objects instantiated within application are allocated chunks of contiguous space. As more objects get instantiated, more allocations happen in the heap. And eventually it will become full.  There are also some object that are external to the heap and are in use by the execution state of the java application e.g. space allocated to static variables, Java threads, JNI references and other JVM internal structures. These set of references is called as Root Set.

[![JVM Memory - Root Set](http://dhaval-shah.com/wp-content/uploads/2017/10/Allocation.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/Allocation.png)

It is quiet obvious that the objects that are directly reachable from the root set (highlighted in light orange) must be kept in the heap since the program's execution state can access them through the root set. It also makes sense that the objects reference from those directly reachable objects should also be kept in the heap since those objects are also accessible from the root set. All the objects in light orange color here are  reachable directly or indirectly through the root set so these are considered alive and hence should not be removed from the heap. Rest of the objects which are not reachable by traversing through the route set are not needed. And hence Garbage Collector (which we will try to understand in subsequent section) will remove those objects and the space occupied by them is reclaimed.

[![JVM Memory - Free Spaces](http://dhaval-shah.com/wp-content/uploads/2017/10/Collect.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/Collect.png)

And then the alive objects can be moved towards one side of the heap leaving contiguous free space towards the end. This part of the process is called  Compaction.

[![JVM Memory - Free space pointer](http://dhaval-shah.com/wp-content/uploads/2017/10/Compact.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/Compact.png)

# Garbage Collector - What, Why and How?

Memory heap in JVM is mainly divided into 3 generations (till Java 7) based on the age of objects -

1.  Young - It is mostly small and is garbage collected frequently
2.  Old - It is larger and is collected infrequently. It also fills up relatively slower
3.  Permanent Generation - It contains meta data comprising of Classes and methods that are required by JVM

[![JVM memory divisions](http://dhaval-shah.com/wp-content/uploads/2017/10/JVM-Heap-Classification.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/JVM-Heap-Classification.png)

Each generation holds object of different age ranges. The age of each object in Young gets incremented each time the space is swept over by a garbage collector. Objects that reach a predefined threshold in terms of its age are moved to the old generation.

The young generation consists of Eden and 2 survivor spaces. Most objects are initially allocated in Eden and the surviving objects are moved to one of the survivor spaces. One of the survivor space is empty at any given time and it serves as the destination of any live objects remaining during the next young collection. The other survivor space contains objects that survived the previous collection but are still too young to be promoted into the old space. Objects are copied between survivor spaces in this way until they are old enough to be copied to the old generation. This illustrates the allocation and promotion of objects in the JVM heap. Allocations happen in the young generation and the objects that grow older than a threshold age which means that the objects that survived a certain number of garbage collections in the young generation are moved to the old generation.

When the young generation becomes full minor collection is induced that collects only the young generation. The old generation is not collected with minor collections. Since the young generation space is usually small these collections are usually fast. When the old generation fills up JVM initiates full collections (aka major collections) in which the entire heap is collected. Major collections last longer than the minor collections because these need to go  over the objects in the entire heap.

Prior to Java 8 it had a third generation called the 'permanent generation'. The JVM uses an internal representation for Java classes and their metadata such as

1.  Methods and field symbols
2.  Byte codes of methods and
3.  Static field values etc

In Java versions prior to Java 8 the classes and their metadata along with intern strings are stored in the permanent generation. The space of this generation is contiguous with the Java heap. Since JDK 8 the JVM does not have the permanent generation. It has been replaced with a new space called the Metaspace.

[![JVM memory - Metaspace](http://dhaval-shah.com/wp-content/uploads/2017/10/Metaspace.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/Metaspace.png)

Classes and their metadata are stored in this new space. Unlike Perm generation it's not contiguous within the Java heap. Most important aspect of Metaspace, **it is allocated out of native memory and since it is allocated out of native memory the maximum space available to it is the whole of system memory**. We can still control it by using the JVM option _MaxMetaspaceSize_ so as not to allow classes in a single JVM process to accidentally use up the entire system memory If _UseCompressedOops_ is turned on and _UseCompressedClassesPointers_ is used, then class metadata is stored in logically 2 different areas of native memory. The size of the region can be set with _CompressedClassSpaceSize_ and by default it is 1 GB.

# Various Garbage Collectors available in the JVM

[![Types of GC](http://dhaval-shah.com/wp-content/uploads/2017/10/Garbage-Collectors.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/Garbage-Collectors.png)

Above picture gives a quick glimpse of the garbage collectors available in the JVM. These garbage collectors implement different algorithms and operate on different memory spaces in the JVM

## Young Generation Collection

They are generally executed using a mechanism called copying collection. Also known as minor collections or stopped keyword collection which means that application threads are paused during collection process.

1.  Serial - uses single GC thread
2.  Parnew - uses multiple GC threads
3.  ParallelScavenge - uses multiple GC threads

## Old Generation Collection

They are variants of mark sweep compact technique which means that live objects in the heap are marked by traversing to the object graph non marked objects are collected and the remaining objects are moved together to one side of the heap

1.  Serial Old - stopped keyword collection that uses a single thread
2.  Concurrent Mark-Sweep (CMS) - concurrent low pause collector
3.  Parallel Old - uses multiple GC threads

## G1 Garbage Collector

Salient features -

1.  Low latency collector for multi core machines
2.  Low GC pauses
3.  Is mainly meant for large heaps .
4.  Is used to collect Young and Old generations.

## JVM Options for GC

| GC Name   | Type     |
| --------  | -------- |
| UseSerialGC | Serial + Serial Old |
| UseParNewGC | Parnew + Serial Old |
| UseConcMarkSweepGC | Parnew + CMS + Serial Old |
| UseParallelGC | Parallel Scavenge + Parallel Old |
| UseG1GC | G1 GC for both the generations |

# How GC works

Lets try to understand how objects are moved to Old Generation through GCs. The process of execution is as below 
1. New objects are created in Eden space. Initially both the survivor spaces are empty 
   [![](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-1.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-1.png) 

2. When Eden space fills up, a minor GC is triggered. After the GC, surviving objects are moved to one of the Survivor space 
   [![](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-2.png "GC Step 2")](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-2.png)
   
3. Once Survivor space is full, surviving objects are moved to other Survivor Space
   [![](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-3.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-3.png) 

4. Objects in Survivor space that have crossed threshold age are moved to Old Generation. 
As per the below image threshold age is 8. 
   [![](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-4.png)](http://dhaval-shah.com/wp-content/uploads/2017/10/GC-Step-4.png)  

# Conclusion

So we gathered basic understanding of JVM memory, its categorization, types of GCs and how GC works w.r.t JVM heap. In subsequent article we will try understanding different tools that can be used for collecting diagnostic data and approaches to dissect memory issues within an application - So stay tuned !