# ARCHITECTURE

</br>

![architecture](/imgage/Screen%20Shot%202024-02-19%20at%2021.02.51.png)

</br>

## Cluster
- A cluster is a collection of one or more nodes (servers) that together hold all your data and provide indexing and search capabilities.
- Each cluster has a unique name (default: elasticsearch).

- A cluster has:
  - **Master node** → manages cluster state.
  - **Data nodes** → store data, handle search and indexing.
  - **Coordinating nodes** → route requests.
  - **Ingest nodes** → preprocess documents (optional).

</br>

## Node
- A node is a single instance of Elasticsearch running on a machine (physical or virtual).
- Each node has:
  - A name (for identification)
  - A role (master, data, ingest, coordinating)
  - Nodes automatically join a cluster using the cluster name.
 
| Node Type         | Role                                                  |
| ----------------- | ----------------------------------------------------- |
| Master Node       | Controls cluster state, shard allocation              |
| Data Node         | Stores actual data, handles CRUD & search             |
| Ingest Node       | Runs pipelines to transform documents before indexing |
| Coordinating Node | Routes client requests and merges results             |

</br>


## Index
- Similar to a database table in relational DBs.
- An index is a logical namespace that groups related documents.
- Each index is divided into smaller units called shards.
  
```
POST /users/_doc/1
{
  "name": "Alice",
  "age": 30
}
```

→ Creates a document in the users index. <br>




## Shard
- A large amount of data can be stored using a cluster, but it can exceed the capability of a single server. To overcome this problem, Elasticsearch allows your index to be divided into several pieces, called **shard**. So, divide your index into several pieces to create shards. You can define the number of required shards while creating an index.
- While creating an index, the number of required shards can be defined. Each shard is independent and fully functional.
- Each index is split into shards for scalability and performance.
- Each shard is an independent Lucene index.

Two types: <br>

Primary shards → store original data. <br>
Replica shards → copies for fault tolerance & load balancing. <br>

Example: <br>
If you have an index with 3 primary shards and 1 replica → total = 6 shards.

<br>

## Replica
- Replicas are an additional copy of shards. They perform queries just like a shard. Elasticsearch enables the users to create replicas of their indexes and shards.

- Elasticsearch provides replicas to avoid any type of failure and helps to enhance the availability of data in case of failure. The failure can be like - a node or shard is going to offline for some reason. Replication not only increases the availability of data but also improve the performance of search by executing parallel search operation in these replicas.

<br>

## Segment (Lucene Level)
- Inside each shard, data is stored in Lucene segments (immutable files).
- When new data is indexed:
- It’s written to an in-memory buffer and translog.
- Later, it’s flushed to disk into a new segment.
Over time, smaller segments are merged for efficiency.

# Data flow

## ⚙️ (Indexing)
- Client sends a JSON document to an Elasticsearch node.
- The coordinating node:
  - Determines the index.
  - Calculates which primary shard should handle it (hash(_id) % number_of_primary_shards).
- The request is routed to that shard’s node.
- The primary shard indexes the document and forwards it to its replicas.
- Acknowledgment is sent back to the client.

## (Search)

- Client sends a search query.
- The coordinating node:
- Broadcasts the query to all relevant shards (primary or replicas).
- Each shard executes the query locally (Lucene handles the actual search).
- Results (document IDs + scores) are sent back to the coordinator.
- The coordinator merges, sorts, and returns the final response to the client.
