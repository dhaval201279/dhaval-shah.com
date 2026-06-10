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
thumbnail: "images/wp-content/uploads/2026/05/cross-cloud-data-highway-throughput.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway-throughput.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway-throughput.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
In [Part 1](https://www.dhaval-shah.com/cross-cloud-data-highway-architecture/) we established RICS's foundational architecture — a stateless, chunk-driven pipeline with a hard separation between the data plane and the metadata-driven control plane. That foundation deliberately deferred one critical question:

> Once the architecture is right, how do you make it fast? 

Part 2 answers that. We dissect the two mechanisms that directly govern throughput — S3 ranged reads and Azure block uploads — and then examine how chunk sizing, bounded parallelism, and a closed-loop concurrency controller work together to keep the pipeline moving efficiently under real production conditions.

# Why Throughput is a Design Decision and not an after-thought
Most engineers treat throughput as something you measure after writing code. In RICS, throughput is designed-in from the first line of architecture. Two levers control it: 
1. Chunk size
2. Concurrency

# S3 Ranged Reads - The Foundation
A ranged read means: _'Give me bytes X to Y of an object — not the whole file.'_ It is an HTTP Range header feature that [S3](https://aws.amazon.com/s3/) supports natively.


``` http
Range: bytes=0-67108863   // first 64 MB
Range: bytes=67108864-134217727  // next 64 MB
```

## Why Ranged Reads
- Without ranged reads: **_failure cost = O(file size)_**. Restart from byte 0.
- With ranged reads: **_failure cost = O(chunk size)_**. Retry only the failed 64 MB.
- Parallelism becomes possible — multiple chunks can be in-flight simultaneously
- JVM heap stays bounded — you wont need to load entire 1 TB of file into memory

## S3 SDK Mechanisms (and Why RICS Chose Raw GetObject)
[AWS SDK v2](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/home.html) offers multiple read mechanisms. RICS deliberately chose the lowest-level option:
- _GetObject with Range header_ — chosen for deterministic chunk control and cross-cloud streaming
- _S3TransferManager_ — rejected because it downloads to local disk and limits resume control
- _Presigned URLs_ — useful for browser/external systems, not streaming pipelines

Considering design considerations outlined in [Part-1](https://www.dhaval-shah.com/cross-cloud-data-highway-architecture/), I needed deterministic chunk control - and hence _GetObject with Range header_ was the best choice

# Azure Block Blobs — Atomic Assembly
[Azure Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/) offers - 
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

Until commit: 
- The blob does not exist
- No partial visibility
- No corruption risk
- This maps perfectly to S3 ranged reads

# Chunk Size Tuning - The Critical Knob

| Chunk Size | Throughput | Retry Cost   |  Azure Blocks Used
| :--------: | :--------: | :-----: | :-----: |
| 16 MB | Lower | Very Low | High (~625 for 10 GB) |
| 64 MB | Balanced | Low | ~160 for 10 GB |
| 256 MB | High | Higher | ~40 for 10 GB |

For large files chunk size must be chosen to stay within the cap of Azure's block blob i.e. No. of blocks per blob and Maximum size of Blob

# Parallel Chunk Processing with Backpressure
RICS processes multiple chunks concurrently using [Project Reactor's](https://projectreactor.io/) _flatMap_ operator with a concurrency cap. This is not just _parallel_ execution — it is demand-driven, backpressure-aware parallelism.

``` java
  Flux
    .fromIterable(chunks)
    .flatMap(chunk -> transferChunk(job, chunk), concurrencyController.getConcurrency()) 
    .then(commitBlob(job))         
```

Key insights - 
1. **_flatMap_** with a concurrency limit is the correct operator for I/O parallelism — not _parallel()_ which is CPU-oriented. 
2. Reactor ensures demand propagates backward: if Azure slows down, S3 reads slow automatically and this in turn ensures app's memory footprint stays bounded

# Adaptive Concurrency
RICS implements adaptive concurrency through a feedback loop:
- If error rate exceeds 10%: decrease concurrency (halve it)
- If average latency is below 200 ms and no errors: increase concurrency (+2)
- If throttling (_HTTP 429/503_) is detected: immediately decrease concurrency

# Latency Characteristics
RICS is a throughput-optimized system, not a latency-optimized one. The primary latency drivers are:
- Cross-cloud network latency (AWS → Azure egress path)
- Chunk size — larger chunks = fewer round trips but higher commit latency

# Conclusion
Throughput in RICS is not a function of raw network bandwidth alone — it is an emergent property of three overlapping decisions: 
1. Right chunk size to balance retry cost against parallelism granularity
2. _flatMap_ with a concurrency cap to achieve parallel I/O without increasing memory footprint 
3. Adaptive Concurrency Controller that continuously adapts to what AWS and Azure are actually tolerating at that moment

[Part 3](https://www.dhaval-shah.com/cross-cloud-data-highway-high-resiliency/) takes this further: we move from how fast it runs to how well RICS survives failure by examining the resiliency mechanisms - So stay tuned!
