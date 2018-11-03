---
title: Data partitioning guidance
description: Guidance for how to separate partitions to be managed and accessed separately.
author: dragon119
ms.date: 11/03/2018

---
# Data partitioning strategies

This article describes some strategies for partioning data in various Azure data stores. For general guidance about when to partition data and best practices, see [Data partioning](./data-partitioning.md)

## Partitioning strategies for Azure SQL Database
Azure SQL Database is a relational database-as-a-service that runs in the cloud. It is based on Microsoft SQL Server. A relational database divides information into tables, and each table holds information about entities as a series of rows. Each row contains columns that hold the data for the individual fields of an entity. The page [What is Azure SQL Database?] on the Microsoft website provides detailed documentation about creating and using SQL databases.

## Horizontal partitioning with Elastic Database
A single SQL database has a limit to the volume of data that it can contain. Throughput is constrained by architectural factors and the number of concurrent connections that it supports. The Elastic Database feature of SQL Database supports horizontal scaling for a SQL database. Using Elastic Database, you can partition your data into shards that are spread across multiple SQL databases. You can also add or remove shards as the volume of data that you need to handle grows and shrinks. Using Elastic Database can also help reduce contention by distributing the load across databases.

> [!NOTE]
> Elastic Database is a replacement for the Federations feature of Azure SQL Database. Existing SQL Database Federation installations can be migrated to Elastic Database by using the Federations migration utility. Alternatively, you can implement your own sharding mechanism if your scenario does not lend itself naturally to the features that are provided by Elastic Database.
>
>

Each shard is implemented as a SQL database. A shard can hold more than one dataset (referred to as a *shardlet*). Each database maintains metadata that describes the shardlets that it contains. A shardlet can be a single data item, or it can be a group of items that share the same shardlet key. For example, if you are sharding data in a multitenant application, the shardlet key can be the tenant ID, and all data for a given tenant can be held as part of the same shardlet. Data for other tenants would be held in different shardlets.

It is the programmer's responsibility to associate a dataset with a shardlet key. A separate SQL database acts as a global shard map manager. This database contains a list of all the shards and shardlets in the system. A client application that accesses data connects first to the global shard map manager database to obtain a copy of the shard map (listing shards and shardlets), which it then caches locally.

The application then uses this information to route data requests to the appropriate shard. This functionality is hidden behind a series of APIs that are contained in the Azure SQL Database Elastic Database Client Library, which is available as a NuGet package. The page [Elastic Database features overview] on the Microsoft website provides a more comprehensive introduction to Elastic Database.

> [!NOTE]
> You can replicate the global shard map manager database to reduce latency and improve availability. If you implement the database by using one of the Premium pricing tiers, you can configure active geo-replication to continuously copy data to databases in different regions. Create a copy of the database in each region in which users are based. Then configure your application to connect to this copy to obtain the shard map.
>
> An alternative approach is to use Azure SQL Data Sync or an Azure Data Factory pipeline to replicate the shard map manager database across regions. This form of replication runs periodically and is more suitable if the shard map changes infrequently. Additionally, the shard map manager database does not have to be created by using a Premium pricing tier.
>
>

Elastic Database provides two schemes for mapping data to shardlets and storing them in shards:

* A **list shard map** describes an association between a single key and a shardlet. For example, in a multitenant system, the data for each tenant can be associated with a unique key and stored in its own shardlet. To guarantee privacy and isolation (that is, to prevent one tenant from exhausting the data storage resources available to others), each shardlet can be held within its own shard.

![Using a list shard map to store tenant data in separate shards](./images/data-partitioning/PointShardlet.png)

*Figure 4. Using a list shard map to store tenant data in separate shards*

* A **range shard map** describes an association between a set of contiguous key values and a shardlet. In the multitenant example described previously, as an alternative to implementing dedicated shardlets, you can group the data for a set of tenants (each with their own key) within the same shardlet. This scheme is less expensive than the first (because tenants share data storage resources), but it also creates a risk of reduced data privacy and isolation.

![Using a range shard map to store data for a range of tenants in a shard](./images/data-partitioning/RangeShardlet.png)

*Figure 5. Using a range shard map to store data for a range of tenants in a shard*

Note that a single shard can contain the data for several shardlets. For example, you can use list shardlets to store data for different non-contiguous tenants in the same shard. You can also mix range shardlets and list shardlets in the same shard, although they will be addressed through different maps in the global shard map manager database. (The global shard map manager database can contain multiple shard maps.) Figure 6 depicts this approach.

![Implementing multiple shard maps](./images/data-partitioning/MultipleShardMaps.png)

*Figure 6. Implementing multiple shard maps*

The partitioning scheme that you implement can have a significant bearing on the performance of your system. It can also affect the rate at which shards have to be added or removed, or the rate at which data must be repartitioned across shards. Consider the following points when you use Elastic Database to partition data:

* Group data that is used together in the same shard, and avoid operations that need to access data that's held in multiple shards. Keep in mind that with Elastic Database, a shard is a SQL database in its own right, and Azure SQL Database does not support cross-database joins (which have to be performed on the client side). Remember also that in Azure SQL Database, referential integrity constraints, triggers, and stored procedures in one database cannot reference objects in another. Therefore, don't design a system that has dependencies between shards. A SQL database can, however, contain tables that hold copies of reference data frequently used by queries and other operations. These tables do not have to belong to any specific shardlet. Replicating this data across shards can help remove the need to join data that spans databases. Ideally, such data should be static or slow-moving to minimize the replication effort and reduce the chances of it becoming stale.

  > [!NOTE]
  > Although SQL Database does not support cross-database joins, you can perform cross-shard queries with the Elastic Database API. These queries can transparently iterate through the data held in all the shardlets that are referenced by a shard map. The Elastic Database API breaks cross-shard queries down into a series of individual queries (one for each database) and then merges the results. For more information, see the page [Multi-shard querying] on the Microsoft website.
  >
  >
* The data stored in shardlets that belong to the same shard map should have the same schema. For example, don't create a list shard map that points to some shardlets containing tenant data and other shardlets containing product information. This rule is not enforced by Elastic Database, but data management and querying becomes very complex if each shardlet has a different schema. In the example just cited, a good solution is to create two list shard maps: one that references tenant data and another that points to product information. Remember that the data belonging to different shardlets can be stored in the same shard.

  > [!NOTE]
  > The cross-shard query functionality of the Elastic Database API depends on each shardlet in the shard map containing the same schema.
  >
  >
* Transactional operations are only supported for data that's held within the same shard, and not across shards. Transactions can span shardlets as long as they are part of the same shard. Therefore, if your business logic needs to perform transactions, either store the affected data in the same shard or implement eventual consistency. For more information, see the [Data consistency primer].
* Place shards close to the users that access the data in those shards (in other words, geo-locate the shards). This strategy helps reduce latency.
* Avoid having a mixture of highly active (hotspots) and relatively inactive shards. Try to spread the load evenly across shards. This might require hashing the shardlet keys.
* If you are geo-locating shards, make sure that the hashed keys map to shardlets held in shards stored close to the users that access that data.
* Currently, only a limited set of SQL data types are supported as shardlet keys; *int, bigint, varbinary,* and *uniqueidentifier*. The SQL *int* and *bigint* types correspond to the *int* and *long* data types in C#, and have the same ranges. The SQL *varbinary* type can be handled by using a *Byte* array in C#, and the SQL *uniqueidentier* type corresponds to the *Guid* class in the .NET Framework.

As the name implies, Elastic Database makes it possible for a system to add and remove shards as the volume of data shrinks and grows. The APIs in the Azure SQL Database Elastic Database client library enable an application to create and delete shards dynamically (and transparently update the shard map manager). However, removing a shard is a destructive operation that also requires deleting all the data in that shard.

If an application needs to split a shard into two separate shards or combine shards, Elastic Database provides a separate split-merge service. This service runs in a cloud-hosted service (which must be created by the developer) and migrates data safely between shards. For more information, see the topic [Scaling using the Elastic Database split-merge tool] on the Microsoft website.

## Partitioning strategies for Azure Storage
Azure storage provides four abstractions for managing data:

* Blob Storage stores unstructured object data. A blob can be any type of text or binary data, such as a document, media file, or application installer. Blob storage is also referred to as Object storage.
* Table Storage stores structured datasets. Table storage is a NoSQL key-attribute data store, which allows for rapid development and fast access to large quantities of data.
* Queue Storage provides reliable messaging for workflow processing and for communication between components of cloud services.
* File Storage offers shared storage for legacy applications using the standard SMB protocol. Azure virtual machines and cloud services can share file data across application components via mounted shares, and on-premises applications can access file data in a share via the File service REST API.

Table storage and blob storage are essentially key-value stores that are optimized to hold structured and unstructured data respectively. Storage queues provide a mechanism for building loosely coupled, scalable applications. Table storage, file storage, blob storage, and storage queues are created within the context of an Azure storage account. Storage accounts support three forms of redundancy:

* **Locally redundant storage**, which maintains three copies of data within a single datacenter. This form of redundancy protects against hardware failure but not against a disaster that encompasses the entire datacenter.
* **Zone-redundant storage**, which maintains three copies of data spread across different datacenters within the same region (or across two geographically close regions). This form of redundancy can protect against disasters that occur within a single datacenter, but cannot protect against large-scale network disconnects that affect an entire region. Note that zone-redundant storage is currently only available for block blobs.
* **Geo-redundant storage**, which maintains six copies of data: three copies in one region (your local region), and another three copies in a remote region. This form of redundancy provides the highest level of disaster protection.

Microsoft has published scalability targets for Azure Storage. For more information, see the page [Azure Storage scalability and performance targets] on the Microsoft website. Currently, the total storage account capacity cannot exceed 500 TB. (This includes the size of data that's held in table storage, file storage and blob storage, as well as outstanding messages that are held in storage queue).

The maximum request rate for a storage account (assuming a 1-KB entity, blob, or message size) is 20,000 requests per second. A storage account has a maximum of 1000 IOPS (8 KB in size) per file share. If your system is likely to exceed these limits, consider partitioning the load across multiple storage accounts. A single Azure subscription can create up to 200 storage accounts. However, note that these limits might change over time.

## Partitioning Azure table storage
Azure table storage is a key-value store that's designed around partitioning. All entities are stored in a partition, and partitions are managed internally by Azure table storage. Each entity that's stored in a table must provide a two-part key that includes:

* **The partition key**. This is a string value that determines in which partition Azure table storage will place the entity. All entities with the same partition key will be stored in the same partition.
* **The row key**. This is another string value that identifies the entity within the partition. All entities within a partition are sorted lexically, in ascending order, by this key. The partition key/row key combination must be unique for each entity and cannot exceed 1 KB in length.

The remainder of the data for an entity consists of application-defined fields. No particular schemas are enforced, and each row can contain a different set of application-defined fields. The only limitation is that the maximum size of an entity (including the partition and row keys) is currently 1 MB. The maximum size of a table is 200 TB, although these figures might change in the future. (Check the page [Azure Storage scalability and performance targets] on the Microsoft website for the most recent information about these limits.)

If you are attempting to store entities that exceed this capacity, then consider splitting them into multiple tables. Use vertical partitioning to divide the fields into the groups that are most likely to be accessed together.

Figure 7 shows the logical structure of an example storage account (Contoso Data) for a fictitious e-commerce application. The storage account contains three tables: Customer Info, Product Info, and Order Info. Each table has multiple partitions.

In the Customer Info table, the data is partitioned according to the city in which the customer is located, and the row key contains the customer ID. In the Product Info table, the products are partitioned by product category, and the row key contains the product number. In the Order Info table, the orders are partitioned by the date on which they were placed, and the row key specifies the time the order was received. Note that all data is ordered by the row key in each partition.

![The tables and partitions in an example storage account](./images/data-partitioning/TableStorage.png)

*Figure 7. The tables and partitions in an example storage account*

> [!NOTE]
> Azure table storage also adds a timestamp field to each entity. The timestamp field is maintained by table storage and is updated each time the entity is modified and written back to a partition. The table storage service uses this field to implement optimistic concurrency. (Each time an application writes an entity back to table storage, the table storage service compares the value of the timestamp in the entity that's being written with the value that's held in table storage. If the values are different, it means that another application must have modified the entity since it was last retrieved, and the write operation fails. Don't modify this field in your own code, and don't specify a value for this field when you create a new entity.
>
>

Azure table storage uses the partition key to determine how to store the data. If an entity is added to a table with a previously unused partition key, Azure table storage creates a new partition for this entity. Other entities with the same partition key will be stored in the same partition.

This mechanism effectively implements an automatic scale-out strategy. Each partition is stored on a single server in an Azure datacenter to help ensure that queries that retrieve data from a single partition run quickly. However, different partitions can be distributed across multiple servers. Additionally, a single server can host multiple partitions if these partitions are limited in size.

Consider the following points when you design your entities for Azure table storage:

* The selection of partition key and row key values should be driven by the way in which the data is accessed. Choose a partition key/row key combination that supports the majority of your queries. The most efficient queries retrieve data by specifying the partition key and the row key. Queries that specify a partition key and a range of row keys can be completed by scanning a single partition. This is relatively fast because the data is held in row key order. If queries don't specify which partition to scan, the partition key might require Azure table storage to scan every partition for your data.

  > [!TIP]
  > If an entity has one natural key, then use it as the partition key and specify an empty string as the row key. If an entity has a composite key comprising two properties, select the slowest changing property as the partition key and the other as the row key. If an entity has more than two key properties, use a concatenation of properties to provide the partition and row keys.
  >
  >
* If you regularly perform queries that look up data by using fields other than the partition and row keys, consider implementing the [index table pattern].
* If you generate partition keys by using a monotonic increasing or decreasing sequence (such as "0001", "0002", "0003", and so on) and each partition only contains a limited amount of data, then Azure table storage can physically group these partitions together on the same server. This mechanism assumes that the application is most likely to perform queries across a contiguous range of partitions (range queries) and is optimized for this case. However, this approach can lead to hotspots focused on a single server because all insertions of new entities are likely to be concentrated at one end or the other of the contiguous ranges. It can also reduce scalability. To spread the load more evenly across servers, consider hashing the partition key to make the sequence more random.
* Azure table storage supports transactional operations for entities that belong to the same partition. This means that an application can perform multiple insert, update, delete, replace, or merge operations as an atomic unit (as long as the transaction doesn't include more than 100 entities and the payload of the request doesn't exceed 4 MB). Operations that span multiple partitions are not transactional, and might require you to implement eventual consistency as described by the [Data consistency primer]. For more information about table storage and transactions, go to the page [Performing entity group transactions] on the Microsoft website.
* Give careful attention to the granularity of the partition key because of the following reasons:
  * Using the same partition key for every entity causes the table storage service to create a single large partition that's held on one server. This prevents it from scaling out and instead focuses the load on a single server. As a result, this approach is only suitable for systems that manage a small number of entities. However, this approach does ensure that all entities can participate in entity group transactions.
  * Using a unique partition key for every entity causes the table storage service to create a separate partition for each entity, possibly resulting in a large number of small partitions (depending on the size of the entities). This approach is more scalable than using a single partition key, but entity group transactions are not possible. Also, queries that fetch more than one entity might involve reading from more than one server. However, if the application performs range queries, then using a monotonic sequence to generate the partition keys might help to optimize these queries.
  * Sharing the partition key across a subset of entities makes it possible for you to group related entities in the same partition. Operations that involve related entities can be performed by using entity group transactions, and queries that fetch a set of related entities can be satisfied by accessing a single server.

For additional information about partitioning data in Azure table storage, see the article [Azure storage table design guide] on the Microsoft website.

## Partitioning Azure blob storage
Azure blob storage makes it possible to hold large binary objects--currently up to 5 TB in size for block blobs or 1 TB for page blobs. (For the most recent information, go to the page [Azure Storage scalability and performance targets] on the Microsoft website.) Use block blobs in scenarios such as streaming where you need to upload or download large volumes of data quickly. Use page blobs for applications that require random rather than serial access to parts of the data.

Each blob (either block or page) is held in a container in an Azure storage account. You can use containers to group related blobs that have the same security requirements. This grouping is logical rather than physical. Inside a container, each blob has a unique name.

The partition key for a blob is account name + container name + blob name. This means each blob can have its own partition if load on the blob demands it. Blobs can be distributed across many servers in order to scale out access to them, but a single blob can only be served by a single server. 

The actions of writing a single block (block blob) or page (page blob) are atomic, but operations that span blocks, pages, or blobs are not. If you need to ensure consistency when performing write operations across blocks, pages, and blobs, take out a write lock by using a blob lease.

Azure blob storage targets transfer rates of up to 60 MB per second or 500 requests per second for each blob. If you anticipate surpassing these limits, and the blob data is relatively static, then consider replicating blobs by using the Azure Content Delivery Network. For more information, see the page [Azure Content Delivery Network] on the Microsoft website. For additional guidance and considerations, see [Using Azure Content Delivery Network].

## Partitioning Azure storage queues
Azure storage queues enable you to implement asynchronous messaging between processes. An Azure storage account can contain any number of queues, and each queue can contain any number of messages. The only limitation is the space that's available in the storage account. The maximum size of an individual message is 64 KB. If you require messages bigger than this, then consider using Azure Service Bus queues instead.

Each storage queue has a unique name within the storage account that contains it. Azure partitions queues based on the name. All messages for the same queue are stored in the same partition, which is controlled by a single server. Different queues can be managed by different servers to help balance the load. The allocation of queues to servers is transparent to applications and users.

 In a large-scale application, don't use the same storage queue for all instances of the application because this approach might cause the server that's hosting the queue to become a hotspot. Instead, use different queues for different functional areas of the application. Azure storage queues do not support transactions, so directing messages to different queues should have little impact on messaging consistency.

An Azure storage queue can handle up to 2,000 messages per second.  If you need to process messages at a greater rate than this, consider creating multiple queues. For example, in a global application, create separate storage queues in separate storage accounts to handle application instances that are running in each region.

## Partitioning strategies for Azure Service Bus
Azure Service Bus uses a message broker to handle messages that are sent to a Service Bus queue or topic. By default, all messages that are sent to a queue or topic are handled by the same message broker process. This architecture can place a limitation on the overall throughput of the message queue. However, you can also partition a queue or topic when it is created. You do this by setting the *EnablePartitioning* property of the queue or topic description to *true*.

A partitioned queue or topic is divided into multiple fragments, each of which is backed by a separate message store and message broker. Service Bus takes responsibility for creating and managing these fragments. When an application posts a message to a partitioned queue or topic, Service Bus assigns the message to a fragment for that queue or topic. When an application receives a message from a queue or subscription, Service Bus checks each fragment for the next available message and then passes it to the application for processing.

This structure helps distribute the load across message brokers and message stores, increasing scalability and improving availability. If the message broker or message store for one fragment is temporarily unavailable, Service Bus can retrieve messages from one of the remaining available fragments.

Service Bus assigns a message to a fragment as follows:

* If the message belongs to a session, all messages with the same value for the * SessionId*  property are sent to the same fragment.
* If the message does not belong to a session, but the sender has specified a value for the *PartitionKey* property, then all messages with the same *PartitionKey* value are sent to the same fragment.

  > [!NOTE]
  > If the *SessionId* and *PartitionKey* properties are both specified, then they must be set to the same value or the message will be rejected.
  >
  >
* If the *SessionId* and *PartitionKey* properties for a message are not specified, but duplicate detection is enabled, the *MessageId* property will be used. All messages with the same *MessageId* will be directed to the same fragment.
* If messages do not include a *SessionId, PartitionKey,* or *MessageId* property, then Service Bus assigns messages to fragments sequentially. If a fragment is unavailable, Service Bus will move on to the next. This means that a temporary fault in the messaging infrastructure does not cause the message-send operation to fail.

Consider the following points when deciding if or how to partition a Service Bus message queue or topic:

* Service Bus queues and topics are created within the scope of a Service Bus namespace. Service Bus currently allows up to 100 partitioned queues or topics per namespace.
* Each Service Bus namespace imposes quotas on the available resources, such as the number of subscriptions per topic, the number of concurrent send and receive requests per second, and the maximum number of concurrent connections that can be established. These quotas are documented on the Microsoft website on the page [Service Bus quotas]. If you expect to exceed these values, then create additional namespaces with their own queues and topics, and spread the work across these namespaces. For example, in a global application, create separate namespaces in each region and configure application instances to use the queues and topics in the nearest namespace.
* Messages that are sent as part of a transaction must specify a partition key. This can be a *SessionId*, *PartitionKey*, or *MessageId* property. All messages that are sent as part of the same transaction must specify the same partition key because they must be handled by the same message broker process. You cannot send messages to different queues or topics within the same transaction.
* Partitioned queues and topics can't be configured to be automatically deleted when they become idle.
* Partitioned queues and topics can't currently be used with the Advanced Message Queuing Protocol (AMQP) if you are building cross-platform or hybrid solutions.

## Partitioning strategies for Cosmos DB

Azure Cosmos DB is a NoSQL database that can store JSON documents using the [Azure Cosmos DB SQL API][cosmosdb-sql-api]. A document in a Cosmos DB database is a JSON-serialized representation of an object or other piece of data. No fixed schemas are enforced except that every document must contain a unique ID.

Documents are organized into collections. You can group related documents together in a collection. For example, in a system that maintains blog postings, you can store the contents of each blog post as a document in a collection. You can also create collections for each subject type. Alternatively, in a multitenant application, such as a system where different authors control and manage their own blog posts, you can partition blogs by author and create separate collections for each author. The storage space that's allocated to collections is elastic and can shrink or grow as needed.

Cosmos DB supports automatic partitioning of data based on an application-defined partition key. A *logical partition* is a partition that stores all the data for a single partition key value. All documents that share the same value for the partition key are placed within the same logical partition. Cosmos DB distributes values according to hash of the partition key. A logical partition has a maximum size of 10 GB. Therefore, the choice of the partition key is an important decision at design time. Choose a property with a wide range of values and even access patterns. For more information, see [Partition and scale in Azure Cosmos DB](/azure/cosmos-db/partition-data).

> [!NOTE]
> Each Cosmos DB database has a *performance level* that determines the amount of resources it gets. A performance level is associated with a *request unit* (RU) rate limit. The RU rate limit specifies the volume of resources that's reserved and available for exclusive use by that collection. The cost of a collection depends on the performance level that's selected for that collection. The higher the performance level (and RU rate limit) the higher the charge. You can adjust the performance level of a collection by using the Azure portal. For more information, see [Request Units in Azure Cosmos DB][cosmos-db-ru].
>
>

If the paritioning mechanism that Cosmos DB provides is not sufficient, you may need to shard the data at the application level. Document collections provide a natural mechanism for partitioning data within a single database. The simplest way to implement sharding is to create a collection for each shard. Containers are logical resources and can span one or more servers. Fixed-size containers have a maximum limit of 10 GB and 10,000 RU/s throughput. Unlimited containers do not have a maximum sotrage size, but must specify a partition key. With application sharding, the client application must direct requests to the appropriate shard, usually by implementing its own mapping mechanism based on some attributes of the data that define the shard key. 

All databases are created in the context of a Cosmos DB database account. A single account can contain several databases, and it specifies in which regions the databases are created. Each account also enforces its own access control. You can use Cosmos DB accounts to geo-locate shards (collections within databases) close to the users who need to access them, and enforce restrictions so that only those users can connect to them.

Consider the following points when deciding how to partition data with the Cosmos DB SQL API:

* **The resources available to a Cosmos DB database are subject to the quota limitations of the account**. Each database can hold a number of collections, and each collection is associated with a performance level that governs the RU rate limit (reserved throughput) for that collection. For more information, see [Azure subscription and service limits, quotas, and constraints][azure-limits].
* **Each document must have an attribute that can be used to uniquely identify that document within the collection in which it is held**. This attribute is different from the shard key, which defines which collection holds the document. A collection can contain a large number of documents. In theory, it's limited only by the maximum length of the document ID. The document ID can be up to 255 characters.
* **All operations against a document are performed within the context of a transaction. Transactions are scoped to the collection in which the document is contained.** If an operation fails, the work that it has performed is rolled back. While a document is subject to an operation, any changes that are made are subject to snapshot-level isolation. This mechanism guarantees that if, for example, a request to create a new document fails, another user who's querying the database simultaneously will not see a partial document that is then removed.
* **Database queries are also scoped to the collection level**. A single query can retrieve data from only one collection. If you need to retrieve data from multiple collections, you must query each collection individually and merge the results in your application code.
* **Cosmos DB supports programmable items that can all be stored in a collection alongside documents**. These include stored procedures, user-defined functions, and triggers (written in JavaScript). These items can access any document within the same collection. Furthermore, these items run either inside the scope of the ambient transaction (in the case of a trigger that fires as the result of a create, delete, or replace operation performed against a document), or by starting a new transaction (in the case of a stored procedure that is run as the result of an explicit client request). If the code in a programmable item throws an exception, the transaction is rolled back. You can use stored procedures and triggers to maintain integrity and consistency between documents, but these documents must all be part of the same collection.
* **The collections that you intend to hold in the databases should be unlikely to exceed the throughput limits defined by the performance levels of the collections**. For more information, see [Request Units in Azure Cosmos DB][cosmos-db-ru]. If you anticipate reaching these limits, consider splitting collections across databases in different accounts to reduce the load per collection.

## Partitioning strategies for Azure Search
The ability to search for data is often the primary method of navigation and exploration that's provided by many web applications. It helps users find resources quickly (for example, products in an e-commerce application) based on combinations of search criteria. The Azure Search service provides full-text search capabilities over web content, and includes features such as type-ahead, suggested queries based on near matches, and faceted navigation. A full description of these capabilities is available on the page [What is Azure Search?] on the Microsoft website.

Azure Search stores searchable content as JSON documents in a database. You define indexes that specify the searchable fields in these documents and provide these definitions to Azure Search. When a user submits a search request, Azure Search uses the appropriate indexes to find matching items.

To reduce contention, the storage that's used by Azure Search can be divided into 1, 2, 3, 4, 6, or 12 partitions, and each partition can be replicated up to 6 times. The product of the number of partitions multiplied by the number of replicas is called the *search unit* (SU). A single instance of Azure Search can contain a maximum of 36 SUs (a database with 12 partitions only supports a maximum of 3 replicas).

You are billed for each SU that is allocated to your service. As the volume of searchable content increases or the rate of search requests grows, you can add SUs to an existing instance of Azure Search to handle the extra load. Azure Search itself distributes the documents evenly across the partitions. No manual partitioning strategies are currently supported.

Each partition can contain a maximum of 15 million documents or occupy 300 GB of storage space (whichever is smaller). You can create up to 50 indexes. The performance of the service varies and depends on the complexity of the documents, the available indexes, and the effects of network latency. On average, a single replica (1 SU) should be able to handle 15 queries per second (QPS), although we recommend performing benchmarking with your own data to obtain a more precise measure of throughput. For more information, see the page [Service limits in Azure Search] on the Microsoft website.

> [!NOTE]
> You can store a limited set of data types in searchable documents, including strings, Booleans, numeric data, datetime data, and some geographical data. For more details, see the page [Supported data types (Azure Search)] on the Microsoft website.
>
>

You have limited control over how Azure Search partitions data for each instance of the service. However, in a global environment you might be able to improve performance and reduce latency and contention further by partitioning the service itself using either of the following strategies:

* Create an instance of Azure Search in each geographic region, and ensure that client applications are directed towards the nearest available instance. This strategy requires that any updates to searchable content are replicated in a timely manner across all instances of the service.
* Create two tiers of Azure Search:

  * A local service in each region that contains the data that's most frequently accessed by users in that region. Users can direct requests here for fast but limited results.
  * A global service that encompasses all the data. Users can direct requests here for slower but more complete results.

This approach is most suitable when there is a significant regional variation in the data that's being searched.

## Partitioning strategies for Azure Redis Cache
Azure Redis Cache provides a shared caching service in the cloud that's based on the Redis key-value data store. As its name implies, Azure Redis Cache is intended as a caching solution. Use it only for holding transient data and not as a permanent data store. Applications that utilize Azure Redis Cache should be able to continue functioning if the cache is unavailable. Azure Redis Cache supports primary/secondary replication to provide high availability, but currently limits the maximum cache size to 53 GB. If you need more space than this, you must create additional caches. For more information, go to the page [Azure Redis Cache] on the Microsoft website.

Partitioning a Redis data store involves splitting the data across instances of the Redis service. Each instance constitutes a single partition. Azure Redis Cache abstracts the Redis services behind a façade and does not expose them directly. The simplest way to implement partitioning is to create multiple Azure Redis Cache instances and spread the data across them.

You can associate each data item with an identifier (a partition key) that specifies which cache stores the data item. The client application logic can then use this identifier to route requests to the appropriate partition. This scheme is very simple, but if the partitioning scheme changes (for example, if additional Azure Redis Cache instances are created), client applications might need to be reconfigured.

Native Redis (not Azure Redis Cache) supports server-side partitioning based on Redis clustering. In this approach, you can divide the data evenly across servers by using a hashing mechanism. Each Redis server stores metadata that describes the range of hash keys that the partition holds, and also contains information about which hash keys are located in the partitions on other servers.

Client applications simply send requests to any of the participating Redis servers (probably the closest one). The Redis server examines the client request. If it can be resolved locally, it performs the requested operation. Otherwise it forwards the request on to the appropriate server.

This model is implemented by using Redis clustering, and is described in more detail on the [Redis cluster tutorial] page on the Redis website. Redis clustering is transparent to client applications. Additional Redis servers can be added to the cluster (and the data can be re-partitioned) without requiring that you reconfigure the clients.

> [!IMPORTANT]
> Azure Redis Cache does not currently support Redis clustering. If you want to implement this approach with Azure, then you must implement your own Redis servers by installing Redis on a set of Azure virtual machines and configuring them manually. The page [Running Redis on a CentOS Linux VM in Azure] on the Microsoft website walks through an example that shows you how to build and configure a Redis node running as an Azure VM.
>
>

The page [Partitioning: how to split data among multiple Redis instances] on the Redis website provides more information about implementing partitioning with Redis. The remainder of this section assumes that you are implementing client-side or proxy-assisted partitioning.

Consider the following points when deciding how to partition data with Azure Redis Cache:

* Azure Redis Cache is not intended to act as a permanent data store, so whatever partitioning scheme you implement, your application code must be able to retrieve data from a location that's not the cache.
* Data that is frequently accessed together should be kept in the same partition. Redis is a powerful key-value store that provides several highly optimized mechanisms for structuring data. These mechanisms can be one of the following:

  * Simple strings (binary data up to 512 MB in length)
  * Aggregate types such as lists (which can act as queues and stacks)
  * Sets (ordered and unordered)
  * Hashes (which can group related fields together, such as the items that represent the fields in an object)
* The aggregate types enable you to associate many related values with the same key. A Redis key identifies a list, set, or hash rather than the data items that it contains. These types are all available with Azure Redis Cache and are described by the [Data types] page on the Redis website. For example, in part of an e-commerce system that tracks the orders that are placed by customers, the details of each customer can be stored in a Redis hash that is keyed by using the customer ID. Each hash can hold a collection of order IDs for the customer. A separate Redis set can hold the orders, again structured as hashes, and keyed by using the order ID. Figure 8 shows this structure. Note that Redis does not implement any form of referential integrity, so it is the developer's responsibility to maintain the relationships between customers and orders.

![Suggested structure in Redis storage for recording customer orders and their details](./images/data-partitioning/RedisCustomersandOrders.png)

*Figure 8. Suggested structure in Redis storage for recording customer orders and their details*

> [!NOTE]
> In Redis, all keys are binary data values (like Redis strings) and can contain up to 512 MB of data. In theory, a key can contain almost any information. However, we recommend adopting a consistent naming convention for keys that is descriptive of the type of data and that identifies the entity, but is not excessively long. A common approach is to use keys of the form "entity_type:ID". For example, you can use "customer:99" to indicate the key for a customer with the ID 99.
>
>

* You can implement vertical partitioning by storing related information in different aggregations in the same database. For example, in an e-commerce application, you can store commonly accessed information about products in one Redis hash and less frequently used detailed information in another.
  Both hashes can use the same product ID as part of the key. For example, you can use "product: *nn*" (where *nn* is the product ID) for the product information and "product_details: *nn*" for the detailed data. This strategy can help reduce the volume of data that most queries are likely to retrieve.
* You can repartition a Redis data store, but keep in mind that it's a complex and time-consuming task. Redis clustering can repartition data automatically, but this capability is not available with Azure Redis Cache. Therefore, when you design your partitioning scheme, try to leave sufficient free space in each partition to allow for expected data growth over time. However, remember that Azure Redis Cache is intended to cache data temporarily, and that data held in the cache can have a limited lifetime specified as a time-to-live (TTL) value. For relatively volatile data, the TTL can be short, but for static data the TTL can be a lot longer. Avoid storing large amounts of long-lived data in the cache if the volume of this data is likely to fill the cache. You can specify an eviction policy that causes Azure Redis Cache to remove data if space is at a premium.

  > [!NOTE]
  > When you use Azure Redis cache, you specify the maximum size of the cache (from 250 MB to 53 GB) by selecting the appropriate pricing tier. However, after an Azure Redis Cache has been created, you cannot increase (or decrease) its size.
  >
  >
* Redis batches and transactions cannot span multiple connections, so all data that is affected by a batch or transaction should be held in the same database (shard).

  > [!NOTE]
  > A sequence of operations in a Redis transaction is not necessarily atomic. The commands that compose a transaction are verified and queued before they run. If an error occurs during this phase, the entire queue is discarded. However, after the transaction has been successfully submitted, the queued commands run in sequence. If any command fails, only that command stops running. All previous and subsequent commands in the queue are performed. For more information, go to the [Transactions] page on the Redis website.
  >
  >
* Redis supports a limited number of atomic operations. The only operations of this type that support multiple keys and values are MGET and MSET operations. MGET operations return a collection of values for a specified list of keys, and MSET operations store a collection of values for a specified list of keys. If you need to use these operations, the key-value pairs that are referenced by the MSET and MGET commands must be stored within the same database.

## Partitioning Strategies for Azure Service Fabric
Azure Service Fabric is a microservices platform that provides a runtime for distributed applications in the cloud. Service Fabric supports .Net guest executables, stateful and stateless services, and containers. Stateful services provide a [reliable collection][service-fabric-reliable-collections] to persistently store data in a key-value collection within the Service Fabric cluster. For more information about strategies for partioning keys in a reliable collection, see [guidelines and recommendations for reliable collections in Azure Service Fabric].

### More information
* [Overview of Azure Service Fabric] is an introduction to Azure Service Fabric.
* [Partition Service Fabric reliable services] provides more information about reliable services in Azure Service Fabric.

## Partitioning strategies for Azure Event Hubs

[Azure Event Hubs][event-hubs] is designed for data streaming at massive scale, and partitioning is built into the service to enable horizontal scaling. Each consumer only reads a specific partition of the message stream. 

The event publisher is only aware of its partition key, not the partition to which the events are published. This decoupling of key and partition insulates the sender from needing to know too much about the downstream processing. (It's also possible send events directly to a given partition, but generally that's not recommended.)  

Consider long-term scale when you select the partition count. After an event hub is created, you can't change the number of partitions. 

For more information about using partitions in Event Hubs, see [What is Event Hubs?].

For considerations about trade-offs between availability and consistency, see [Availability and consistency in Event Hubs].

