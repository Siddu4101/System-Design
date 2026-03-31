# ⚡ Caching

Notes:
<p align="center">
  <img src="./cache-1.jpg" alt="Cache diagram 1" style="width:420px;height:auto;"/><br/>
  <img src="./cache-2.jpg" alt="Cache diagram 2" style="width:420px;height:auto;"/><br/>
  <img src="./cache-3.jpg" alt="Cache diagram 3" style="width:420px;height:auto;"/><br/>
  <img src="./cache-4.jpg" alt="Cache diagram 4" style="width:420px;height:auto;"/>
</p>
---

## 📚 Table of Contents
1. [What Is a Cache?](#1-what-is-a-cache)
2. [Where to Cache? (Strategies)](#2-where-to-cache-strategies)
   - [External Caching](#21-external-caching)
   - [In-Process Caching](#22-in-process-caching)
   - [Content Delivery Networks (CDN)](#23-content-delivery-networks-cdn)
   - [Client-Side Caching](#24-client-side-caching)
3. [Cache Access Patterns / Architectures](#3-cache-access-patterns--architectures)
   - [Cache Aside](#31-cache-aside)
   - [Write-Through](#32-write-through)
   - [Write-Back (Write-Behind)](#33-write-back-write-behind)
   - [Read-Through](#34-read-through)
4. [Cache Eviction Policies](#4-cache-eviction-policies)
5. [Common Caching Issues & Fixes](#5-common-caching-issues--fixes)
   - [Cache Stampede (Thundering Herd)](#51-cache-stampede-thundering-herd)
   - [Cache Consistency](#52-cache-consistency)
   - [Hot Keys](#53-hot-keys)
6. [When to Use Caching](#6-when-to-use-caching)

---

## 1. What Is a Cache? 💡

A **cache** is **temporary storage for recently accessed data** that lets us serve **future requests faster**, improving **latency** and **overall performance**.

> Think of it like keeping your most-used tools on your desk instead of in a box in the garage.

---

## 2. Where to Cache? (Strategies) 🗺️

### 2.1 External Caching

Caching is placed **outside** the application (e.g., Redis, Memcached).

**Flow:**
1. App checks cache.
2. If **miss**, fetch from DB.
3. Store result in cache.
4. Return response.

```mermaid
sequenceDiagram
    autonumber
    participant App as Application
    participant Cache as Redis (External)
    participant DB as Database

    App ->>+ Cache: Get Key
    Cache -->>- App: Cache Miss

    App ->>+ DB: Fetch Data
    DB -->>- App: Data Returned

    Note right of App: App now has data in memory

    App ->>+ Cache: Update Cache (Set Key)
    Cache -->>- App: Acknowledged
    App -->> App: Send Response to User
```

**Pros ✅**
- Shared across app instances.
- Can scale cache independently.

**Cons ❌**
- Extra network hop (slower than in-memory).

---

### 2.2 In-Process Caching (Local / In-Memory)

Each application instance maintains its **own cache in memory** (e.g., Guava cache, Caffeine, in-memory maps).

```mermaid
sequenceDiagram
    autonumber
    participant LB as Load Balancer
    participant App1 as App Instance 1 (with Cache)
    participant App2 as App Instance 2 (with Cache)
    participant DB as Database

    Note over LB, App2: Scenario: Same request sent twice
    
    LB ->>+ App1: Request A
    Note right of App1: Check Internal Memory
    App1 ->>+ DB: Cache Miss -> Fetch from DB
    DB -->>- App1: Data
    App1 ->> App1: Store in Local RAM
    App1 -->>- LB: Response A
    
    LB ->>+ App2: Request A (again)
    Note right of App2: Cache is empty here!
    App2 ->>+ DB: Redundant Fetch from DB
    DB -->>- App2: Data
    App2 ->> App2: Store in Local RAM (Duplicate)
    App2 -->>- LB: Response A
    
    Note over App1, App2: Issue: Data is cached twice in different places
```

**Pros ✅**
- Ultra low latency (no network call).
- Very simple to implement.

**Cons ❌**
- Each instance has its **own copy** (duplicate data).
- Harder to keep data consistent across instances.

---

### 2.3 Content Delivery Networks (CDN)

A **CDN** stores **static content** (HTML, CSS, JS, images, videos, files) **near the user**, reducing **network latency**.

```mermaid
sequenceDiagram
    participant Client as Client (User)
    participant CDN as CDN Server (Regional)
    participant Origin as Origin Server

    Client ->>+ CDN: 1. Request for Static Content

    alt 1a. Cache Hit
        CDN -->> Client: Return Content Directly (Faster)
    else 1b. Cache Miss
        Note over CDN, Origin: Data not found in regional CDN
        CDN ->>+ Origin: 2. Fetch Content from Origin
        Origin -->>- CDN: Return Content
        CDN ->> CDN: 3. Store Content locally
        CDN -->> Client: Return Content (Slower first time)
    end

    deactivate CDN
```

**Use cases 🌍**
- Public static assets.
- Global user base.

---

### 2.4 Client-Side Caching (Browser / App) 🧑‍💻

Content is cached **on the client device**, typically in the **browser cache** (disk or memory): images, CSS, HTML, files, etc.

```mermaid
graph LR
    subgraph ClientSide["Client-Side (User Device)"]
        direction TB
        UserApp[User Browser/App]

        subgraph BrowserCache["Internal Caches"]
            DiskCache[(Disk Cache)]
        end

        UserApp -.-> DiskCache
    end

    ApplicationServer[External Application Server]

    UserApp -->|1. First Request| ApplicationServer
    ApplicationServer -->|2. Content + Headers| UserApp

    UserApp -.->|3. Save Locally| DiskCache
    UserApp -->|4. Check Local| DiskCache
    DiskCache -.->|5. Immediate Hit| UserApp

    style ClientSide fill:#f9f9f9,stroke:#333,stroke-width:2px
    style BrowserCache fill:#fff,stroke:#666,stroke-dasharray: 5 5
```

**Controlled by:**
- HTTP cache headers (`Cache-Control`, `ETag`, `Last-Modified`, etc.).

---

## 3. Cache Access Patterns / Architectures 🏗️

How the application interacts with cache + database.

### 3.1 Cache Aside (Lazy Loading) 🛋️

The application **controls the flow**:

1. Check cache.
2. If **hit** → return data.
3. If **miss** → load from DB, put into cache, return.

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as Cache
    participant DB as Database
    
    Note over App, Cache: Application controls data flow
    
    App->>+Cache: 1. Check in Cache
    alt Cache Hit (found)
        Cache-->>-App: Return Data (Immediate)
    else Cache Miss (not found)
        Cache-->>App: Data Not Found
        App->>+DB: 2. Get From DB
        DB-->>-App: Return DB Data
        App->>+Cache: 3. Put Data back in Cache
        Cache-->>-App: OK
        App->>App: 4. Return Data to user
    end
```

**Pros ✅**
- Cache only stores **actually used** data.
- Easy to reason about.

**Cons ❌**
- First request after a miss is **slow**.

---

### 3.2 Write-Through ✍️➡️💾

On writes/updates:

1. App **writes to cache**.
2. Cache **synchronously writes to DB**.

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as Cache
    participant DB as Database
    
    Note over App, DB: Synchronous Write Sequence
    
    App->>+Cache: 1. Write Data
    Cache ->>+ DB: 2. Sync Write (Sync Push) to DB
    DB -->>- Cache: Write Complete
    Cache -->>- App: Write Acknowledged
    
    Note right of App: Subsequent read requests immediately find data in cache.
```

**Pros ✅**
- Cache and DB are **always in sync** (for successful writes).
- Reads are **always cacheable** immediately.

**Cons ❌**
- Slower writes (must hit both cache and DB).
- Still risk inconsistency if DB write fails but cache succeeds (must handle transactions / error handling carefully).

---

### 3.3 Write-Back (Write-Behind) ✍️➡️🕒

Similar to Write-Through, but **DB writes are async and batched**.

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as Cache
    participant DB as Database
    
    Note over App, DB: Asynchronous Batch Write
    
    App->>+Cache: 1. Write Data
    Cache -->>- App: Write Acknowledged (Fast)
    
    par Async Batch Write
        Cache -->> Cache: Accumulate Writes
        Cache ->>+ DB: 2. Async batch write to DB
        DB -->>- Cache: Complete
    end
```

**Pros ✅**
- Very **fast** writes (app only waits for cache).
- Good for heavy write workloads.

**Cons ❌**
- Risk of **data loss** if cache fails before flushing to DB.
- More complex implementation.

---

### 3.4 Read-Through 👀➡️📚

The application **talks only to the cache**. On miss, the **cache itself** loads data from the DB, stores it, and returns it.

```mermaid
sequenceDiagram
    participant App as Application
    participant Cache as Cache
    participant DB as Database

    App ->>+ Cache: 1. Request Data
    Note over Cache, DB: Cache acts as a data access layer

    alt Cache Hit
        Cache -->> App: Return Data (Direct)
    else Cache Miss
        Cache ->>+ DB: 2. Cache itself fetches from DB
        DB -->>- Cache: Return DB data
        Cache ->> Cache: 3. Store in Cache
        Cache -->> App: 4. Send back response to Application
    end

    deactivate Cache
```

> **Common in CDNs** and some managed caching solutions.

**Pros ✅**
- Centralized logic for data loading.
- App code is simpler (only one endpoint: the cache).

**Cons ❌**
- Cache system becomes **more complex**.

---

## 4. Cache Eviction Policies 🧹

Strategies to **remove entries** from cache when space is needed or to avoid stale data.

1. **LRU (Least Recently Used)**
   - Evict the item that **has not been used for the longest time**.
   - Prioritizes **recently used** data.

2. **LFU (Least Frequently Used)**
   - Evict the item **used the fewest times**.
   - Good when you want to keep **popular (hot)** data.

3. **FIFO (First-In-First-Out)**
   - Evict the **earliest inserted** item first.
   - Simple, but ignores use frequency and recency.

4. **TTL (Time-to-Live)**
   - Each entry has an **expiry time**.
   - After TTL, item is invalidated/removed.
   - Helps avoid **stale data** and control freshness.

Often, systems **combine** these (e.g., LRU + TTL).

---

## 5. Common Caching Issues & Fixes 🚨

### 5.1 Cache Stampede (Thundering Herd)

Many requests hit the system at the **same time** for the **same data** when:
- The cache entry has **expired**, or
- There is a **cache miss** for a popular key.

This can overwhelm the DB or backend.

**Mitigations 🛡️**
1. **Request Coalescing / Single Flight**
   - Let **one** request recompute and fill the cache.
   - Other requests **wait** for that result instead of all hitting the DB.

2. **Pre-computation / Refresh Ahead**
   - Refresh popular keys **before** they expire.
   - Avoids synchronized expiry for hot keys.

3. **Rate Limiting / Backpressure**
   - Limit how many concurrent requests can hit the backend.

---

### 5.2 Cache Consistency

Cache may return **stale data** while DB has **newer data**.

**Mitigations 🛡️**
1. **Evict-on-Update (Invalidate Cache on Write)**
   - When data changes in DB, **evict the corresponding cache key**.
   - Next read will fetch fresh data from DB and repopulate cache.
   - Good when stale data is **not acceptable** (e.g., profile images, balances).

2. **Strict TTL / Short-Lived Entries**
   - Use **short TTL** where some staleness is acceptable.
   - Ensures data is periodically refreshed.

3. **Read-Your-Own-Writes Techniques** (pattern-level)
   - For critical flows, read from DB immediately after write, or bypass cache temporarily.

---

### 5.3 Hot Keys 🔥

A **single key** is accessed **very frequently**, overloading:
- A single cache node, or
- A shared cache cluster.

**Mitigations 🛡️**
1. **Replication + Load Balancing**
   - Replicate that key across **multiple cache nodes**.
   - Use a **load balancer** or client-side sharding to spread traffic.

2. **CDN for Public Data**
   - For public-facing content, use a **CDN** to spread load globally.

3. **Application-Level (In-Process) Cache**
   - Let each application instance keep its own copy.
   - Distributes read load across app instances.

---

## 6. When to Use Caching ✅

Use caching when you have:

- **Read-heavy workload** (many more reads than writes).
- **Strict response time (latency) requirements**.
- **Expensive queries/computations** (joins, aggregations, external API calls).
- **High database CPU utilization** and you want to offload reads.

> Rule of thumb: Cache **data that is read often, changes infrequently, and is expensive to recompute or fetch.**
