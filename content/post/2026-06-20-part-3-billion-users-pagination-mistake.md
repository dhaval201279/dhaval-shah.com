---
title: Billion User Trap - The Pagination Mistake That Can Take Down Your Database
author: Dhaval Shah
type: post
date: 2026-06-30T02:00:50+00:00
url: /billion-user-pagination-db-design/
categories:
  - architecture
  - distributed-systems
  - database-optimization
tags:
  - architecture
  - distributed-systems
  - database-optimization
thumbnail: "images/wp-content/uploads/2026/06/1-billion-user-kyc-pagination-db-design-part-3.png"
---

[![](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-pagination-db-design-part-3.png)](https://www.dhaval-shah.com/images/wp-content/uploads/2026/06/1-billion-user-kyc-pagination-db-design-part-3.png)
-----------------------------------------------------------------------------------------------------------------------------------------

# Background
In [Part 1](https://www.dhaval-shah.com/billion-user-trap-2-simple-apis/), we established that **access patterns** & not entities should drive your design. In [Part 2](https://www.dhaval-shah.com/billion-user-trap-db-design/), we fixed the database by replacing brittle three-table JOINs with a covering index, reducing P99 on the listing query. The schema was right. The index was right. The listing API itself was fast.
And then it quietly started taking the database down - not because of a bad query, but because of how that fast query was being **paginated**.

# The Query That Kills Production
It's one line of SQL. It is the default behavior in most ORMs.

And at scale, it silently impacts database performance.

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

What happens while OFFSET query runs - It holds **buffer pool pages** and **I/O bandwidth** that your real-time user queries also need.

---

# Cursor-Based Pagination

The correct approach is **keyset pagination** (also called **_cursor-based pagination_**).

> Instead of "give me page N," you ask: "give me 50 records that come after a specific record."

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
4. Query time is **_O(1)_** with respect to page depth - **page 100,000 is exactly as fast as page 1**

The same query at page 100,000 now runs in **4ms** instead of **1,800ms**. Not 10% faster. 450X faster.

---

## Implementing the Cursor Token
Exposing raw timestamps and IDs as cursor values, leaks your schema. Most common mistake I have seen is considering Base 64 encoding as a security mechanism that will safeguard your application from leaking schema.

The correct cursor design addresses this security problem, and ties the payload structure directly to the index that serves it.

### Step-1 : The payload mirrors the index
The cursor payload is not an arbitrary bag of fields - it must contain exactly the columns in the same order as the index that powers the query. Recall the covering index from [Part 2](https://www.dhaval-shah.com/billion-user-trap-db-design/) :

``` sql
  CREATE INDEX idx_user_summary ON users(region, created_at DESC, user_id)
    INCLUDE (full_name, kyc_status);
```
The cursor payload must comprise of the same three fields and that too in the same order

``` java
  public record CursorPayload(
    String region,
    Instant createdAt,
    UUID userId
  ) {}
```
### Step-2 : JSON instead of pipe-separated strings
Pipe-delimited strings are fragile - they break the moment a field value contains the delimiter

### Step-3 : HMAC-signed cursor - encoding with integrity
The cursor must be a signed token : **payload + a signature** computed over that payload using a server-held secret key

``` java
@Component
public class CursorCodec {

    private static final String HMAC_ALGORITHM = "HmacSHA256";
    private final SecretKeySpec signingKey;
    private final ObjectMapper mapper;

    public CursorCodec(@Value("${cursor.signing.secret}") String secret) {
        this.signingKey = new SecretKeySpec(secret.getBytes(StandardCharsets.UTF_8), HMAC_ALGORITHM);
        this.mapper = new ObjectMapper().registerModule(new JavaTimeModule());
    }

    public String encode(CursorPayload payload) {
        try {
            byte[] payloadBytes = mapper.writeValueAsBytes(payload);
            byte[] signature = sign(payloadBytes);

            // Final token = base64(payloadBytes) "." base64(signature)
            String encodedPayload = Base64.getUrlEncoder().withoutPadding().encodeToString(payloadBytes);
            String encodedSignature = Base64.getUrlEncoder().withoutPadding().encodeToString(signature);

            return encodedPayload + "." + encodedSignature;
        } catch (Exception e) {
            throw new CursorEncodingException("Failed to encode cursor", e);
        }
    }

    public CursorPayload decode(String cursorToken) {
        try {
            String[] parts = cursorToken.split("\\.");
            if (parts.length != 2) {
                throw new InvalidCursorException("Malformed cursor token");
            }

            byte[] payloadBytes = Base64.getUrlDecoder().decode(parts[0]);
            byte[] providedSignature = Base64.getUrlDecoder().decode(parts[1]);

            byte[] expectedSignature = sign(payloadBytes);
            if (!MessageDigest.isEqual(expectedSignature, providedSignature)) {
                // Constant-time comparison — prevents timing attacks on signature verification
                throw new InvalidCursorException("Cursor signature mismatch — possible tampering");
            }

            return mapper.readValue(payloadBytes, CursorPayload.class);
        } catch (InvalidCursorException e) {
            throw e;
        } catch (Exception e) {
            throw new InvalidCursorException("Failed to decode cursor", e);
        }
    }

    private byte[] sign(byte[] data) throws Exception {
        Mac mac = Mac.getInstance(HMAC_ALGORITHM);
        mac.init(signingKey);
        return mac.doFinal(data);
    }
}
```
A few details worth calling out :
- The signing key lives in configuration, and not in code
- Token format is **_payload.signature_**

Your API response carries the cursor for the next page :
``` json
  {
    "users": [...],
    "nextCursor": "eyJyZWdpb24iOiJOQSIsImNyZWF0ZWRBdCI6IjIwMjYtMDYtMTVUMDQ6MzA6MDBaIiwidXNlcklkIjoiM2Y0OGEyLi4uIn0.k3J8x9PqRz...",
    "hasMore": true
  }
```

Client sends _?cursor=eyJyZWdpb24i..._ on the next request. 

No page numbers, no offsets. Stateless, Scalable & Secured.

## Advantages of this approach
- Consistent performance at any depth
- No database-killing deep scans

## The Trade-off You Must Acknowledge
Cursor pagination shown here as part of my solution to the problem statement comes with constraints -
1. Jump-to-page navigation ("go to page 37") - **impossible with cursors**
2. Total count - accurate total requires a COUNT(*) which is itself expensive at scale
3. Bidirectional browsing - **reverse cursors are possible but with some additional complexity**

# Architect's Note
OFFSET-based pagination is one of the most common database performance mistake I have seen in platforms that creep in due to original design assumptions. It runs fine for initial years, and then quietly starts eating database capacity as the volume of data grows.
If your platform's user listing or any **"search and scroll"** feature uses OFFSET under the hood - it's worth auditing before scale forces the issue.
