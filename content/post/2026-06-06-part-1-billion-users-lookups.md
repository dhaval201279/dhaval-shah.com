---
title: The Billion-User Trap
author: Dhaval Shah
type: post
date: 2026-06-06T02:00:50+00:00
url: /billion-user-trap-2-simple-apis/
categories:
  - architecture
  - system-design
tags:
  - architecture
  - system-design
thumbnail: "images/wp-content/uploads/2026/06/1-billion-user-kyc-blog-series-part-1.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-blog-series-part-1.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-blog-series-part-1.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
This multi part series walks through the real architectural decisions and real mistakes behind designing a globally distributed KYC User Profile system - a system that should be capable of serving billions of users at 10M+ RPD with sub-500 ms P99 latency.  It's mainly about weighing trade-offs and making architectural decisions.

**_Not theory. Not a tutorial. No "here's how Redis works."_**

If you already know what a [cache stampede](https://en.wikipedia.org/wiki/Cache_stampede) is, what [CDC](https://en.wikipedia.org/wiki/Change_data_capture) does, and why _OFFSET pagination_ is dangerous — this series is written for you. If you're a CTO or engineering leader who has watched a "simple" looking service quietly become your biggest operational liability - this series is especially for you.

Every part covers one assumption that looked reasonable on day one and became a production problem at scale. The database design that passed every review. The pagination that worked fine until it didn't. The cache layer that was added to fix latency and made almost no difference.

That's what makes them worth writing about.

# It Started With Two APIs
## The Lie We Tell Ourselves at the Beginning

The request sounded almost trivial.

A product manager walked into a planning meeting and wrote two bullet points on the whiteboard:
1. **API 1 :** List users with KYC status
2. **API 2 :** Fetch full user details including all accounts

Clean. Simple. Two endpoints.

A senior developer in the room said: **_"I can have a prototype running by end of week."_**

Nobody pushed back.

That's the first mistake.

## The Requirements That Seemed Harmless
The system was a KYC (Know Your Customer) profile service for a FinTech platform. Every user had:
1. A core identity record (First Name, Last Name, DOB, address)
2. KYC verification status and documents
3. Multiple financial accounts (savings, current, wallet — up to 4 per user)

The UI needed two things:

 - **API 1 :** User Listing Return a paginated list of users. Each record: User ID, First Name, Last Name, KYC status. No account details.
 - **API 2 :** User Detail Return the full profile for a selected user: complete KYC data, all linked accounts, everything.

A straightforward CRUD problem. Until the next sentence in the requirements doc.

## Then the Projected Numbers Arrived
**_Billions of users._**

**_10 million requests per day._**

**_Multi-region rollout — North America first, then APAC, EMEA._**

**_P99 latency: ≤ 500ms._**

Everything changed with these projected numbers. The features were still 2 simple GET APIs. But the entire conversation shifted:
- **"What tables do we need?"** became **"What are the dominant read patterns?"**
- **"Which DB framework we use?"** became **"How do we avoid hot partitions?"**
- **"Should we use Postgres or MySQL?"** became **"How do we handle cross-region consistency?"**

# The First Architectural Principle: Access Patterns Over Entities
Most engineers design storage around what data exists or is going to get persisted.

> **Experienced architects design storage around how data is accessed.**

The difference sounds subtle - but it is not.

**_"List users"_** and **_"Fetch user details"_** look similar on a whiteboard. In production they are fundamentally different problems

| Dimension | List Users | Fetch User Details   |
| :-------- | :-------- | :----- |
| Access type | Range scan / full table | Point lookup by ID |
| Data volume per request | Partial (KYC status only) | Full (KYC + all accounts) |
| Cache strategy | Shared, TTL-based | Per-user, invalidation-driven |
| DB pressure at scale | Index scans, pagination cost | Key-value lookup, predictable |
| Failure mode | Slow query + cascade | Cache miss + single DB hit |

**_Treating these two access patterns the same is the first architectural mistake._**

# What Breaks the System
Scale doesn't kill systems by itself.

**Hidden assumptions kill systems.** Scale just surfaces them faster.

The most dangerous assumptions w.r.t given use case:
1. *"Our users are evenly distributed."* They're not. 20% of users drive 80% of API calls.
2. *"Joins are fine - the database is fast."* Joins are fine until they run across 500 million rows.
3. *"Pagination is a UI concern."* Pagination is a database concern as well.

# The Architectural Decision Log Starts Here

Before a single line of code, before schema design, before choosing a cache: **document your access patterns**.

For this system, the two dominant reads are:

```
Read Pattern 1: GET /users?page=N&size=50
  - Frequency: Very high (every UI load, every export job)
  - Data: userId, name, kycStatus
  - Latency target: < 200ms P99
  - Consistency: Eventual OK (few seconds)

Read Pattern 2: GET /users/{userId}
  - Frequency: High but more targeted
  - Data: Full KYC + all accounts
  - Latency target: < 300ms P99
  - Consistency: Strong preferred (compliance context)
```

These two patterns drive every decision in this series. Database design, Cache topology, Pagination strategy, Consistency model. All of it flows from here.

# What Comes Next

In [Part 2](), we look at the database design - the normalized schema with elegant joins - and exactly where it breaks down before you hit 100 million users.
