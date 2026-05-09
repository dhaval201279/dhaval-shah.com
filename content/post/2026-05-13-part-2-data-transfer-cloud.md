---
title: High Throughput - Cross-Cloud Data Highways
author: Dhaval Shah
type: post
date: 2026-05-11T02:00:50+00:00
url: /cross-cloud-data-highway-high-throughput/
categories:
  - azure
  - AWS
  - data-engineering
tags:
  - azure
  - data-engineering
  - AWS
thumbnail: "images/wp-content/uploads/2026/05/cross-cloud-data-highway.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

# Why Throughput is a Design Decision and not an after-thought
Most engineers treat throughput as something you measure after writing code. In RICS, throughput is designed-in from the first line of architecture. Two levers control it: 
1. Chunk size
2. Concurrency

# S3 Ranged Reads - The Foundation
A ranged read means: _'Give me bytes X to Y of an object — not the whole file.'_ It is an HTTP Range header feature that [S3]() supports natively.


``` http
Range: bytes=0-67108863   // first 64 MB
Range: bytes=67108864-134217727  // next 64 MB
```

## Why Ranged Reads
- Without ranged reads: **_failure cost = O(file size)_**. Restart from byte 0.
- With ranged reads: **_failure cost = O(chunk size)_**. Retry only the failed 64 MB.
- Parallelism becomes possible — multiple chunks can be in-flight simultaneously
- JVM heap stays bounded — you wont need to load a 10 GB file into memory

## S3 SDK Mechanisms (and Why RICS Chose Raw GetObject)
[AWS SDK v2]() offers multiple read mechanisms. RICS deliberately chose the lowest-level option:
- _GetObject with Range header_ — chosen for deterministic chunk control and cross-cloud streaming
- _S3TransferManager_ — rejected because it downloads to local disk and limits resume control
- _Presigned URLs_ — useful for browser/external systems, not streaming pipelines

Considering design considerations outlined in [Part-1](), I needed deterministic chunk control - and hence _GetObject with Range header_ was the best choice

# Azure Block Blobs — Atomic Assembly
[Azure Blob Storage]() offers - 
1. Block Blobs
2. Append Blobs
3. Page Blobs

RICS uses _block blobs_ because they support parallel uploads and atomic commit

## Thow-Phase Upload Protocol
### Phase 1 - Stage Blocks
Upload each chunk independently. Blocks are invisible to readers during staging.

``` java
  blockId = Base64(String.format("%06d", chunkIndex))
  blockBlobClient.stageBlock(blockId, inputStream, chunkLength)
```

### Phase 2 - Commit Block List
Atomically assemble the final blob from the ordered block list

``` java
  blockBlobClient.commitBlockList(orderedBlockIds)
```

Until commit: the blob does not exist. No partial visibility. No corruption risk. This maps perfectly to S3 ranged reads.

# Chunk Size Tuning - The Critical Knob

| Chunk Size | Throughput | Retry Cost   |  Azure Blocks Used
| :--------: | :--------: | :-----: | :-----: |
| 16 MB | Lower | Very Low | High (~625 for 10 GB) |
| 64 MB | Balanced | Low | ~160 for 10 GB |
| 256 MB | High | Higher | ~40 for 10 GB |

For large files chunk size must be chosen to stay within the cap of Azure's block blob i.e. No. of blocks per blob and Maximum size of Blob

# Parallel Chunk Processing with Backpressure
RICS processes multiple chunks concurrently using [Project Reactor's]() _flatMap_ operator with a concurrency cap. This is not just _parallel_ execution — it is demand-driven, backpressure-aware parallelism.

``` java
  Flux
    .fromIterable(chunks)
    .flatMap(chunk -> transferChunk(job, chunk), concurrencyController.getConcurrency()) 
    .then(commitBlob(job))         
```

Key insighs - 
1. **_flatMap_** with a concurrency limit is the correct operator for I/O parallelism — not _parallel()_ which is CPU-oriented. 
2. Reactor ensures demand propagates backward: if Azure slows down, S3 reads slow automatically and this in turn ensures app's memory footprint stays bounded

# Adaptive Concurrency
RICS implements adaptive concurrency through a feedback loop:
- If error rate exceeds 10%: decrease concurrency (halve it)
- If average latency is below 200 ms and no errors: increase concurrency (+2)
- If throttling (_HTTP 429/503_) is detected: immediately decrease

# Latency Characteristics
RICS is a throughput-optimized system, not a latency-optimized one. The primary latency drivers are:
- Cross-cloud network latency (AWS → Azure egress path)
- Chunk size — larger chunks = fewer round trips but higher commit latency
- Concurrency level — more parallelism less time

# Conclusion




^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We looked at the core of the problem context i.e. 
Imagine you need to move terabytes or petabytes of large files from [AWS S3](https://aws.amazon.com/s3/) to [Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/), and that too with 
1. Reliability &
2. High Throughput 

The naïve approach (download the whole file, upload whole file) collapses under real‑world realities: 
- Network glitches
- Process crashes
- Cloud throttling 
- Feasibility of buffering a 1 TB of file in memory

What looks like a simple file-copy operation - is a classic distributed systems problem with: 
1. Partial failures 
2. Idempotency
3. Data consistency across cloud

Recently I implemented it for one of my client and hence thought to unpack the foundational design principles that turned their existing fragile script into a production‑grade, resumable, and exactly‑once transfer pipeline.

Considering the magnitude and complexity of problem, this will be shared over a series of articles that will cover **core architecture**, **chunk‑level strategies**, **cross‑cloud consistency**, and **why deterministic identifiers** are key to this problem

# High Level Architecture

[![RICS - High Level Architecture ](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/RICS_Architecture_drawio.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/RICS_Architecture_drawio.png)

RICS (Resilient Inter Cloud Streaming) is structured as a streaming, pull-based transfer pipeline with a logical separation between the data plane and the control plane.

## Data Plane
- Streams bytes directly from AWS S3 to Azure Blob Storage using _ranged reads_ and _block uploads_
- No intermediate disk — data flows chunk by chunk through the JVM heap
- Stateless application that can be scaled horizontally

## Control Plane
- Metadata store tracks every file and every chunk: PENDING → IN_PROGRESS → COMPLETED / FAILED
- Drives retry, resume, backpressure, and throttling decisions
- Acts as the single source of truth for transfer state

The key architectural insight: data plane is streaming; control plane is metadata-driven. These two concerns are mutually exclusive.

# Core Design Principles
## 1. Chunked Transfers - Not a Single Stream
Files are split into fixed-size chunks (64–256 MB, tunable). Each chunk maps to:
- An S3 ranged GET (HTTP Range header)
- An Azure block upload (stageBlock API)

**This creates the foundation for resumability, parallel upload, and bounded retry.**
The chunk-to-block mapping is deterministic, which is what makes the whole system **idempotent**.

## 2. Stateless Workers
Workers carry zero in-memory state between operations. All progress is externalized to the metadata store ([Azure Cosmos DB](https://learn.microsoft.com/en-us/azure/cosmos-db/), or any other persistent store) managed by Control Plane. Since state lives within meta-data store, RICS application is - 
- Reliable : w.r.t data consistency and correctness
- Scalable : Throughput can be increased by adding more instances

## 3. Eventual Consistency with Correctness Guarantees
RICS doesn't chase strong cross-cloud consistency. Instead it guarantees:
- A file is visible to consumers only after atomic blob commit
- Partial uploads are never exposed
- Retries converge to the same correct final state

> RICS trade immediate consistency for throughput, but never expose corrupt data

# Technology Stack
1. JDK 21
2. Spring Boot 3.x
3. Spring Core Reactor
4. AWS SDK
5. Azure Blob SDK

# Key Trade-offs
- No real-time replication — this is batch-oriented, high data volume movement
- Double egress cost (AWS out + Azure in) — justified by reliability and control
- Eventual consistency across clouds — acceptable when commit is atomic

# Conclusion
In this article we took a stab at what problem RICS intends to solve along with its high level architecture, design principles and architectural trade-offs.

Architecture tells you what the system looks like. Performance tells you how fast it actually moves.
In [Part 2](), we will go under the hood of the two mechanisms that determine throughput: _S3 ranged_ reads and _Azure block blob_ uploads - So stay Tuned! 
