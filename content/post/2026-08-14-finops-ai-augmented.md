---
title: "Three Azure Cost Leaks - And the Analysis Process That Found Them"
author: Dhaval Shah
type: post
date: 2026-08-14T01:00:50+00:00
url: /finops-azure-ai-review/
categories:
  - finops
  - operations
  - ai-augmented-se
tags:
  - finops
  - operations
  - ai-augmented-se
thumbnail: "images/wp-content/uploads/2026/07/sre-gc-ai-review.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/07/sre-gc-ai-review.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/07/sre-gc-ai-review.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
This is the fifth post in a series on AI-augmented software engineering across the disciplines that matter most for production grade enterprise systems. The earlier posts covered ground that's probably more familiar with software engineering fraternity:

1. [Three Fintech Architecture Post-Mortems](https://www.dhaval-shah.com/fintech-post-mortem-ai-review/)
2. [The GC Summary Report Wasn't Wrong](https://www.dhaval-shah.com/gc-comparison-ai-review/)
3. [Same JSON Storage Problem, Different Database](https://www.dhaval-shah.com/db-optimization-ai-review/)
4. [A Black Friday Incident That Took 9 Days to Resolve](https://www.dhaval-shah.com/sre-gc-ai-review/)

This post covers different ground.Every previous post dealt with something visibly broken - an alert firing, a query degrading, an app crashing. [FinOps](https://www.finops.org/) problems don't usually announce themselves. They accumulate quietly in a billing dashboard that most engineering teams look at once a quarter, if at all!

The three cost patterns in this post came from [Azure](https://portal.azure.com/) infrastructure, where I was involved as Consultant in assessing and optimizing their cloud bills — a VMSS-based payment API, an ADF pipeline moving data from on-premises to Snowflake via Blob Storage, and the staging layer between them. None of the decisions that created these cost patterns were wrong in isolation. The VMSS autoscaling threshold made sense given what we knew about the traffic profile. The Integration Runtime sizing was conservative by design. The Blob Storage staging data not being cleaned up was an ownership gap, not negligence.

What's interesting about FinOps analysis is precisely this: the cost inefficiency usually lives in the gap between a reasonable local decision and its cost at scale. The analysis process I'll walk through is less about finding mistakes and more about asking "what does this decision actually cost, and is that cost proportional to the value it's delivering?"

I'll cover three patterns, show the cost calculation for each, and share the structured prompt I used to work through each analysis systematically.

# 1. VMSS autoscaling - the instance that wasn't ready yet

**The infrastructure:** A [VMSS](https://learn.microsoft.com/en-us/azure/virtual-machine-scale-sets/)-backed payment API processing 750+ TPS at peak, deployed on *Standard_D4s_v3* instances (4 vCPU, 16GB RAM), autoscaling configured to trigger scale-out at 70% average CPU utilisation across the scale set.

**The decision that made sense:** 70% CPU was chosen as the trigger to give headroom before instances saturated. Scale out before you're in trouble. That's reasonable autoscaling philosophy.

**The cost pattern it created:** The application had a bootstrap sequence - Spring context initialisation, cache
warm-up, connection pool establishment - that took ~90 seconds before an instance was accepting live traffic at its intended capacity. The VMSS health probe would mark an instance healthy before it was actually warm, because health probe passes liveness, not readiness under load.

**What this meant in practice:** when a CPU spike triggered scale-out, new instances joined the load balancer before they were ready to handle 750 TPS. Traffic routed to them caused higher latency responses. The autoscaler, seeing high CPU utilization (partly because new instances were themselves under cold-start load), triggered another scale-out event. By the time all instances were warm and load had distributed, the scale set was running 2-3 instances more than the traffic profile actually needed. And the scale-in cooldown period kept them warm for 10 more minutes after load had settled.

**The cost calculation:**

```
Instance type:  Standard_D4s_v3
Hourly rate:    ~₹13.40/hr per instance (East US pricing, Pay-As-You-Go)
Over-provision: 2 extra instances × 10-minute cooldown overhang
                = 2 × (10/60) hrs per scale event
Scenario:       5 scale events per business day (peak traffic pattern)

Extra cost per day:
  2 instances × (10/60) hr × ₹13.40/hr × 5 events = ₹22.33/day

Monthly:  ₹22.33 × 22 business days = ₹491/month

At USD equivalent (~₹84/USD): ~$5.85/month
```

That number looks extremely small. Scale it: a fleet of 8 VMSS applications with similar traffic profiles, the same autoscaling mis-calibration, and the calculation becomes ~₹3,928/month (~$46.76/month) in cooldown overhang alone - before accounting for the additional cost of scale events triggered by cold-start CPU pressure inflating
the autoscaler's view of actual load.

**What the analysis surfaced:** The right fix wasn't a different CPU threshold. It was decoupling bootstrap latency from scale-out readiness. At implementation level, we replaced the VMSS health probe with a custom readiness endpoint that returned healthy only after the warm-up sequence completed, and reducing the scale-out cool down (which could be shorter, since false-positive scale events would no longer occur from cold-start CPU spikes).

The threshold of 70% itself was reasonable. The assumption baked into it - that an instance at 70% load means a new instance would immediately contribute capacity - was the part that didn't hold.

# 2. ADF pipeline — paying for headroom that the data didn't need

**The infrastructure:** A daily [ADF pipeline](https://learn.microsoft.com/en-us/azure/data-factory/introduction) moving data from two sources - 300K [CosmosDB](https://learn.microsoft.com/en-us/azure/cosmos-db/) documents and 1 million+ rows across CSV files in [Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/) - transforming and loading into Snowflake. [Integration Runtime](https://learn.microsoft.com/en-us/azure/data-factory/concepts-integration-runtime) configured with 32 Data Integration Units (DIUs).

**The decision that made sense:** 32 DIUs was chosen to ensure the pipeline completed within the overnight batch window. More DIUs means more parallelism, faster throughput, safer SLA. For a once-daily batch with a hard completion deadline before business hours, "size up to be safe" is a defensible call.

**The cost pattern it created:** ADF charges per DIU-hour consumed per Copy Activity. A pipeline that processes
300K CosmosDB documents and 1M CSV rows at 32 DIUs completes the copy activities in approximately 45 minutes. The same pipeline at 8 DIUs - which is appropriate for this data volume based on ADF's own sizing guidance of 4-8 DIUs per TB for typical structured data - would complete in approximately 2.5-3 hours. Both are well within a reasonable overnight window.

**The cost calculation:**

```
ADF DIU pricing:     $0.25 per DIU-hour (standard IR, Azure region pricing)
Current config:      32 DIUs × (45 min / 60) = 32 × 0.75 = 24 DIU-hours per run
Optimised config:    8 DIUs × (2.5 hrs) = 8 × 2.5 = 20 DIU-hours per run

Current monthly:     24 DIU-hours × $0.25 × 30 days = $180/month
Optimised monthly:   20 DIU-hours × $0.25 × 30 days = $150/month

Saving: $30/month

But the actual saving is larger than this calculation suggests. At 32 DIUs, the copy activities scale-out linearly - CosmosDB read throughput and Blob Storage egress both have per-operation costs that scale with parallelism.

CosmosDB read at 32 parallel readers: ~12,000 RU/s burst at peak
CosmosDB read at 8 parallel readers:  ~3,000 RU/s sustained
CosmosDB pricing:                      $0.008 per 100 RU/s per hour

Monthly CosmosDB RU cost difference:
  32 DIUs: 12,000 RU/s × $0.008/100 RU × 0.75hr × 30 = $21.60/month
  8 DIUs:   3,000 RU/s × $0.008/100 RU × 2.5hr  × 30 = $18.00/month

Total combined saving: ~$33.60/month
```

Again, one pipeline, $33.60/month - unremarkable. Across a data platform with 12 daily ADF pipelines similarly over-provisioned: ~$403/month, or ~$4,836/year.

**What the analysis surfaced:**
The right question wasn't **"can we reduce DIUs?"**
Intsead, it was **"what is our actual overnight batch window, and how many DIUs do we need to complete within stipulated timeframe?"**

Answer: window is 6 hours (midnight to 6am), the pipeline at 8 DIUs completes in under 3 hours. 32 DIUs was solving for a constraint that didn't exist in the actual operating environment.

**A secondary finding:** the ADF pipeline was set to retry 3 times on failure with a 30-minute wait between retries. With 32 DIUs, a transient CosmosDB 429 [(throttling error)](https://learn.microsoft.com/en-us/azure/cosmos-db/troubleshoot-request-rate-too-large?tabs=resource-specific) causes a full retry of the copy activity at 32 DIUs, burning another 24 DIU-hours. At 8 DIUs, the same transient error retries at lower throughput - which is actually less likely to cause 429s in the first place, since the pipeline is no longer bursting CosmosDB reads at 12,000 RU/s.

# 3. Blob Storage - the staging data nobody owns

**The infrastructure:** Azure Blob Storage used as an intermediate staging layer for an on-premises data warehouse migration to Snowflake. On-prem data extracted, landed in Blob Storage, picked up by Snowflake's external stage, loaded, confirmed.

**The decision that made sense:** keeping staging data after Snowflake load was never an explicit decision. It was an absence of a decision - no lifecycle policy, no cleanup job, no post-migration checklist that included "delete from staging after confirmed load." The data just accumulated.

**The cost pattern it created:** Six months of daily pipeline runs, each landing approximately 15GB of data
(1M+ rows at average row size ~15KB). 
No cleanup. 
Storage tier: Hot (the default, chosen for pipeline reliability - you want the data accessible
if a retry is needed).

```
Daily data landed:         ~15GB
Accumulation period:       180 days
Total staging data:        ~2.7TB

Hot tier pricing (LRS):    $0.018 per GB per month
Monthly storage cost:      2,700GB × $0.018 = $48.60/month
                           (and growing by ~450GB/month as pipeline continues)

What it should have been:
  Keep data for 7 days (sufficient for retry window and confirmation)
  Then delete or move to Archive tier

At 7-day retention:        ~105GB at any time
Monthly cost at Hot tier:  105GB × $0.018 = $1.89/month

Monthly saving:            $46.71/month
Annual saving if left:     $560.52/year (and growing as data accumulates)
```

**The compounding problem:** Hot tier also charges for read operations. Every ADF pipeline run that scans the staging container to identify new files touches all objects in the container - including the 180 days of already loaded data sitting there. At 2.7TB with typical list-and-check operations, that's additional egress and operation cost on data that should have been gone months ago.

**What the analysis surfaced:** A Blob Storage lifecycle policy - a single JSON configuration, no code, no
new infrastructure - moves data from Hot to Cool after 7 days and to Archive or deletion after 30 days. That's the fix. The analysis confirmed something obvious in retrospect: nobody had asked "who owns post-migration cleanup?" when the pipeline was designed, because the pipeline's job was to move data, not to manage what happened to the staging copy afterward.

# 4. The structured analysis process
Three cost patterns, three different root causes, one consistent observation: none of them were visible without asking a specific question about each resource.

The Azure Cost Management dashboard showed aggregate spend by resource group. It showed that VMSS, ADF, and Storage were the top three line items. It didn't show that VMSS was over-provisioning during cooldown, that ADF was using 4 times more DIUs than the data volume needed, or that Blob Storage was accumulating because of a missing lifecycle policy.

The structured prompt I used for each analysis asked 5 things:

1. What is this resource doing, and what does it cost at current configuration to do that?
2. Is the configuration sized for actual usage or for a worst-case assumption that the data doesn't support?
3. What is the specific decision or gap that created the current configuration - and was that decision made with cost visibility, or purely for operational safety?
4. What is a principled right-size for this resource given actual observed usage, and what would that cost?
5. What would I expect to see in cost reports over the next 30 days if the right-sizing was applied - expressed as a number I can verify, not "lower cost"?

The fifth question is the one that makes the analysis actionable rather than theoretical. A cost saving estimate that can't be verified in the next billing cycle is a guess. One that produces a specific number to check against is a commitment - and it's the difference between a FinOps recommendation and a FinOps engagement that closes.

The full prompt templates for each of the three analyses are in the [GitHub repo]().


# Conclusion
Three infrastructure components. Three cost patterns. Total combined saving across one platform: approximately $80/month, or ~$960/year.

Unremarkable at the scale of a single platform. Patterns that repeat across every Azure deployment in an organisation - VMSS autoscaling everywhere, ADF pipelines sized conservatively across a data platform, staging containers without lifecycle policies - and the number becomes meaningful quickly.

The more important observation: none of these findings required a dedicated FinOps tool or a specialist. They required someone asking "what does this configuration actually cost relative to what it's doing?" against each resource that appeared as a significant line item in the billing dashboard. That's replicable without new tooling, which is the point.

P.S. — The three prompt templates used across this analysis are available in the [se-ai-templates](https://github.com/dhaval201279/se-ai-templates/tree/main/finops) repository on GitHub.