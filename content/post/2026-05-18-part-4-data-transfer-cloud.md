---
title: Observable - Cross-Cloud Data Highways
author: Dhaval Shah
type: post
date: 2026-05-23T02:00:50+00:00
url: /cross-cloud-data-highway-observability/
categories:
  - azure
  - AWS
  - data-engineering
tags:
  - azure
  - data-engineering
  - AWS
thumbnail: "images/wp-content/uploads/2026/05/cross-cloud-data-highway-observability-2.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway-observability-2.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/05/cross-cloud-data-highway-observability-2.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
In the [previous article](https://www.dhaval-shah.com/cross-cloud-data-highway-high-resiliency/), we explored resiliency and fault tolerance as critical enablers of high‑volume inter‑cloud data transfer. Yet, even the most resilient system cannot succeed without deep visibility into its live operations. This final part of the series focuses on observability

# Observability - A First Class Requirement
RICS builds observability into the architecture with two pillars: 
1. Metrics
2. Structured logs

# Pillar-1 : Metrics
## Throughput Metrics
- **_files_transferred_total_** - counter, labelled by status (_SUCCESS_, _FAILED_, _DEAD\_LETTER_)
- **_bytes_transferred_total_** - gauge of cumulative bytes moved
- **_transfer_throughput_mbps** - derived metric
- **_transfer_backlog_depth_** - files in PENDING state

## Laltency Metrics
- **_chunk_transfer_duration_seconds_**
- **_file_transfer_duration_seconds_**
- **_s3_read_latency_seconds_**
- **_azure_write_latency_seconds_**
  
## Error & Reliability Metrics
- **_chunk_retry_count_** - how many chunks needed retries
- **_throttling_events_total_** - HTTP 429/503 from S3 and Azure
- **_dead_letter_files_total_** - files that exhausted retries
- **_commit_failure_total_** - blob commit failures

## Concurrency Metrics
- **_active_concurrent_chunks_** - real time in-flight chunk count
- **_concurrency_adjutment_events_** - adaptive controller decisions

# Pillar-2 : Structured Logs
RICS uses JSON-structured logging correlated by transferId. Every log line carries:

``` json
{
  "transferId": "550e8400-e29b-41d4-a716-446655440000",
  "fileId": "bigfile-20260523.bin",
  "chunkIndex": 37,
  "operation": "STAGE_BLOCK", 
  "durationMs": 145, 
  "status": "SUCCESS" 
}
```
## Log Levels by Scenario
- **_INFO:_** File start, chunk completed, file committed, file dead-lettered
- **_WARN:_** Chunk retry, throttling detected, concurrency decreased
- **_ERROR:_** Commit failure, unrecoverable error, dead-letter transition
- **_DEBUG:_** Byte range details, block ID, SDK response codes (disabled in prod by default)

# Alerting Strategy
| Alert | Condition | Severity   |  Action
| :--------: | :--------: | :-----: | :-----: |
| High Error Rate | > 5% chunks failing | P1 | Page on call |
| Throughput drop | < 50% baseline for 5 min | P2 | Investigate Throttling |
| Backlog spike | > 500 PENDING files | P2 | Investigate |
| Dead-letter growth | > 10 files/hour | P3 | Review Logs |
| Concurrency saturation | MAX for > 10 min | P3 | Tune configurations |

# Operational Runbooks
Every alert links to a runbook. Key runbooks for RICS:
- How to re-drive a dead-letter file: update metadata status to PENDING
- How to pause all transfers
- How to investigate a slow file: pull logs by transferId, compare chunk latencies

# Conclusion
Observability closes the loop on resilient inter‑cloud transfer by ensuring that every design choice is measurable, every anomaly is detectable, and every incident is actionable. With metrics, structured logs and runbook‑driven alerting, RICS empowers operators to maintain performance, reliability, and trust at scale. 

This four‑part series has walked through the essential dimensions of building a high‑volume, resilient inter‑cloud data transfer system. We began with [architecture and design fundamentals](https://www.dhaval-shah.com/cross-cloud-data-highway-architecture/), [laying the groundwork for high throughput pipelines](https://www.dhaval-shah.com/cross-cloud-data-highway-high-throughput/). In [Part 3](https://www.dhaval-shah.com/cross-cloud-data-highway-high-resiliency/), we focused on resiliency and fault tolerance, ensuring the system can withstand failures gracefully. Finally, in Part 4, we emphasized observability as a first‑class requirement, enabling operators to measure, monitor, and act with confidence in production.