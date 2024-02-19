# ARCHITECTURE

</br>

![architecture](/imgage/Screen%20Shot%202024-02-19%20at%2021.02.51.png)

</br>

## Cluster
- The cluster is a set of one or more nodes that works together to hold the data.
- Clusters provide search capabilities and joined indexing across all nodes for entire data

</br>

## Node
- A node is an instance of Elasticsearch, which is used to store data. 
- It creates when an Elasticsearch instance starts running. 

</br>

## Shard
- A large amount of data can be stored using a cluster, but it can exceed the capability of a single server. To overcome this problem, Elasticsearch allows your index to be divided into several pieces, called **shard**. So, divide your index into several pieces to create shards. You can define the number of required shards while creating an index.

- While creating an index, the number of required shards can be defined. Each shard is independent and fully functional.

<br>

## Replica
- Replicas are an additional copy of shards. They perform queries just like a shard. Elasticsearch enables the users to create replicas of their indexes and shards.

- Elasticsearch provides replicas to avoid any type of failure and helps to enhance the availability of data in case of failure. The failure can be like - a node or shard is going to offline for some reason. Replication not only increases the availability of data but also improve the performance of search by executing parallel search operation in these replicas.

<br>

## Index as a DB
- Index is a container to store data similar to a database in the relational databases. An index contains a collection of documents that have similar characteristics or are logically related. If we take an example of an e-commerce website, there will be one index for products, one for customers, and so on. Indices are identified by the lowercase name. The index name is required to perform the add, update, or delete operations on the document.

<br>

## Type as a Table
- Type is a logical grouping of the documents within the index. In the previous example of product index, we can further group documents into types, like electronics, fashion, furniture, etc. Types are defined on the basis of documents having similar properties in it. It isn’t easy to decide when to use type over index. Indices have more overheads, so sometimes, it is better to use different types in the same index for better performance. There are a couple of restrictions to use types as well. Two fields having the same name in a different type of document should be of the same data type (string, date, etc).

<br>

## Document as a Row in DB