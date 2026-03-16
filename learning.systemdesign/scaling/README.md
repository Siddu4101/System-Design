# 📈 Scaling in System Design

<div style="text-align: center;">
  <img src="./HorizontalAndVerticalScaling.jpg"
       alt="Horizontal vs Vertical Scaling Diagram"
       style="width:420px;height:auto;" />
</div>

Designing scalable systems is all about handling **more load** without sacrificing **performance, reliability, or cost-efficiency**. Two foundational strategies are:

- **Vertical Scaling (Scale Up)** 🏗️  
- **Horizontal Scaling (Scale Out)** 🌐

This guide summarizes their differences, trade-offs, and common usage patterns.

---

## 🏗️ Vertical Scaling (Scale Up)

> "Make a single machine more powerful."

You improve capacity by **upgrading the existing server**:

- Add more **CPU cores** 🧠
- Add more **RAM** 🧮
- Use faster **disk / SSD / NVMe** 💾
- Use a bigger, more powerful **instance type** (e.g., AWS `t3.medium → r7g.4xlarge`)

```text
          ┌──────────────────────────┐
          │        Clients           │
          └────────────┬─────────────┘
                       │  HTTP / gRPC
                       ▼ 
              ┌───────────────────┐
              │   Application     │OLD: 8GB RAM 512GB Storage
              │ (More CPU / RAM)  │NEW: 64GB RAM 1TB Storage
              └───────────────────┘
```

### When Vertical Scaling Makes Sense
- Early prototype / MVP.
- Low traffic, simple monolith.
- Systems where **stateful scaling** is hard and availability requirements are moderate.

---

## 🌐 Horizontal Scaling (Scale Out)

> "Add more machines and distribute the load."

You improve capacity by running **multiple instances** of your service and putting a **load balancer** in front.

```text
          ┌──────────────────────────┐
          │        Clients           │
          └────────────┬─────────────┘
                       │  HTTP / gRPC
                       ▼
              ┌───────────────────┐
              │   Application     |
              | (load balancer)   │
              └───────┬───────────┘
          ┌───────────┼───────────┐
          ▼           ▼           ▼
    ┌─────────┐  ┌─────────┐  ┌─────────┐
    │ App #1  │  │ App #2  │  │ App #3  │
    └─────────┘  └─────────┘  └─────────┘
    Each machine has 21GB RAM and 330 GB of storage (almost same as above but in different systems)
```

### When Horizontal Scaling Makes Sense
- When you need to handle **large or unpredictable traffic**.
- When **high availability / fault tolerance** is a requirement.
- Cloud-native, microservices, or internet-scale applications.

---

## ⚖️ Vertical vs Horizontal Scaling – At a Glance

| Aspect             | Vertical Scaling 🏗️                   | Horizontal Scaling 🌐                                    |
|--------------------|----------------------------------------|----------------------------------------------------------|
| How it scales      | Bigger machine                         | More machines                                            |
| Complexity         | Low                                    | Medium / High                                            |
| Scaling limitation | At some point H/W limit occurs         | Usually higher (limited by architecture)                 |
| Fault tolerance    | Low (single point of failure)          | High (resilient if one machine down other can handle it) |
| Cost behavior      | Expensive at high end                  | Scales with number of nodes                              |
| Typical use cases  | Small apps, DB servers, legacy systems | Web apps, APIs, large-scale services                     |
| Upgrade impact     | Often needs downtime                   | Can be zero-downtime (rolling updates)                   |

---

## 🧠 Key Design Considerations for Horizontal Scaling

Horizontal scaling **changes how you design systems**. A few big themes:

### 1. Stateless vs Stateful
- **Stateless services**
  - Each request is independent.
  - Any instance can handle any request.
  - Great for scaling web servers, APIs.
- **Stateful services**
  - Store session or user-specific state in memory.
  - Harder to scale out → use **external state stores**.

👉 Best practice: keep application servers **stateless** and move state to:
- Databases (SQL/NoSQL) 🗄️
- Caches (Redis, Memcached) ⚡
- Object storage (S3, GCS, etc.) 📦

### 2. Load Balancing

Common strategies:
- **Round-robin** – rotate through instances.
- **Least connections** – send to the least busy instance.
- **IP hash / session affinity** – keep a user “sticky” to a specific instance.

Also consider:
- **Health checks** – auto-remove unhealthy instances.
- **Circuit breakers / timeouts** – prevent cascading failures.
