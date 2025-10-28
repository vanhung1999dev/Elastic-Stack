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
