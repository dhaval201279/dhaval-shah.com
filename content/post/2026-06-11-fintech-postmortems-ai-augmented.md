---
title: Three Fintech Architecture Post-Mortems - What AI-Augmented Review Would Have Caught
author: Dhaval Shah
type: post
date: 2026-06-11T02:00:50+00:00
url: /fintech-post-mortem-ai-review/
categories:
  - architecture
  - distributed-systems
  - ai-augmented-se
tags:
  - architecture
  - distributed-systems
  - ai-augmented-se
thumbnail: "images/wp-content/uploads/2026/06/fintech-architecture-ai-augmentation.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/fintech-architecture-ai-augmentation.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/fintech-architecture-ai-augmentation.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
Architecture decisions rarely break at design time. They break a year later, under double the load, when an unwritten assumption proves false. I’ve seen this play out in synchronous coupling, storage models, and service splits - while architecting and designing FinTech platforms. That’s why I now run every major design decision through an AI-augmented review process.

What follows are three post‑mortems from real payment systems - and the simple prompts that would have flagged each failure before it hit production.

# POST-MORTEM 1: The Synchronous Coupling That Ate Our SLA

## Incident Summary

**System:** Payment orchestration service at a high-throughput fintech platform

**Decision made:** Route all payment status checks through synchronous REST calls to a downstream fraud scoring service, co-located in the same data centre  

**Reasoning at the time:** "They're in the same DC - latency will be negligible. Synchronous is simpler to implement and triage. We'll add async later if needed."  

**What happened:** 
After launch, the fraud scoring service underwent an infrastructure upgrade. During a rolling restart window - planned, within documented SLA - the payment orchestration service's thread pool exhausted. Synchronous calls queued, timeouts cascaded upstream. The payment platform's p99 latency went from 45ms to 4,200ms. Not because the fraud service was down. Because it was *slow*.

The circuit breaker existed. It was misconfigured - the threshold was set for *error rate*, not *latency*. Slow doesn't trip an error-rate circuit breaker. We had tested failure. We hadn't tested *slowness*.

**Root cause:** The architecture decision had an assumption - "co-located = negligible latency penalty" - that was never written down, never challenged, and never tested as a failure mode. The assumption was reasonable.

**Remediation cost:** Couple of months of redesign - migration to async event-driven status propagation via [Kafka]().

**What an AI-augmented ADR would have caught:**  
The [failure mode analysis prompt](https://github.com/dhaval201279/se-ai-template/blob/main/adr/templates/01-failure-mode-analysis.md) surfaces exactly this: "What happens to upstream services if this dependency is slow rather than down?" The synchronous coupling assumption would have appeared in the trade-off matrix as a latency coupling risk - and the remediation (async event propagation) was already available in the architecture.

# POST-MORTEM 2: The Storage Model That Optimised Its Way Into a Corner
## Incident Summary

**System:** Payment transaction store for a high-volume e-commerce payments platform (Oracle Database, 1M+ rows, sustained 1K+ TPS at peak)

**The journey:** Three storage model decisions

**Time to full resolution:** Couple of man-months of development, testing, and Stage deployment

**Phase 1 - The original decision: Oracle JSON column storage**

The team chose to store payment transaction payloads as JSON in an Oracle `JSON` column. The reasoning was sound: transaction payloads varied by payment method - UPI, card, netbanking, wallet - each with a different field structure. A flexible JSON column 
avoided a wide, sparse relational table with dozens of nullable columns for fields that only applied to one payment method.

*What we assumed:* "JSON storage gives us schema flexibility. Oracle's JSON functions let us query inside the payload when needed."

*What we didn't fully evaluate:* Query patterns at 1,000+ TPS against deeply nested JSON in Oracle are materially different from query patterns at 100 TPS. Oracle's `JSON_VALUE` and `JSON_EXISTS` functions applied to 2–3 level nested paths - for example, extracting `$.paymentDetail.cardInfo.network` or filtering on `$.metadata.riskSignals.score` - do not use standard B-tree indexes. At low volume, response times were acceptable. As transaction volume crossed 500K rows, lookup queries began degrading. p99 query latency crossed 8 seconds.

**Phase 2 - The optimisation: virtual columns on JSON fields**

The team's response was technically reasonable: extract the most-queried JSON fields as Oracle virtual columns, index them, and route lookup queries through the indexed virtual columns rather than full JSON path expressions.

*What this looked like in practice:*

```sql
-- Virtual column extracting merchant category from nested JSON
merchant_category VARCHAR2(50) GENERATED ALWAYS AS
  (JSON_VALUE(payload, '$.paymentDetail.merchantInfo.category')) VIRTUAL;

-- Index on virtual column for reconciliation queries
CREATE INDEX idx_txn_merchant_cat ON payment_transactions(merchant_category);
```

Five virtual columns were added across two deployments, covering the most frequent lookup paths: merchant category, payment network,  risk score band, settlement currency, and transaction sub-type.

*Initial result:* Reconciliation query latency dropped from > 8s to < 400ms. The fix appeared to work.

*What emerged in production:*

1. **DML performance degradation under write load.** Virtual columns computed by `JSON_VALUE` add per-row evaluation overhead on every `INSERT` and `UPDATE`. At 1,000+ TPS with peak bursts, Oracle's redo log volume increased measurably.

2. **Query plan instability on multi-column predicates.** When queries filtered on two virtual columns simultaneously (e.g. `merchant_category` AND `payment_network`), Oracle's cost-based optimiser periodically chose a full-table scan over the indexed virtual columns - particularly after statistics jobs ran and recalculated selectivity estimates on the JSON-derived values.

3. **Operational brittleness during schema evolution.** When a new payment method (EMI via credit card) introduced a new nested field - `$.paymentDetail.emiInfo.tenure` - that needed a virtual column, the `ALTER TABLE` to add the virtual column on a live table with 800K+ rows caused a prolonged lock wait during a period of active write traffic.

**Root cause of Phase 2 failure:** Virtual columns on JSON fields solve the read problem by transferring the cost to writes and DDL operations. At low TPS, that transfer is invisible. At 1,000+ TPS the cost of the transfer became the new production problem.

**What an AI-augmented ADR would have caught:**
The [10 year regret test prompt](https://github.com/dhaval201279/se-ai-template/blob/main/adr/templates/03-ten-year-regret-test.md) directly surfaces this class of risk: "For each benefit claimed, what is the corresponding cost deferred or transferred?" Both costs are in Oracle's documentation. Neither was explicitly evaluated in the original ADR.

# POST-MORTEM 3: The Monolith We Called a Service

## Incident Summary

**System:** Core banking integration layer at a payments platform  

**Decision made:** Build the integration layer as a single Spring Boot service, with clear internal package boundaries by domain. "We can extract services later if we need to - the package structure will make it easy."  

**Reasoning at the time:** "Microservices at our current scale is operational overhead we can't afford. We'll modularise internally and split later."  

**What happened:**
The internal package structure held for a year. Then two things happened simultaneously: the engineering team grew from < 10 to > 20 engineers, and a regulatory requirement mandated that PCI-DSS scope be isolated from non-payment transaction processing.

The "clear package boundaries" had eroded. Cross-package imports had accumulated. The PCI isolation requirement meant the payment processing domain needed to be extracted. The extraction effort discovered that the payment and notification domains shared a database schema - which was designed for convenience and now had a structural coupling that team had never intended.

**Root cause:** The "extract later" assumption requires that the conditions for extraction remain stable. They never do. Team growth changes coupling decisions. Regulatory requirements impose extraction on a timeline that doesn't match engineering readiness. The decision to defer decomposition was reasonable; the decision to not track coupling metrics was the actual failure.

**Remediation cost:** A 6 month extraction programme to isolate PCI scope. Database schema decomposition required a 2 months of dual-write migration period with both services running simultaneously against a shared schema under active traffic.

**What an AI-augmented ADR would have caught:**  
The [Alternative Architecture Challenger prompt](https://github.com/dhaval201279/se-ai-template/blob/main/adr/templates/05-alternative-challenger.md) surfaces the specific failure mode: "What is your strategy for tracking coupling as the system grows? What metric will tell you when 'extract later' becomes 'extract urgently'?" It also surfaces the regulatory risk: "Are there compliance boundaries - PCI, GDPR, SOC 2 - that will eventually force decomposition regardless of your preference?"

# Conclusion
Three different systems, three different failure modes, one underlying pattern: every post-mortem above traces back to an assumption that was never made explicit.

The AI-augmented ADR process documented here doesn't eliminate that risk. What it does is make the assumption visible - early enough to evaluate it honestly, before the cost of being wrong gets compounded

The prompts are not the answer. They are a question generator. Your answers - drawn from your system's specific constraints, your team's operational reality, and your own production experience - are the record that matters.

All five prompt templates used across these post-mortems are available at [se-ai-templates](https://github.com/dhaval201279/se-ai-template/tree/main/adr/templates). Use them, adapt them, and if there is something that your architecture needs and is missing - open a PR with your scenario.

Stay tuned for next article!