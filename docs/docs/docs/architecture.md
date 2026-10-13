# QuickNotes System Architecture & Scaling Specification

## 1. System Requirements

### 1.1 Functional Requirements
1. **Note Management:** Users must be able to create, read, update, archive, pin, and permanently delete notes.
2. **Organization:** Users can create tags and categorize notes with one or more tags.
3. **Filtering & Search:** Users can filter their notes feed by tag, status (active/archived/pinned), and search by keyword.
4. **Multi-Device Synchronization:** Changes made on mobile, tablet, or desktop web clients must reflect consistently across sessions.

### 1.2 Non-Functional Requirements
1. **High Availability:** $99.9\%$ uptime (~8.76 hours maximum downtime per year).
2. **Low Latency:** p95 read latency $< 100\text{ms}$; p95 write response time $< 200\text{ms}$.
3. **Data Durability & ACID Consistency:** Zero data loss for saved notes; atomic transactional persistence.
4. **Scalability:** Horizontal scaling capacity to support $1,000,000$ registered accounts.
5. **Security & Privacy:** TLS 1.3 encryption in transit, AES-256 at rest, strict tenant data isolation (users cannot access other users' notes).

---

## 2. Capacity & Load Estimation (1 Million Users)

### 2.1 Assumptions & Active User Calculations
* **Total Registered Users:** $1,000,000$.
* **Daily Active Ratio:** $20\%$ active each day $\implies \mathbf{200,000\text{ DAU}}$.
* **Write Activity:** Each active user creates or updates $2$ notes per day:
  $$\text{Daily Writes} = 200,000 \times 2 = 400,000\text{ writes/day}$$
* **Read Activity:** Each active user browses or reloads their notes list $20$ times per day:
  $$\text{Daily Reads} = 200,000 \times 20 = 4,000,000\text{ reads/day}$$
* **Seconds per day:** $86,400\text{ seconds}$.
* **Peak Traffic Multiplier:** $5\times$ average load.

### 2.2 Throughput Estimates (QPS)
* **Write Throughput (POST / PUT / DELETE):**
  * Average Writes/sec: $\frac{400,000}{86,400} \approx \mathbf{4.63\text{ writes/sec}}$.
  * Peak Writes/sec ($5\times$): $4.63 \times 5 \approx \mathbf{23.15\text{ writes/sec}}$.
* **Read Throughput (GET /notes):**
  * Average Reads/sec: $\frac{4,000,000}{86,400} \approx \mathbf{46.30\text{ reads/sec}}$.
  * Peak Reads/sec ($5\times$): $46.30 \times 5 \approx \mathbf{231.50\text{ reads/sec}}$.
* **Read-to-Write Ratio:** $4,000,000 : 400,000 = \mathbf{10 : 1}$ (Read-heavy profile).

### 2.3 Storage Growth per Year
* Average note record footprint:
  * Title ($\le 100$ bytes) + Body ($\approx 500$ bytes) + Metadata & Tag references ($\approx 150$ bytes) $\approx 750\text{ bytes}$ per row.
* Annual Note Creations:
  $$400,000\text{ notes/day} \times 365\text{ days} = 146,000,000\text{ notes/year}$$
* Annual Raw Storage:
  $$146,000,000 \times 750\text{ bytes} \approx 109,500,000,000\text{ bytes} \approx \mathbf{109.5\text{ GB/year}}$$
* Adding relational indexes, WAL logs, and backup overhead ($3\times$ factor): $\approx \mathbf{328.5\text{ GB/year}}$. This comfortably fits within a standard managed PostgreSQL cloud cluster (e.g., AWS RDS or Aurora) with room for multi-year growth.

---

## 3. System Architecture Diagram

```
[Mobile / Web Client]
          |
          | 1. DNS Query (GeoDNS Anycast)
          v
      [Cloud DNS]
          |
          | 2. HTTPS Request (TLS 1.3)
          +-------------------------------------------------------------+
          |                                                             |
          v                                                             v
    [Edge CDN]                                             [Application Load Balancer]
  (Static Assets:                                            (AWS ALB / NGINX Plus)
  HTML, CSS, JS)                                                        |
                                                                        | Reverse Proxy (Round-Robin)
                                                                        v
                                                   +----------------------------------------+
                                                   |       Stateless App Server Fleet       |
                                                   |  [App Server 1]      [App Server 2]    |
                                                   +----------------------------------------+
                                                        |                               |
                                           Cache Query  |                               | Read / Write
                                                        v                               v
                                              [In-Memory Cache]               [Database Proxy / LB]
                                            (Redis Cluster: Feeds)            (PgBouncer Connection Pool)
                                                        ^                               /        \
                                                        |                              / Writes   \ Reads
                                                        | Invalidate / Warm           v            v
                                                        |                     [Primary DB]    [Read Replica]
                                                        |                    (PostgreSQL 16)  (PostgreSQL 16)
                                                        |                           |                ^
                                                        | Enqueue Task              | Async WAL Sync |
                                                        v                           +----------------+
                                                 [Message Queue]
                                                (AWS SQS / RabbitMQ)
                                                        |
                                                        | Consume Task
                                                        v
                                                 [Worker Fleet]
                                           (Search Indexing & Analytics)
```

---

## 4. Component Explanations (One Sentence Per Component)

1. **DNS (Cloud DNS):** Translates the domain name `api.quicknotes.app` into optimal server IP addresses using Anycast routing to direct clients to the nearest point of presence.
2. **CDN (Content Delivery Network):** Caches and serves static frontend client bundles (`index.html`, `style.css`, `api.js`) from global edge locations, minimizing latency and shielding backend servers from static asset traffic.
3. **Application Load Balancer (ALB):** Distributes incoming HTTPS API requests evenly across healthy application servers while terminating TLS and performing continuous health checks.
4. **App Servers (App Server 1 & App Server 2):** Runs the stateless application business logic, validates JWT tokens and input schemas, orchestrates cache lookups, and communicates with persistence layers.
5. **Cache (Redis Cluster):** Stores precomputed active user note lists and session tokens in memory to deliver sub-millisecond responses and eliminate repetitive database reads.
6. **Primary Database (PostgreSQL Master):** Serves as the authoritative source of truth for transactional writes, enforcing relational schemas, unique constraints, and referential integrity.
7. **Read Replica (PostgreSQL Replica):** Handles read queries asynchronously replicated from the primary database, scaling horizontal query capacity without write lock contention.
8. **Message Queue (AWS SQS):** Buffers non-critical asynchronous tasks (such as full-text search indexing, telemetry, and external notifications) to prevent write latency spikes.
9. **Worker (Background Processor):** Pulls asynchronous background jobs from the message queue to update search indexes and execute batch maintenance tasks independently of user requests.

---

## 5. Step-by-Step Request Flows

### 5.1 Read Flow (`GET /api/v1/notes`)
1. **Client Request:** The user opens the app; client sends `GET /api/v1/notes?limit=10` with `Authorization: Bearer <jwt>`.
2. **Routing:** DNS resolves the host to the **Application Load Balancer**, which forwards the request to an available **App Server** via least-connections routing.
3. **Authentication:** The App Server validates the JWT signature, extracts `user_id = 104`, and checks permissions.
4. **Cache Lookup (Cache-Aside):** The App Server queries the **Redis Cache** for key `user:104:notes_feed`.
   * **Cache Hit ($90\%$ of calls):** Redis returns the cached JSON note list in $<2\text{ms}$. The server returns `200 OK` directly.
   * **Cache Miss ($10\%$ of calls):** The App Server queries the **Read Replica** via PgBouncer connection pool executing the indexed SQL query:
     ```sql
     SELECT id, title, body, is_pinned, updated_at FROM notes 
     WHERE user_id = 104 AND is_archived = FALSE 
     ORDER BY is_pinned DESC, updated_at DESC LIMIT 10;
     ```
5. **Cache Warm:** The App Server writes the result back into Redis with a 5-minute TTL (`SET user:104:notes_feed <json> EX 300`).
6. **Response:** The App Server returns HTTP `200 OK` with the note payload to the client.

### 5.2 Write Flow (`POST /api/v1/notes`)
1. **Client Submission:** User fills out the form and clicks "Add Note"; client issues `POST /api/v1/notes` with JSON payload `{ "title": "My Note", "body": "Content" }`.
2. **Load Balancing & Validation:** The **Application Load Balancer** forwards the request to an App Server. The server validates that `title` is present, non-empty, and $\le 100$ characters.
3. **Primary Database Write:** The App Server executes an `INSERT` statement against the **Primary Database** within an ACID transaction:
   ```sql
   INSERT INTO notes (user_id, title, body) VALUES (104, 'My Note', 'Content') RETURNING id, created_at;
   ```
4. **Cache Invalidation:** The App Server invalidates the user's cached feed in Redis:
   ```redis
   DEL user:104:notes_feed
   ```
5. **Async Job Enqueue:** The App Server publishes an asynchronous event `{ "event": "note_created", "note_id": 4820, "user_id": 104 }` to the **Message Queue**.
6. **Immediate Response:** The App Server responds to the client with `201 Created` containing the full note object (response time $<50\text{ms}$).
7. **Background Indexing:** A **Worker** pulls the event from the message queue and updates the background full-text search index asynchronously without blocking the client.

---

## 6. Architectural Trade-offs

### Trade-off 1: Cache-Aside (Lazy Loading) vs. Write-Through Caching
* **Decision Chosen:** Cache-Aside pattern.
* **Why Chosen:** Under Cache-Aside, notes are only loaded into Redis when requested. Because active users check their notes frequently while inactive users rarely check, this prevents unused notes from consuming precious Redis RAM.
* **Drawback & Mitigation:** The first read after a write incurs a cache miss penalty (~15ms SQL query). This is mitigated by our indexed B-Tree schema and short query execution times.

### Trade-off 2: Asynchronous Database Replication vs. Synchronous Replication
* **Decision Chosen:** Asynchronous Read Replication with PostgreSQL streaming WAL.
* **Why Chosen:** Asynchronous replication decouples write transaction commits from network latency between database nodes. The primary database commits writes instantly without waiting for read replicas to confirm, guaranteeing high write throughput ($>1,000$ writes/sec capacity).
* **Drawback & Mitigation:** Slight replica lag ($<100\text{ms}$). If a user creates a note and immediately reads from a replica, there is a theoretical risk of stale data. We mitigate this by reading recently modified records directly from the Primary DB or serving the newly created item directly from the client's local memory.

---

## 7. Avoiding Single Points of Failure (SPOF)

1. **DNS Redundancy:** Anycast DNS with automatic multi-datacenter failover guarantees global name resolution resilience.
2. **Stateless Compute Fleet:** Application servers do not store state locally. If App Server 1 crashes, the Load Balancer health check detects failure in $<2\text{ seconds}$ and reroutes all traffic to App Server 2 without dropping sessions.
3. **Multi-AZ Database Failover:** The Primary Database utilizes a synchronous standby replica in an alternate Availability Zone (Multi-AZ). If the primary node experiences hardware failure, the system promotes the standby to primary in $<30\text{ seconds}$ with zero data loss.
4. **Redis Sentinel / Multi-Node Cluster:** Redis runs in a 3-node cluster with auto-failover. If the cache is temporarily unavailable, the app servers fail open and route read requests directly to read replicas.
5. **Dead-Letter Queues (DLQ):** The Message Queue routes poison pills or failing jobs to a Dead-Letter Queue after 3 retries, ensuring bad tasks never block worker pipeline throughput.
