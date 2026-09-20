Sure. Here is the expanded cheat sheet including **MySQL** and **Redis**.

## Database choice cheat sheet

| Technology | Best for | Strengths | Weaknesses |
|---|---|---|---|
| **PostgreSQL** | General-purpose relational apps | SQL, joins, ACID, strong consistency, advanced features | Harder to scale horizontally, not ideal for extreme write scale |
| **MySQL** | Relational apps with simple to moderate complexity | Mature, reliable, widely used, easy to operate, good read performance | Fewer advanced features than PostgreSQL in many cases, still not ideal for extreme distributed scale |
| **MongoDB** | Document-oriented apps | Flexible schema, nested documents, fast iteration | Weak for joins and strong relational modeling |
| **DynamoDB** | Managed key-value/document at scale on AWS | Fully managed, low ops, very fast key-based access | Must know access patterns, limited ad hoc querying |
| **Cassandra** | Massive write-heavy distributed systems | High availability, multi-region, high write throughput, horizontal scale | Query-first modeling, no joins, operational complexity |
| **Redis** | Caching, fast temporary data, sessions, queues, counters | Extremely fast in-memory access, simple data structures | Usually not the system of record, memory cost, data durability depends on setup |

---

## When to use each one

### PostgreSQL
Use it when you need:
- SQL
- joins
- transactions
- strong data integrity
- complex querying and reporting

Examples:
- Payments
- Orders
- Inventory
- SaaS backends
- Admin systems

### MySQL
Use it when you need:
- a reliable relational database
- common web application patterns
- simple transactions and joins
- broad ecosystem support

Examples:
- Traditional web apps
- Content platforms
- E-commerce backends
- OLTP systems

**MySQL vs PostgreSQL:**  
Choose **PostgreSQL** if you want richer SQL features and more advanced querying. Choose **MySQL** if your use case is straightforward relational storage and your team already knows it well.

### MongoDB
Use it when:
- your data is document-shaped
- schema changes frequently
- you want fast iteration
- nested objects map naturally to your domain

Examples:
- User profiles
- Product metadata
- CMS content
- Catalog data

### DynamoDB
Use it when:
- you are on AWS
- you want a fully managed database
- access patterns are known in advance
- you need scale without managing servers

Examples:
- Sessions
- User state
- Shopping carts
- Per-user lookup data

### Cassandra
Use it when:
- write throughput is huge
- high availability matters
- you need multi-region support
- queries are predictable and key-based
- you can denormalize data

Examples:
- Messaging
- Metrics
- IoT telemetry
- Activity feeds
- Logging

### Redis
Use it when:
- you need very low latency
- the data is temporary or derived
- you want caching
- you need counters, rate limiting, leaderboards, or session storage

Examples:
- Cache for database reads
- Rate limiting
- Session store
- Queue-like workflows
- Leaderboards
- Distributed locks, with caution

**Important:** Redis is usually not the main database for durable business data. It is commonly an in-memory support layer.

---

## Quick decision guide

### Need joins, SQL, and transactions?
- **PostgreSQL** or **MySQL**

### Need flexible document storage?
- **MongoDB**

### Need managed scale on AWS with known access patterns?
- **DynamoDB**

### Need massive writes and always-on distributed availability?
- **Cassandra**

### Need ultra-fast cache or temporary state?
- **Redis**

---

## PostgreSQL vs MySQL

### Choose PostgreSQL if:
- you need advanced SQL
- you need more powerful querying
- you want stronger support for complex data types
- your app may grow in complexity

### Choose MySQL if:
- your relational needs are simpler
- you want a very common web stack choice
- your team already runs MySQL comfortably

In many interview answers, you can say:
- **PostgreSQL** for richer functionality
- **MySQL** for straightforward relational workloads

---

## Redis vs everything else

Redis is different from the others because it is usually a **cache or fast state store**, not the primary source of truth.

Use Redis for:
- caching hot data
- reducing DB load
- rate limits
- sessions
- ephemeral counters
- pub/sub or lightweight coordination

Do not use Redis as the only store for:
- financial records
- durable order history
- long-term business data

---

## One-line summaries

- **PostgreSQL**: best all-round relational database
- **MySQL**: reliable relational database for common web apps
- **MongoDB**: best for flexible documents
- **DynamoDB**: best managed AWS key-value/document store
- **Cassandra**: best for huge write-heavy distributed workloads
- **Redis**: best for cache and ultra-fast temporary data

---

## Interview-ready answer

---


“If I need relational integrity, joins, and transactions, I’d choose PostgreSQL or MySQL. If the data is document-shaped and evolving, I’d choose MongoDB. If I’m on AWS and want a fully managed database with predictable access patterns, I’d choose DynamoDB. If the workload is extremely write-heavy and distributed across regions, Cassandra is a strong fit. And if I need very fast caching, sessions, or counters, Redis is the right tool.”

If you want, I can next give you a **single comparison table for all 6**, or a **system-design scenario guide** like “what database for chat app, ecommerce, analytics, social feed, leaderboard, and session management.”


## Cassandra row update

This sentence is describing a core trade-off in Cassandra’s storage engine.

Cassandra is built on an LSM-style write path, which means it does **not** overwrite the old value in place when you update a row. Instead, it appends a new version of the data with a newer timestamp. That makes writes fast, but it creates extra work later for reads and storage cleanup.

## What happens on an update?

Suppose a row starts like this:

```text
user_id = 42, status = "active"
```

Later, you update it:

```text
user_id = 42, status = "inactive"
```

In a traditional in-place storage engine, the database may go to the old record and replace `"active"` with `"inactive"`.

In Cassandra, the update is treated more like:

- keep the old version
- append a new version
- mark the new one as the latest based on timestamp

So now Cassandra may have multiple versions of the same row spread across different SSTables.

## Why are SSTables involved?

Cassandra writes data in a sequence like this:

1. write goes to the commit log for durability
2. write goes to the memtable in memory
3. memtable flushes to disk as an SSTable
4. future updates create more SSTables over time

Because SSTables are immutable, Cassandra does not edit them in place. That is why a frequently updated row can end up scattered across many SSTables.

## What does “newer timestamp” mean?

Cassandra uses timestamps to decide which value is the latest version of a cell or row.

If one SSTable has:

- `status = "active"` at time `100`

and another has:

- `status = "inactive"` at time `200`

then Cassandra knows `"inactive"` wins, because it has the newer timestamp.

So the database may store both values temporarily, but only one is considered current.

## Why does this hurt reads?

When you read a row, Cassandra may have to check multiple SSTables to reconstruct the latest version.

That means it may need to:

- look in SSTable A
- look in SSTable B
- look in SSTable C
- compare timestamps
- ignore older overwritten values
- return only the newest visible data

This extra work is called **read amplification**.

### Read amplification means:
A single logical read turns into many physical disk checks.

The more updates a row has, the more SSTables may contain old versions of that row, and the slower reads can become.

## Why does it waste space?

Until compaction runs, old overwritten versions still exist on disk.

So if a row is updated many times:

- old versions are still stored
- newer versions are also stored
- disk usage grows
- space is not reclaimed immediately

This is what the text means by “wastes space until compaction consolidates them.”

## What is compaction doing?

Compaction is the cleanup process.

It merges SSTables together and:

- keeps the newest version of each key
- drops overwritten values
- removes data that is no longer needed
- writes cleaner SSTables back to disk

So compaction reduces:

- read amplification
- disk waste
- the number of SSTables a read must scan

## A simple example

Imagine one user profile gets updated 5 times:

- `name = "A"` in SSTable 1
- `name = "B"` in SSTable 2
- `name = "C"` in SSTable 3
- `name = "D"` in SSTable 4
- `name = "E"` in SSTable 5

Before compaction, a read may need to inspect several SSTables to find the latest value.

After compaction, Cassandra can merge those versions and keep only:

- `name = "E"`

## Why this matters in practice

This is why Cassandra is great for:

- append-heavy workloads
- time-series data
- logs
- events
- messages

and weaker for workloads that:

- update the same rows repeatedly
- overwrite a lot of fields frequently
- need low-latency reads on heavily updated records

## In one line

Cassandra is optimized for fast writes by appending new versions instead of updating in place, but repeated updates make reads more expensive until compaction cleans up the old versions.

If you want, I can also explain this with a **visual timeline**, or compare it with how **PostgreSQL/MySQL** handle updates.


Sure.

## Memtable

A **memtable** is Cassandra’s in-memory write buffer.

When a write comes in, Cassandra does not immediately put it into a big on-disk table. Instead, it first:

1. appends the write to the **commit log** for durability
2. stores the write in the **memtable** in memory

The memtable is basically the current working set of recent writes.

### Why it exists
- It makes writes fast
- It avoids constant random disk writes
- It lets Cassandra batch many writes before flushing to disk

### What happens next
When the memtable gets full, Cassandra flushes it to disk and turns it into an SSTable.

---

## SSTable

**SSTable** stands for **Sorted String Table**.

It is an immutable on-disk file that stores data in sorted order.

### Key idea
Once Cassandra writes data into an SSTable, it does **not** modify that file.

If new updates arrive, they go into a new memtable, and later into a new SSTable.

### Why it exists
- Sorted data makes lookups efficient
- Immutable files are simpler and faster to manage
- Writes can be append-like instead of in-place updates

### What SSTables contain
An SSTable usually includes:
- the actual data
- an index
- metadata
- sometimes a summary or bloom filter to speed up lookups

---

## How memtable and SSTable work together

A write flows like this:

```text
Client write
   -> Commit log
   -> Memtable
   -> Flush
   -> SSTable on disk
```

A read may check:
- memtable first
- then one or more SSTables

Because of this, Cassandra can serve recent writes even before they are flushed to disk.

---

## What is “representation”?

I think you may mean **representation of data** in Cassandra, but the word is a bit ambiguous. In this context, it usually means how Cassandra stores a row internally across memtables and SSTables.

### Logical representation
From your application’s point of view, you see:
- rows
- columns
- primary key
- partition key
- clustering columns

### Physical representation
Internally, Cassandra may represent the same logical row as:
- one version in memtable
- another version in one SSTable
- another older version in another SSTable

So the “representation” of the data is split across memory and disk, not stored as one single mutable row like in some relational databases.

---

## Simple example

Suppose you update a user’s status multiple times:

- `active`
- `inactive`
- `active again`

Cassandra may store these as multiple versions across different SSTables. The newest timestamp wins.

So the physical representation is versioned, while the logical result looks like one current row.

---

## Why this design matters

This design gives Cassandra:
- very fast writes
- good horizontal scale
- predictable append-heavy performance

But it also causes:
- read amplification
- compaction work
- tombstones for deletes
- more operational complexity

If you want, I can next explain:
1. **commit log vs memtable vs SSTable**, or  
2. **how a read works in Cassandra step by step**.


##

This code defines a **Cassandra table for time-series sensor data**, and it uses **time bucketing** to keep partitions small and efficient.

```sql
CREATE TABLE sensor_readings (
    sensor_id TEXT,
    day DATE,
    reading_time TIMESTAMP,
    value DOUBLE,
    PRIMARY KEY ((sensor_id, day), reading_time)
);
```

## What each column means

- **sensor_id TEXT**  
  Identifies which sensor produced the reading.

- **day DATE**  
  A bucket column that groups readings by day.

- **reading_time TIMESTAMP**  
  The exact time the reading was taken.

- **value DOUBLE**  
  The actual sensor value, like temperature or humidity.

---

## What the primary key means

The key part is:

```sql
PRIMARY KEY ((sensor_id, day), reading_time)
```

This has two parts:

### 1. Partition key: `(sensor_id, day)`
This means all readings for:
- one specific sensor
- on one specific day

go into the same partition.

So if `sensor_id = S1` and `day = 2024-01-15`, all readings for that sensor on that day are stored together.

### 2. Clustering column: `reading_time`
Inside that partition, rows are ordered by `reading_time`.

So readings for the same sensor and day are stored in time order, which makes range queries efficient.

---

## Why this design is good

This is a classic Cassandra pattern for time-series data.

### It prevents unbounded partitions
If you used only `sensor_id` as the partition key, then all history for one sensor would go into one giant partition. Over time that partition could become too large and slow.

By adding `day`, Cassandra creates a separate partition for each sensor per day, which keeps partitions small and manageable.

### It supports time-based queries
This design is ideal for queries like:

```sql
SELECT * FROM sensor_readings
WHERE sensor_id = 'S1' AND day = '2024-01-15';
```

or:

```sql
SELECT * FROM sensor_readings
WHERE sensor_id = 'S1'
  AND day = '2024-01-15'
  AND reading_time >= '2024-01-15 10:00:00'
  AND reading_time <= '2024-01-15 11:00:00';
```

These queries are efficient because they target one partition and then scan rows in sorted order.

---

## What this table is optimized for

This schema is good when:
- readings arrive continuously
- you usually query data by sensor and time range
- older data can be bucketed by day
- you want predictable, fast reads and writes

Typical use cases:
- IoT sensor data
- metrics
- telemetry
- logs
- event streams

---

## Why `day` is important

The `day` column is not just extra data, it is a **partitioning strategy**.

It helps solve a common Cassandra problem, which is **partition growth**.

Without it:
- one sensor could accumulate millions of rows in one partition

With it:
- each day becomes its own bounded partition
- compaction and reads stay more efficient

---

## How the data is organized

For example:

- `S1, 2024-01-15, 10:00`
- `S1, 2024-01-15, 10:05`
- `S1, 2024-01-15, 10:10`

These all go into one partition.

Then:
- `S1, 2024-01-16, 09:00`

goes into a different partition.

So Cassandra stores the same sensor’s data in daily chunks.

---

## One important interview point

This schema works because it matches the query pattern.

Cassandra is query-first, so you design the table around how you will read the data:
- partition by `sensor_id + day`
- order within the partition by `reading_time`

That gives you efficient writes and time-range reads.

---

## In one sentence

This table stores readings per sensor per day in a single partition, with readings sorted by time, so Cassandra can handle time-series data efficiently without creating huge partitions.

If you want, I can also explain:
1. **why double parentheses are used in the primary key**, or  
2. **how to query this table correctly in Cassandra**.


##

This snippet shows the four common `Cache-Control` policies and how they affect who can cache a response, for how long, and whether the cached copy can be reused without asking the server again.

`Cache-Control` is an HTTP response header that tells browsers, CDNs, and proxies what they are allowed to do with the response. In interviews, this is usually discussed as part of HTTP caching and conditional requests, often alongside `ETag` and `304 Not Modified`.

### 1. `Cache-Control: public, max-age=3600`

This means the response can be cached by **any cache**, including shared caches like CDNs or reverse proxies.

- `public` means it is okay for shared caches to store it.
- `max-age=3600` means the response is considered fresh for **3600 seconds**, which is 1 hour.

So if this is a product catalog page or some other non-personalized data, a CDN can serve it directly for an hour without contacting your origin server again.

### 2. `Cache-Control: private, max-age=60`

This means the response is cacheable, but **only in a private cache**, usually the user’s browser.

- `private` means shared caches should not store it.
- `max-age=60` means it stays fresh for 60 seconds.

This is useful for per-user content, like a dashboard or account summary, where caching helps the same user but the data should not be reused by other users.

### 3. `Cache-Control: no-cache`

This does **not** mean “do not cache at all”.

It means the response may be stored, but it must be **revalidated with the server before reuse**. In other words, a cache cannot just serve the stored copy blindly.

This is helpful when you want caching efficiency but also want to make sure the client always checks whether the data changed. Typically this works with `ETag` or `Last-Modified`.

### 4. `Cache-Control: no-store`

This means **do not store the response anywhere**.

- No browser cache
- No shared cache
- No disk storage

Use this for highly sensitive data, like one-time secrets, banking session data, or anything you never want persisted locally.

### Quick mental model

- `public` = cache anywhere
- `private` = cache only for one user/client
- `no-cache` = can cache, but must revalidate before using
- `no-store` = do not cache at all

### Why this matters

These directives help balance:
- **performance** by reducing repeated server calls
- **correctness** by preventing stale or wrong data
- **security/privacy** by avoiding unwanted storage of sensitive responses

If you want, I can also explain how `Cache-Control` works together with `ETag` and `If-None-Match` using a simple request flow.

##

`If-None-Match` is a **conditional request header**. It means:

> “Only send me the resource if its current ETag does **not** match what I already have.”

So the server compares the value in `If-None-Match` with the resource’s current `ETag`.

---

## How the server interprets it

There are two main outcomes:

### 1. ETag matches
If the server’s current ETag is the same as the one in `If-None-Match`, then the client already has the latest version.

The server responds with:

```http
304 Not Modified
```

No body is sent. The client or cache reuses its stored copy.

### 2. ETag does not match
If the resource changed, the ETag is different.

The server responds with:

```http
200 OK
```

and sends the new representation plus the new `ETag`.

---

## Example

### First response

```http
GET /users/123
```

Server returns:

```http
200 OK
ETag: "user-v1"

{
  "id": 123,
  "name": "Alice"
}
```

Now the client stores both:
- the body
- the ETag `"user-v1"`

---

### Later request with `If-None-Match`

```http
GET /users/123
If-None-Match: "user-v1"
```

#### Case A: nothing changed
Server still has version `"user-v1"`.

Response:

```http
304 Not Modified
ETag: "user-v1"
```

Meaning:
- your cached copy is still valid
- do not download the body again

#### Case B: resource changed
Suppose the user name changed, and the server now has:

```http
ETag: "user-v2"
```

Response:

```http
200 OK
ETag: "user-v2"

{
  "id": 123,
  "name": "Alice Cooper"
}
```

Meaning:
- your old copy is stale
- here is the new data

---

## Important rule

`If-None-Match` is mostly used with `GET` and `HEAD`.

It is asking:

> “Has this representation changed since the version I know about?”

For writes, the related header is usually `If-Match`, which is used for optimistic concurrency control.

---

## One special case: `If-None-Match: *`

This means:

> “Only proceed if the resource does not already exist.”

That is useful for create-if-absent behavior.

Example:

```http
PUT /files/report.pdf
If-None-Match: *
```

This says, “Create it only if there is no current file at this path.”

---

## Simple mental model

- `ETag` = version label from the server
- `If-None-Match` = version label the client already has
- server compares them
- if they match, return `304`
- if they differ, return `200` with the new content

If you want, I can also show the difference between `If-None-Match` and `If-Match` in one table.

##

Not exactly. The main server does **not usually keep a separate manually maintained ETag registry for every piece of data**.

Instead, the server computes or assigns an ETag for the **final response representation** it sends back.

## What an ETag really represents

An ETag is a validator for a **specific response body**, not necessarily for a database row.

So if the response is:

```json
{
  "id": 123,
  "name": "Alice"
}
```

the server might generate an ETag based on:

- a version number
- a last-updated timestamp
- a hash of the final response body
- a hash of the underlying data used to build that response

---

## How it is maintained

There are a few common strategies.

### 1. Version field in the database
If a resource is stored in one row or document, the simplest approach is to keep a version column:

```text
user_id | name  | email | version
123     | Alice | a@x.com | 7
```

When the row changes, increment `version`.

Then the server can create:

```http
ETag: "user-123-v7"
```

This is common for single-resource endpoints.

---

### 2. Hash of the response body
The server can serialize the response and hash it.

For example, if the response body changes, the hash changes, so the ETag changes.

This works well when:
- the response is generated from multiple fields
- the response is mostly stable
- exact byte-level identity matters

But it can be expensive if the response is large.

---

### 3. Hash/version from underlying sources
For complex responses built from joins or internal queries, the server often does **not** hash the final JSON every time if it is costly.

Instead, it may derive the ETag from the version of the underlying data sources.

Example: a `/dashboard` response may depend on:
- user profile version
- unread notification count version
- recent orders version

The server might compute a combined validator like:

```text
ETag = hash(user_version, notification_version, order_version)
```

If any input changes, the combined ETag changes.

---

## What about joins and multiple internal queries?

This is where ETags become more of an **application design choice**.

Suppose the endpoint is:

```http
GET /profile
```

and the response is assembled from:
- `users` table
- `orders` table
- `addresses` table

There are two main ways to handle it:

### Option A: Compute ETag from the final response
The server runs the joins, builds the full response, then computes a hash of the output.

Pros:
- simple to reason about
- exact match for the response

Cons:
- expensive for large or frequently requested responses
- you still have to execute the joins before you know the ETag

### Option B: Compute ETag from source versions
Each source has a version or timestamp, and the server combines them.

Example:

```text
user_version = 7
order_version = 19
address_version = 3
combined_etag = hash(7, 19, 3)
```

Pros:
- faster than hashing the whole response
- can avoid expensive body generation if nothing changed

Cons:
- harder to implement
- must be careful to include every relevant source, or caches become incorrect

---

## Important caveat with joins

If a response is built from many tables, the ETag is only correct if it reflects **all data that can affect the response**.

Example:

```http
GET /orders/123/summary
```

If the summary includes:
- order info
- shipping status
- payment status

then the ETag must change when any of those change.

If you forget to include payment status in the validator, the server might incorrectly return `304 Not Modified` even though the response should have changed.

---

## In practice

For simple systems:
- use a row `version`
- or `updated_at`
- or a hash of the final response

For complex systems:
- use a derived validator from the underlying entities
- often backed by cache layers or precomputed metadata

For very expensive aggregate endpoints:
- sometimes people avoid ETags entirely and use a TTL cache instead
- or they cache the assembled response and attach a version key to it

---

## Rule of thumb

- **Single resource**: ETag from row/document version is easy
- **Aggregated response**: ETag from combined source versions or response hash
- **Expensive joins**: prefer derived versions or caching, not full recomputation every time

---

## Short answer

No, the server does not usually keep a global ETag per “data object” in a universal sense. It maintains or computes an ETag for the **response representation**, and for complex responses it may derive that ETag from the versions of all underlying data sources.

If you want, I can show you a concrete example of how to generate an ETag for a response built from **3 joined tables**.

##

This is a **safe update** pattern using **optimistic concurrency control**.

## What it means

`If-Match: "v3-9a1f"` says:

> “Only perform this `PUT` if the current version of product `42` is still exactly `v3-9a1f`.”

The client usually got that ETag from a previous `GET` response.

---

## How the server interprets it

When the server receives:

```http
PUT /products/42
If-Match: "v3-9a1f"
```

it compares `"v3-9a1f"` with the **current ETag** of `/products/42`.

### If they match
The resource has not changed since the client last saw it.

So the server applies the update and returns something like:

```http
200 OK
ETag: "v4-2b7c"
```

or sometimes:

```http
204 No Content
```

depending on whether the API returns a body.

### If they do not match
Someone else updated the product after the client read it.

Then the server rejects the write with:

```http
412 Precondition Failed
```

That means the precondition in `If-Match` was not satisfied.

---

## Example

### Step 1: Client reads the product

```http
GET /products/42
```

Response:

```http
200 OK
ETag: "v3-9a1f"

{
  "id": 42,
  "name": "Keyboard",
  "price": 99
}
```

The client stores:
- the product data
- the ETag `"v3-9a1f"`

---

### Step 2: Another user updates the product

Now the server changes the product, maybe price becomes 109.  
Its ETag changes to something like `"v4-2b7c"`.

---

### Step 3: Original client tries to update using the old ETag

```http
PUT /products/42
If-Match: "v3-9a1f"

{
  "id": 42,
  "name": "Keyboard Pro",
  "price": 119
}
```

#### If current ETag is still `"v3-9a1f"`
Update succeeds.

#### If current ETag is now `"v4-2b7c"`
Update is rejected:

```http
412 Precondition Failed
```

---

## Why this is useful

Without `If-Match`, the client could accidentally overwrite someone else’s changes.

This is the classic **lost update problem**:

- Client A reads version 1
- Client B reads version 1
- Client B updates it to version 2
- Client A updates using stale data and overwrites B’s changes

`If-Match` prevents that by ensuring the update only applies to the version the client actually read.

---

## Mental model

- `GET` gives you the current data and ETag
- `PUT` or `PATCH` sends back that ETag in `If-Match`
- server only updates if the ETag still matches
- otherwise returns `412 Precondition Failed`

---

## Short summary

`If-Match` means:

> “Apply this write only if the resource is still in the version I know.”

`412 Precondition Failed` means:

> “The resource changed, so your update is based on stale data.”

If you want, I can also show the difference between **`If-Match` vs `If-None-Match`** in a small table.
