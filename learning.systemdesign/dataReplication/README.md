# 🌐 Data Replication

Notes:

<p align="center">
  <img src="./data-replication-1.jpg" alt="Data Replication Illustration 1"></img><br>
  <img src="./data-replication-2.jpg" alt="Data Replication Illustration 2"></img><br>
  <img src="./data-replication-3.jpg" alt="Data Replication Illustration 3"></img>
</p>

Data replication is the process of keeping multiple copies of the same data across different servers. This ensures the system remains reliable, fast, and available even if some components fail.
---

## 🧭 At a Glance

- 🔁 **Idea:** Keep multiple copies of the same data on different machines.
- 🎯 **Goals:** High availability, fault tolerance, faster reads, and disaster recovery.
- 🧱 **Patterns:** Primary–Replica (Master–Slave) and Multi–Master (Master–Master).
- ⚖️ **Sync Modes:** Synchronous, Asynchronous, and Semi–Synchronous replication.

## 📚 Table of Contents

- [🚀 Why do we need Replication?](#why-replication)
- [🛠 Types of Data Replication](#types-of-data-replication)
  - 1️⃣ [Primary-Replica (Master-Slave)](#primary-replica)
  - 2️⃣ [Multi-Master Replication (Master-Master)](#multi-master)
- [⚡ Ways to Propagate Data](#ways-to-propagate-data)
  - ⏱️ [Synchronous Replication](#synchronous-replication)
  - 🏎️ [Asynchronous Replication](#asynchronous-replication)
  - ⚖️ [Semi-Synchronous Replication](#semi-synchronous-replication)
- [📊 Comparison Flow of different kinds of synchronization](#comparison-sync-flow)

---

## 🚀 Why do we need Replication? <a id="why-replication"></a>

* **High Availability:** If one server fails, another replica can serve the request, ensuring zero downtime.
* **Fault Tolerance:** Data is not lost even if a machine crashes.
* **Better Read Performance:** Multiple replicas can be used to serve read requests simultaneously.
* **Disaster Recovery:** If a whole data center goes down, we can failover to a replica in another region.

---

## 🛠 Types of Data Replication <a id="types-of-data-replication"></a>

### 1. Primary-Replica (Master-Slave) 👑 <a id="primary-replica"></a>

In this model, one node is designated as the **Primary** for all write operations, while **Replica** nodes handle the read traffic.

**How it works:**

1. Clients send all **Writes** (and critical reads) to the Primary node.
2. **Read** requests are distributed among Replicas via a **Load Balancer**.
3. The Primary node logs the data changes to a **Write-Ahead Log (WAL)**.
4. Replicas replay these logs in the same order to stay consistent.

```mermaid
graph TD
    Client((Client))

    subgraph Traffic_Distribution [Traffic Layer]
        LB[Load Balancer]
    end

    subgraph Data_Layer [Data Layer]
        Primary[(Primary Node)]
        Replica1[(Replica 1)]
        Replica2[(Replica 2)]
    end

%% Write Flow
    Client -- "Write / Critical Read" --> Primary

%% Read Flow
    Client -- "Standard Read" --> LB
    LB --> Replica1
    LB --> Replica2

%% Replication Flow
    Primary -. "Sync Logs (WAL)" .-> Replica1
    Primary -. "Sync Logs (WAL)" .-> Replica2

```

| **Pros** ✅ | **Cons** ❌ |
| :--- | :--- |
| Excellent for read-heavy systems | Replication Lag (Delay) |
| Simple to implement | Single point of failure (Primary) |
| Easy failover for replicas | Potential data loss during primary failover |

> 💡 **Note**
>
> * 🧩 **On Replica Failure:** Requests are served by other replicas; a new replica is spun up to handle traffic.
> * 🏆 **On Primary Failure:** One replica is promoted to Primary (the one with the latest data).

---

### 2. Multi-Master Replication (Master-Master) 👑👑 <a id="multi-master"></a>

Multiple nodes act as Primary nodes. Any node can accept a write request and then synchronize with the others.

**How it works:**

* A client sends a write to **Node A**.
* Node A writes to its WAL and propagates the change to all other nodes.
* All nodes eventually sync to have the same data.

```mermaid
graph TD
    Client((Client))

    NodeA[(Master Node A)]
    NodeB[(Master Node B)]
    NodeC[(Master Node C)]

    Client <-->|Read/Write| NodeA
    Client <-->|Read/Write| NodeB
    Client <-->|Read/Write| NodeC
    NodeB <-->|Sync| NodeC
    NodeA <-->|Sync| NodeB
    NodeC <-->|Sync| NodeA


```

| **Pros** ✅ | **Cons** ❌ |
| :--- | :--- |
| No single point of failure | Extremely complex conflict resolution |
| Low latency for geo-distributed systems | Hard to maintain strict consistency |

> ⚠️ **Conflict Resolution:** To resolve concurrent updates, systems often use **Last Version Wins (LWW)** based on timestamps or unique DB node priorities.

---

## ⚡ Ways to Propagate Data <a id="ways-to-propagate-data"></a>

How data moves from the Primary to the Replicas defines the balance between speed and safety.

### 1. Synchronous Replication ⏱️ <a id="synchronous-replication"></a>

The write is successful only when **all** replicas confirm they have received and written the data.

* **Pros:** Strong consistency, no data loss.
* **Cons:** High latency, poor performance at scale.

```mermaid
graph TD
    Client((Client))
    Primary[(Primary)]
    Replica1[(Replica 1)]
    Replica2[(Replica 2)]

    Client -- "1. Write Request" --> Primary
    Primary -- "1. Replicate" --> Replica1
    Primary -- "1. Replicate" --> Replica2
    Replica1 -- "1. ACK" --> Primary
    Replica2 -- "1. ACK" --> Primary
    Primary -- "1. Write Success" --> Client
```

### 2. Asynchronous Replication 🏎️ <a id="asynchronous-replication"></a>

The Primary confirms the write to the client immediately after writing it locally, without waiting for replicas.

* **Pros:** Fast writes, scales very well.
* **Cons:** Risk of data loss if Primary fails before syncing; replication lag.

```mermaid
graph TD
    Client((Client))
    Primary[(Primary)]
    Replica1[(Replica 1)]
    Replica2[(Replica 2)]

    Client -- "1. Write Request" --> Primary
    Primary -- "1. Write Success" --> Client
    
    subgraph Background_Sync [Background Process]
        Primary -. "2. Sync Logs" .-> Replica1
        Primary -. "2. Sync Logs" .-> Replica2
    end
```

### 3. Semi-Synchronous Replication ⚖️ <a id="semi-synchronous-replication"></a>

The Primary waits for at least **one** replica to acknowledge the write before confirming to the client.

* **Pros:** Better durability than Async, faster than full Sync.
* **Cons:** Still some latency; partial data loss is possible if both Primary and the synced replica fail simultaneously.

```mermaid
graph TD
    Client((Client))
    Primary[(Primary)]
    Replica1[(Replica 1)]
    Replica2[(Replica 2)]

    Client -- "1. Write Request" ---> Primary
    Primary -- "1. Replicate" --> Replica1
    Replica1 -- "1. ACK" --> Primary
    Primary -- "1. Write Success" --> Client

    subgraph Late_Sync [Delayed Sync]
        Primary -. "2. Replicate" .-> Replica2
    end
```

---

### 📊 Comparison Flow of different kinds of synchronization <a id="comparison-sync-flow"></a>

```mermaid
sequenceDiagram
    participant Client
    participant Primary
    participant Replica1 as Replica (Fast)
    participant Replica2 as Replica (Slow)

    Note over Client, Replica2: Synchronous
    Client->>Primary: Write Request
    par Propagate to All
        Primary->>Replica1: Propagate Data
        Primary->>Replica2: Propagate Data
    end
    par Wait for ACKs
        Replica1-->>Primary: ACK
        Replica2-->>Primary: ACK
    end
    Primary-->>Client: Write Successful (High Latency, High Safety)

    Note over Client, Replica2: Asynchronous
    Client->>Primary: Write Request
    Primary-->>Client: Write Successful (Low Latency, Higher Risk)
    Note right of Primary: background sync
    Primary-)Replica1: Propagate Data (Later)
    Primary-)Replica2: Propagate Data (Later)

    Note over Client, Replica2: Semi-Synchronous (Balanced)
    Client->>Primary: Write Request
    par Propagate
        Primary->>Replica1: Propagate Data
        Primary->>Replica2: Propagate Data
    end
    Replica1-->>Primary: ACK (Fastest Replica)
    Primary-->>Client: Write Successful (Lower Latency than Sync)
    Replica2-->>Primary: ACK (Later)
```