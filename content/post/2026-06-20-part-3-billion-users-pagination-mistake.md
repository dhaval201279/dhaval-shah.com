---
title: Billion User Trap - The Pagination Mistake That Can Take Down Your Database
author: Dhaval Shah
type: post
date: 2026-06-22T02:00:50+00:00
url: /billion-user-trap-db-design/
categories:
  - architecture
  - distributed-systems
  - database-optimization
tags:
  - architecture
  - distributed-systems
  - database-optimization
thumbnail: "images/wp-content/uploads/2026/06/1-billion-user-kyc-db-design-part-2.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-db-design-part-2.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-db-design-part-2.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background

# The Query That Kills Production

It's one line of SQL. It is the default behavior in most ORMs.

And at scale, it silently destroys database performance.

```sql
SELECT user_id, full_name, kyc_status
FROM   users
ORDER  BY created_at DESC
LIMIT  50 OFFSET 5000000;
```

This is **offset-based pagination**. It works correctly. And it will become your worst problem once your user table crosses tens of millions of rows.

---

# What OFFSET Actually Does

Here's what most engineers believe **_OFFSET_** does:

> "Skip to the 5 millionth record and return the next 50."

Here's what the database actually does:

> "Scan and load the first **5,000,050** rows. Discard the first 5,000,000. Return the last 50."

And that too - **EVERY TIME**

The database must traverse every row before the offset to reach the rows you want. The deeper the page, the more work. **The more concurrent users paginating, the more overlapping full-scans running simultaneously.**

---

# The Performance Profile of OFFSET at Scale

On a 200M row users table (PostgreSQL, SSD-backed, well-tuned):

| Page Number (50 records/page) | OFFSET Value | Approx. Query Time (P99) |
|---|---|---|
| Page 1 | 0 | 4ms |
| Page 100 | 4,950 | 6ms |
| Page 1,000 | 49,950 | 18ms |
| Page 10,000 | 499,950 | 140ms |
| Page 100,000 | 4,999,950 | 1,800ms |

What happens while OFFSET query runs - It holds buffer pool pages and I/O bandwidth that your real-time user queries also need.

---

# Cursor-Based Pagination: The Fix That Actually Works

The correct approach is **keyset pagination** (also called **_cursor-based pagination_**).

> Instead of "give me page N," you ask: "give me 50 records that come after this specific record."

```sql
-- First page (no cursor)
SELECT user_id, full_name, kyc_status, created_at
FROM   users
WHERE  region = 'NA'
ORDER  BY created_at DESC, user_id DESC
LIMIT  50;

-- Subsequent pages (cursor = last record from previous page)
SELECT user_id, full_name, kyc_status, created_at
FROM   users
WHERE  region = 'NA'
  AND  (created_at, user_id) < (:lastCreatedAt, :lastUserId)
ORDER  BY created_at DESC, user_id DESC
LIMIT  50;
```

What changed:

1. No OFFSET - the WHERE clause positions the query directly in the index
2. The database uses the index to jump to the exact start position
3. Reads exactly 50 rows. Not 5,000,050.
4. Query time is **O(1)** with respect to page depth - **page 100,000 is exactly as fast as page 1**

The same query at page 100,000 now runs in **4ms** instead of **1,800ms**. Not 10% faster. 450X faster.

---
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
In the [first part](https://www.dhaval-shah.com/billion-user-trap-2-simple-apis/) we established the foundational principle that separates systems which survive scale from systems that collapse under it - **_design for access patterns, not for entities_**. 

In this second part we double click on the database design of the User Profile system and try to understand its impact from performance engineering and scalability standpoint. Specifically - why the schema that adheres to proper normalization, proper foreign keys, and proper indexes, becomes a latency problem.

# Every Architecture Diagram Is Pristine Before the First 100 Million Rows
Show me an ER diagram from a planning session and I'll show you something that may cause a production incident when your platform scales. Not because engineers are bad at design. Because the diagram was designed for the data they had or had to persist. Not the data they'd get and would be getting accessed.

The KYC User Profile system had a natural, normalized design. Clean and elegant!

``` sql
-- Users: core identity
CREATE TABLE users (
  user_id       UUID PRIMARY KEY,
  first_name     VARCHAR(255),
  last_name     VARCHAR(255),
  email         VARCHAR(255) UNIQUE,
  phone         VARCHAR(20),
  date_of_birth DATE,
  region        VARCHAR(10),
  created_at    TIMESTAMP,
  updated_at    TIMESTAMP
);

-- KYC: verification state and documents
CREATE TABLE kyc_records (
  kyc_id        UUID PRIMARY KEY,
  user_id       UUID REFERENCES users(user_id),
  status        ENUM('PENDING','VERIFIED','REJECTED','EXPIRED'),
  verified_at   TIMESTAMP,
  doc_type      VARCHAR(50),
  doc_reference VARCHAR(255),
  updated_at    TIMESTAMP
);

-- Accounts: financial accounts per user
CREATE TABLE accounts (
  account_id    UUID PRIMARY KEY,
  user_id       UUID REFERENCES users(user_id),
  account_type  ENUM('SAVINGS','CURRENT','WALLET','INVESTMENT'),
  account_no    VARCHAR(30),
  balance       DECIMAL(18,4),
  status        VARCHAR(20),
  opened_at     TIMESTAMP
);
```

And here is the query for User Detail API which is equally elegant:
``` sql
SELECT u.*, k.status, k.verified_at, k.doc_type,
       a.account_id, a.account_type, a.account_no, a.balance
FROM   users u
JOIN   kyc_records k ON k.user_id = u.user_id
JOIN   accounts a    ON a.user_id = u.user_id
WHERE  u.user_id = :userId;
```

This works for 100 users, 100,000 users. But beyond certain million no. of users, API starts showing signs of latency degradation - and that's because **database joins are worst enemies of scale**

# Why Joins Become Your Worst Enemy at Scale
A database JOIN is not a free operation. For a point lookup on a single _user\_id_ across three tables, with proper indexes, you're looking at **3 B-tree** traversals plus result merging. 

At low concurrency this seems fine.

But at 10 million requests per day (roughly 115 RPS average, with peaks easily hitting 500-1000 RPS), every millisecond of database CPU matters.

## The hidden cost of JOINs at scale
1. **Lock contention across tables :** When a KYC update happens, the _kyc\_records_ table takes a row-level write lock. Concurrent reads doing a JOIN that touches that row wait
2. **Buffer contention :** When the working set of data, your queries need is larger than the DB buffer size - pages are being loaded, evicted, and reloaded in a continuous cycle
3. **Query plan instability :** This may be due to frequent changes in your DB object _statistics_
4. **The N+1 ORM problem :** The "elegant" JOIN becomes N+1 fetches when the ORM decides to lazily load accounts

# The Right Mental Model: Design for Access Patterns, Not just for Normalization
The normalized schema is correct for writes. Normalization avoids update anomalies.

It is wrong as the primary read model at scale!

Hence the proposal to shift to [CQRS — Command Query Responsibility Segregation](https://en.wikipedia.org/wiki/Command_Query_Responsibility_Segregation).

Writes still go to the normalized source-of-truth schema. But reads get their own optimized data stores. 

For this system, two read models are required:
1. **Read Model 1: User Summary Store**
Optimized for the listing API. Contains exactly: userId, firstName, lastName, kycStatus, region. Pre-joined. No relationships to traverse.
2. **Read Model 2: User Detail Store**
Optimized for the detail API. Contains the full denormalized user record: KYC fields + all accounts. Point lookup by userId. One read, complete response.

With CQRS, the operational complexity here is real: you now have two representations of the same data. They must stay consistent. That consistency mechanism is part of this series.

# The Indexes That Actually Matter
Indexes are one  of the most common hotspots as far as DB performance is concerned.

The indexes most teams create 
``` sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_kyc_user_id ON kyc_records(user_id);
CREATE INDEX idx_accounts_user_id ON accounts(user_id);
```

Standard Indexes are sub optimal for listing API query at scale because query using standard index requires 2 steps -
  1. **Index Scan -** B-tree traversal to find the row identifier that matches WHERE clause
  2. **Heap Fetch -** For each row identifier found, DB goes to the heap and fetches the full row 

What you actually need
``` sql
-- Covering index for the summary read (index-only scan, no heap fetch)
CREATE INDEX idx_user_summary ON users(region, created_at DESC, user_id)
  INCLUDE (full_name, kyc_status);
```
A covering index stores the additional columns you need directly inside the index structure. When the query planner uses this index, Step 2 i.e. Heap Fetch is not required

# What Comes Next
The data model is sorted. But the listing API still has a major performance / scalability issue that most teams won't find until they're in production. Stay tuned for [Part - 3]()

# Architect's Note
Three-table JOIN, proper indexes, clean normalization - Correct on day one. But by year two we start seeing latency degradations.
Fundamentally speaking - the gap between a schema that works and a schema that scales is always access pattern analysis. That's not taught in SQL courses. It's learnt by making hands dirty while triaging and fixing production incidents.
