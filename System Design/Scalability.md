# Scalability

Scalability is the ability of a system to handle increased load by adding resources without requiring a redesign or suffering unpredictable performance degradation.

---

## 1. Core Metrics & Measurement

Evaluating scalability relies on tracking concrete load and performance metrics:

* **Load Metrics:**
  * **Requests Per Second (RPS):** API calls handled per unit time.
  * **Concurrent Users:** Total active sessions at any given moment.
  * **Throughput / Query Rate:** Data transfer rate (e.g., GB/s) and database queries per second (QPS).
  * **Data Volume & Message Rate:** Total storage footprint and messages processed per second in queues.
* **Latency Profile Under Load:**
  * **Sublinear / Linear Degradation:** Response time remains stable or grows predictably as traffic scales.
  * **Superlinear Degradation:** Contention (CPU, locks, connection pools) causes cascading latency spikes and request timeouts.

---

## 2. Vertical Scaling vs. Horizontal Scaling

| Dimension | Vertical Scaling (Scale Up) | Horizontal Scaling (Scale Out) |
| :--- | :--- | :--- |
| **Mechanism** | Upgrades existing host (more CPU, RAM, NVMe SSD). | Adds more commodity instances behind a Load Balancer. |
| **Complexity** | Minimal; no distributed coordination or synchronization needed. | High; requires stateless tiers and distributed state management. |
| **Limits** | Hard hardware ceiling; exponential cost curve at the high end. | Elastic and virtually unbounded. |
| **Resilience** | Single Point of Failure (SPOF); upgrades typically cause downtime. | High fault tolerance via redundancy across multiple availability zones. |
| **Primary Use Case** | Relational databases and workloads requiring strict ACID/strong consistency. | Stateless application tiers, web servers, and distributed data layers. |

---

## 3. Stateless Application Architecture

Horizontal scaling of compute tiers requires making services stateless:

* **Session Offloading:** Keep application nodes free of local session state; persist sessions in external in-memory stores (e.g., Redis, Memcached) or use signed tokens (JWT).
* **Decoupled File Storage:** Store static assets and user uploads in distributed object storage (e.g., AWS S3, GCS) instead of local block storage.
* **Stateless Routing:** Avoid sticky sessions so load balancers can distribute traffic dynamically based on instance load or round-robin algorithms.

---

## 4. Database Scaling Patterns

| Strategy | Workload Pattern | Key Architectural Trade-offs & Mechanics |
| :--- | :--- | :--- |
| **Read Replicas** | Read-Heavy ($10:1$ to $100:1$) | Asynchronous replication offloads reads from the primary write node. Introduces **replication lag** and eventual consistency trade-offs. |
| **Caching Layers** | High-frequency, repetitive reads | Cache-Aside or Write-Through caching (e.g., Redis) mitigates database query pressure and reduces read latency. |
| **Database Sharding** | Write-Heavy / Multi-TB datasets | Horizontal partitioning across multiple DB instances using a shard/partition key. |
| **Specialized NoSQL** | High write ingestion | Engines with LSM-trees (e.g., Cassandra, ScyllaDB) optimize for high-throughput write appends. |

---

## 5. Senior-Level Failure Modes & Mitigations

* **Read-After-Write Inconsistency (Replication Lag):**
  * *Problem:* A client writes to the primary DB, immediately reads from a lagging replica, and receives stale data.
  * *Mitigations:* Route recent writers to the primary DB for a short time window (session tracking/LSN check), use cache-aside updates on write, or perform optimistic UI updates.
* **Connection Pool Exhaustion:**
  * *Problem:* Auto-scaling application instances ($10 \rightarrow 100$) causes thousands of direct database connections, exhausting DB memory and CPU via OS thread context switching.
  * *Mitigations:* Introduce a connection pooler/multiplexer (e.g., AWS RDS Proxy, PgBouncer) to route thousands of incoming application connections through a small, fixed set of active DB backend connections.
* **Hot Shards & Cross-Partition Queries:**
  * *Problem:* Non-uniform data access (e.g., an enterprise tenant generating disproportionate load) overwhelms single shards; non-partition-key queries trigger expensive scatter-gather operations.
  * *Mitigations:* Salt partition keys, isolate VIP/enterprise tenants into dedicated shards, and build secondary indexes via CQRS/event streams for non-shard-key lookups.

---

## Appendix: Terminology & Definitions

* **ACID (Atomicity, Consistency, Isolation, Durability):** A set of standard database properties that guarantee database transactions are processed reliably and safely, even during system crashes.
* **Cache-Aside Pattern:** A caching pattern where the application code first checks the cache; if a cache miss occurs, it reads from the database and updates the cache for future requests.
* **Connection Multiplexing:** A technique where many short-lived or low-activity client connections are mapped onto a smaller pool of persistent, active database connections to reduce database memory and CPU overhead.
* **CQRS (Command Query Responsibility Segregation):** An architectural design pattern that separates read operations (queries) from write/mutation operations (commands), often using separate data models and databases for each.
* **Eventual Consistency:** A consistency model where, if no new updates are made, all replicas across a distributed system will eventually converge and return the same data.
* **Hot Shard:** A condition in a sharded database where a single partition receives a disproportionate amount of read or write traffic compared to others, creating a performance bottleneck.
* **Key Salting:** Adding random or sequential prefix/suffix bits (salt) to a partition key to force high-volume data to spread evenly across multiple database shards.
* **Log Sequence Number (LSN):** A unique, monotonically increasing identifier assigned to each record in a database transaction log (Write-Ahead Log), used to measure replication progress and consistency.
* **LSM-Tree (Log-Structured Merge-tree):** A write-optimized data structure that buffers writes in memory (MemTable) before flushing them sequentially to disk in immutable files (SSTables).
* **p99 Latency (99th Percentile):** The maximum response time experienced by the fastest $99\%$ of requests, serving as a critical metric for tail latency and worst-case performance under load.
* **Read-After-Write Consistency (Read-Your-Own-Writes):** A consistency guarantee ensuring that once an actor updates a piece of data, any subsequent read operation performed by the same actor will reflect that update.
* **Replication Lag:** The delay between a data modification being committed on the primary database and that same update being applied and visible on a read replica.
* **Scatter-Gather (Fan-out) Query:** A query pattern where a coordinator sends requests simultaneously across all database shards, waits for individual responses, and aggregates the results before returning them to the client.
* **Single Point of Failure (SPOF):** A component or node whose individual failure causes the entire system to stop functioning.
* **Sticky Sessions (Session Affinity):** A routing technique where a load balancer binds a specific user's session to a single physical server, preventing dynamic load rebalancing.
* **Write-Through Cache:** A caching strategy where the application writes data synchronously to both the cache and the primary database before confirming completion to the caller.
