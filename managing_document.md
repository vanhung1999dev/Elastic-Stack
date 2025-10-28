<summary> Creating Indexing</summary>
<details>
## 📦 1. What Is an Index?

An **index** is a logical namespace that maps to one or more **primary shards**, and each shard can have **zero or more replica shards**.  
Internally, each shard is a **Lucene index** — a full-text search engine library that stores inverted indices.

Each index has:
- **Settings** → Number of shards, replicas, refresh intervals, analyzers, etc.
- **Mappings** → Field definitions (data types, analyzers, etc.)
- **Aliases** → Alternative names for routing or versioning.


### 🧩 Example Request
```bash
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name":    { "type": "text" },
      "price":   { "type": "float" },
      "in_stock": { "type": "boolean" }
    }
  }
}
```

## 🧠 What Happens Internally

#### Request Received by Coordinating Node
- The node that receives the request acts as the coordinator.
- It validates the JSON structure and checks if the index name is valid and unique.

#### Cluster State Update
- The coordinator sends a CreateIndexClusterStateUpdateRequest to the Master Node.
- The master node is the only one that can modify the cluster state (metadata about indices and shards).
- The master validates:
  - Name uniqueness
  - Template matching (if index templates are defined)
  - Settings correctness
    
#### Shard Metadata Creation
  - Master node adds entries for each primary and replica shard into the cluster metadata.
  - These are not yet real shards on disk — they are planned allocations.

#### Shard Allocation
- The master node decides which data nodes will host which shards.
- It uses the Cluster Routing Table and allocation deciders (like disk space, node load, etc.).
- Once assigned, the master updates the cluster state and propagates it to all nodes.

#### Shard Initialization
  - The assigned data nodes create actual directories on disk, initialize Lucene indices, and write metadata files under:
  ```
  /var/lib/elasticsearch/nodes/0/indices/<index_uuid>/0/index/
  ```
  - Each primary shard is created empty — ready to accept new documents.

#### Acknowledgment
- Once all primary shards are successfully created, the master node responds with:
- { "acknowledged": true, "shards_acknowledged": true, "index": "products" }

### Shards, Replicas, and Their Roles
| Type              | Description                                    | Behavior                                                |
| ----------------- | ---------------------------------------------- | ------------------------------------------------------- |
| **Primary Shard** | The main shard that handles indexing requests. | Each document is first written here.                    |
| **Replica Shard** | A copy of a primary shard.                     | Provides fault tolerance and increases read throughput. |

```
number_of_shards: 3
number_of_replicas: 1
```
If the cluster has 3 data nodes, Elasticsearch tries to balance: <br>
```
Node 1 → P0, R1
Node 2 → P1, R2
Node 3 → P2, R0
```

#### Auto-Creation of Indices
Elasticsearch can auto-create an index when you index a document into a non-existent index: <br>
```
POST /users/_doc/1
{ "name": "Alice" }
```

This will create an index users with default settings: <br>
```
"number_of_shards": 1,
"number_of_replicas": 1
```

</details>


<summary>Delete indexing</summary>
<details>
 ### 🧩 Example Request
```
DELETE /products
```

### 🧠 What Happens Internally

#### Request Validation
- The coordinator node forwards the request to the master node.

#### Cluster State Update
- The master removes the index metadata and shard routing information from the cluster state.

#### Shard Deletion
- Each data node hosting shards for that index receives a notification.
- The data node:
- Closes the shard’s open file handles.
  - Deletes the shard’s data from disk.
  - Removes translog and Lucene segments.

#### Acknowledgment
- After all nodes confirm deletion, Elasticsearch returns:
- { "acknowledged": true }

#### Cleanup
- Deleted index names are removed from caches and memory.
- Disk space is reclaimed asynchronously.

</details>

<summary>Update Document<summary>
<details>


## 🧠 1. Overview

In Elasticsearch, documents are **immutable** — meaning you **cannot modify** them directly.  
When you update a document, Elasticsearch actually performs these steps internally:

1. Retrieves the existing document.
2. Modifies it in memory (applies your changes).
3. Deletes the old document version.
4. Indexes the new version with an incremented `_version`.

So, an **update** is really a **delete + reindex** operation under the hood.

---

## 🧾 2. Updating a Document Field

You can use the `_update` API to modify an existing field.

### Example: Update an existing field

```bash
POST /my_index/_update/1
{
  "doc": {
    "price": 49.99
  }
}
```

✅ This updates the field price to 49.99 for the document with ID 1. <br>
If the document doesn’t exist, Elasticsearch will return a 404 unless you specify doc_as_upsert. <br>

### 🧩  Adding a New Field

To add a new field to an existing document, you can also use the same _update API. <br>

```
POST /my_index/_update/1
{
  "doc": {
    "discount": 10
  }
}
```

✅ This adds a new field called discount with value 10. <br>
If this field doesn’t exist in the mapping, Elasticsearch will dynamically add it (if dynamic mapping is enabled). <br>

### ⚙️ Using doc_as_upsert
If you want to create the document when it doesn’t exist, you can use doc_as_upsert. <br>
```
POST /my_index/_update/2
{
  "doc": {
    "price": 59.99,
    "stock": 100
  },
  "doc_as_upsert": true
}
```

✅ If document ID 2 exists → it’s updated. <br>
✅ If it doesn’t exist → it’s created with the given fields <br>

### 🚫 Replacing the Entire Document
If you want to completely replace a document, use the standard PUT API: <br>
```
PUT /my_index/_doc/1
{
  "name": "Smartphone X",
  "price": 599,
  "stock": 150
}
```

✅ This overwrites the old document with the new one (same _id). <br>


### 🧠 What Is a Script Update?

When you use the **Update API** with a `script`, Elasticsearch:
1. Retrieves the existing `_source` document.
2. Executes your script logic in the cluster.
3. Applies the modifications to `_source`.
4. Reindexes the modified version (internally).

---

## 🧩 Basic Script Update Example

Let’s say we have a document:

```json
PUT /users/_doc/1
{
  "name": "Alice",
  "age": 25,
  "login_count": 3
}
```

We can increment her login_count by one using: <br>
```
POST /users/_update/1
{
  "script": {
    "source": "ctx._source.login_count += params.inc",
    "lang": "painless",
    "params": {
      "inc": 1
    }
  }
}
```

| Field         | Description                                   |
| ------------- | --------------------------------------------- |
| `ctx._source` | The document’s source JSON (fields & values). |
| `ctx._id`     | Document ID.                                  |
| `ctx._index`  | Index name.                                   |
| `ctx.op`      | Operation type (`update`, `delete`, `none`).  |
| `params`      | Custom parameters passed in your request.     |

</details>

<summary>Routing</summary>
<details>

When you index or search for a document in Elasticsearch, the system must answer two critical questions:

1. **Where should this document be stored?**
2. **How can this document be found later?**


## 🧱 What Is Routing?

**Routing** in Elasticsearch determines **which shard** a document belongs to.  
Every document in an index is stored in **exactly one primary shard**, and Elasticsearch uses a **routing value** to decide *which* one.

---

## ⚙️ The Routing Formula

Elasticsearch uses this internal formula:
```
shard = hash(_routing) % number_of_primary_shards
```


Where:

| Component | Description |
|------------|-------------|
| `hash()` | A consistent hashing function applied to the routing value. |
| `_routing` | The value used for routing. Default is the document’s `_id`. |
| `number_of_primary_shards` | The total number of primary shards configured for the index (fixed at creation). |

✅ Example:
- Index has **5 primary shards**.
- Document ID is `"user_42"`.
- Elasticsearch computes:  
  `hash("user_42") % 5 → shard 2`  
  → Document stored in **primary shard 2** (and its replicas).

---

## 📦 Default Routing: `_id`

By default, the routing key is simply the document’s **`_id`**.

```json
PUT /users/_doc/1
{
  "name": "Alice",
  "age": 25
}
```

Elasticsearch rehashes the same _id → computes the same shard number → directly looks up that shard. <br>

That’s how it instantly finds the document without searching the whole cluster. <br>
</details>


<summary>How to Read data</summary>
<details>
  # 🔍 How Elasticsearch Reads Data

Now that we understand **routing**, we can explore how Elasticsearch **reads data** — that is, how it processes **search** and **get** requests efficiently across a distributed cluster.

This process involves:
- The **Coordinating Node**
- The **Routing mechanism**
- **Replica shards**
- **Adaptive Replica Selection (ARS)**

---

## 🧭 1. The Big Picture

When you send a query (e.g., `GET`, `SEARCH`, `MULTI_GET`, `SCROLL`, etc.) to Elasticsearch:

1. The request hits one node — any node in the cluster.
2. That node becomes the **Coordinating Node**.
3. The coordinating node:
   - Determines **which shards** hold the data (using routing).
   - Forwards requests to the relevant **primary or replica shards**.
4. Each shard executes the query locally and returns results.
5. The coordinating node merges, sorts, and returns a unified response.

---

## 🏗️ 2. The Coordinating Node

Every node in Elasticsearch can act as a **coordinating node**.

### Responsibilities:
| Step | Role | Description |
|------|------|-------------|
| 1️⃣ | Receive request | Accepts the client’s query or GET request. |
| 2️⃣ | Routing resolution | Computes the shard(s) using `hash(_routing) % num_primary_shards`. |
| 3️⃣ | Request fan-out | Sends sub-requests to target shards (primary or replicas). |
| 4️⃣ | Merge phase | Collects responses and merges or reduces results. |
| 5️⃣ | Return response | Sends the final aggregated result back to the client. |

---

## ⚙️ 3. Read Paths: Document Retrieval vs Search Query

### a) **Single Document Read (GET /index/_doc/id)**

When you fetch one document:

1. The coordinating node computes **the shard** using the routing formula.
2. It picks **either the primary or one of the replicas** that holds the shard.
3. That shard directly retrieves the document and returns it.
4. The coordinating node sends the response to the client.

✅ Fast — constant-time lookup, no broadcast.

---

### b) **Search Query (POST /index/_search)**

A search query usually spans **multiple shards**.

#### 🔸 Phase 1 — Query Phase
1. Coordinating node determines which shards to query (based on index routing).
2. Sends the query to **one replica of each relevant shard**.
3. Each shard executes the query locally:
   - Applies filters
   - Collects matching document IDs and scores (top N)
   - Returns only metadata (not full documents)

#### 🔸 Phase 2 — Fetch Phase
1. Coordinating node merges shard results, sorts globally (by score, time, etc.).
2. Determines the top N global hits.
3. Fetches full `_source` documents for those hits from the corresponding shards.
4. Merges them into the final response.

---

## 🧩 4. How Shard Replicas Help Reads

Each shard has:
- **1 primary shard**
- **0 or more replica shards**

### ✅ Reads can happen from:
- Primary shard
- Any replica shard

This improves:
- **Throughput** (read load is distributed)
- **Availability** (if a primary is down, replicas still serve reads)

---

## ⚡ 5. Adaptive Replica Selection (ARS)

By default, Elasticsearch can read from any replica of a shard.  
But not all replicas are equal — some nodes are busier or slower.

**Adaptive Replica Selection (ARS)** dynamically decides **which replica** to read from based on **real-time performance metrics**.

---

### 🧠 How ARS Works

Each node maintains performance statistics for shards, including:

| Metric | Meaning |
|---------|----------|
| Response time | How fast a shard replies to requests |
| Queue size | How many operations are waiting on that node |
| Service time | Average time to process previous requests |

When a coordinating node receives a read request:
1. It calculates a **score** for each replica shard:
```
score = (outstanding_requests + 1) * moving_avg_latency
```

2. The replica with the **lowest score** is chosen.
3. The system continuously adjusts based on real response times.

✅ Result: **Fastest available replica** handles the next read.

---

### 🧮 Example

Imagine `user_123`’s document is stored in:
- Shard 2 primary → Node A
- Replica 1 → Node B
- Replica 2 → Node C

If Node A is busy but Node C responds fastest,
Elasticsearch’s ARS will route reads for shard 2 to **Node C** next time.

This ensures:
- Lower latency
- Load balancing
- Faster failover recovery

---

## 📡 6. Putting It All Together

### Example Flow: `GET /users/_doc/1`

1. **Client → Node X (Coordinating node)**
- Receives the request.

2. **Routing**
- Node X computes `hash("1") % 5 → shard 2`.

3. **Replica Selection**
- Node X uses ARS to choose the best replica for shard 2.

4. **Data Retrieval**
- Node C (holding shard 2 replica) returns document `_id = 1`.

5. **Response**
- Node X merges response (trivial here) → returns to client.


Each node runs the query locally and sends partial results.  
Node 1 merges all hits → sorts → fetches top N → returns combined result.

---

## Summary

| Concept | Description |
|----------|-------------|
| **Coordinating Node** | Receives client request, routes it to relevant shards, merges results. |
| **Routing** | Determines which shard holds the document (via hash). |
| **Shard Replica** | Each shard can have multiple replicas that serve reads. |
| **Adaptive Replica Selection (ARS)** | Chooses the fastest replica dynamically using latency and queue metrics. |
| **Query Phase** | Each shard executes query locally and returns hits (metadata only). |
| **Fetch Phase** | Coordinating node fetches full `_source` for top hits and merges results. |

---

## Key Takeaways

- Elasticsearch distributes both **reads and writes** across nodes.
- The **coordinating node** acts as a request router and aggregator.
- **Routing** determines which shard(s) hold your data.
- **Adaptive Replica Selection (ARS)** ensures reads are always directed to the *fastest* replicas.
- Search operations follow a **Query → Fetch** two-phase model.
- This architecture allows horizontal scaling with low latency and fault tolerance.

---

### 📘 References

- [Elasticsearch Docs — Search Execution and Coordinating Node](https://www.elastic.co/guide/en/elasticsearch/reference/current/execution-control.html)
- [Elastic Blog — Adaptive Replica Selection Explained](https://www.elastic.co/blog/improving-resiliency-and-performance-with-adaptive-replica-selection)
- [Elastic Architecture Overview](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)


</details>

<summary>How to Write data</summary>
<details>
  
We’ll explore how Elasticsearch handles indexing, updates, deletions, and replication using **primary terms**, **sequence numbers**, and **checkpoints**.

---

## 🗂 1. High-Level Overview

When you **index** or **update** a document, the request flows through the following path:

1. **Client → Coordinating Node**  
   - The client sends a write request (e.g., `POST /index/_doc/1`).
   - The coordinating node decides which shard (primary) should handle it — using **routing** (usually `_id`-based hash).

2. **Coordinating Node → Primary Shard**  
   - The coordinating node forwards the request to the **primary shard** responsible for that document.

3. **Primary Shard → Replica Shards**  
   - The primary shard processes the write, assigns metadata (sequence number, term), then forwards it asynchronously to replica shards.

4. **Acknowledgment**  
   - Once replicas confirm, the primary acknowledges success to the coordinating node, which then responds to the client.

---

## ⚙️ 2. Key Concepts

### 🔹 Primary Term

- Each **primary shard** has a **primary term**.
- It increments every time a **shard is promoted** (e.g., when the old primary fails and a replica becomes primary).
- Used to ensure **write consistency** — so replicas don’t apply old operations after leadership changes.

> **Example:**  
> - Term 1 → initial primary  
> - Term 2 → new primary after failover  
>  
> If an operation with term 1 arrives after term 2 starts, it’s ignored — ensuring no stale writes.

---

### 🔹 Sequence Number (seq_no)

- Every write operation (index, update, delete) is assigned a **monotonically increasing sequence number**.
- Used to maintain **order** of operations across replicas.
- Each shard maintains:
  - `max_seq_no`: the latest sequence number assigned.
  - `local_checkpoint`: last sequence number fully processed on this shard.

> **Purpose:**  
> Ensures that all shards (primary + replicas) apply operations in **exactly the same order**.

---

### 🔹 Checkpoints

There are two types of checkpoints:

1. **Local Checkpoint**
   - Tracks the highest sequence number that has been **successfully processed** on a shard.

2. **Global Checkpoint**
   - The highest sequence number for which **all shards (primary and replicas)** have processed the operation.
   - Ensures consistency across the shard group.

> **Analogy:**  
> Think of the global checkpoint as “everyone’s caught up to here.”

---

## 📤 3. The Write Flow Step-by-Step

Let’s break it down in detail:

### 1️⃣ The Client Sends a Write Request

```http
POST /products/_doc/1
{
  "name": "iPhone 16",
  "price": 1299
}
```

The coordinating node determines the target shard using the routing formula: <br>
```
shard = hash(_id) % number_of_primary_shards
```

### Primary Shard Processes the Write
The primary shard: <br>
- Assigns a sequence number (seq_no = 101 for example).
- Applies the operation locally (writes it to the translog and Lucene segment).
- Forwards the operation to all replica shards.

### Replicas Apply the Write
Each replica: <br>
- Verifies it’s still on the same primary term.
- Applies the operation with the given sequence number.
- Updates its local checkpoint.

Once all replicas confirm, the global checkpoint may advance. <br>


### Acknowledgment to Client
When: <br>
- The primary and all active replicas have written the operation to their translogs, and
- The translog has been safely fsynced to disk
- Then the write is acknowledged to the client.

🧩 Depending on the index’s replication settings (wait_for_active_shards), Elasticsearch might wait for some or all replicas. <br>

### 🔄 Replication Model
- Elasticsearch uses an asynchronous primary-replica replication model.
- Primary executes the operation first.
- Replicas catch up via replication threads.
- The system ensures eventual consistency via sequence numbers and global checkpoints.

Replication Steps: <br>

- Client → Coordinating node → Primary
- Primary executes and assigns seq_no + term
- Primary → Replicas (send write operation)
- Replicas confirm → Global checkpoint updated
- Client receives ACK

### Handling Failures

- If the primary shard fails before the write completes:
- A replica is promoted to primary.
- The new primary has a higher term.
- Pending writes from the old term are discarded if not replayed.
- Replicas that missed operations catch up from the translog of the new primary.

| Concept               | Description                                  | Example                  |
| --------------------- | -------------------------------------------- | ------------------------ |
| **Primary Term**      | Version of shard leadership                  | Term 2 after failover    |
| **Sequence Number**   | Global operation order per shard             | 101                      |
| **Local Checkpoint**  | Latest confirmed operation on shard          | 100                      |
| **Global Checkpoint** | Latest confirmed operation across all shards | 98                       |
| **Translog**          | Durable write log for recovery               | `/data/nodes/0/translog` |

</details>
