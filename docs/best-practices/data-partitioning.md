---
title: Data partitioning guidance
description: Guidance for how to separate partitions to be managed and accessed separately.
author: dragon119
ms.date: 11/03/2018

---
# Data partitioning

In many large-scale solutions, data is divided into *partitions* that can be managed and accessed separately. Partitioning can improve scalability, reduce contention, and optimize performance. It can also provide a mechanism for dividing data by usage pattern. For example, you can archive older data in cheaper data storage.

However, the partitioning strategy must be chosen carefully to maximize the benefits while minimizing adverse effects.

> In this article, the term *partitioning* means the process of physically dividing data into separate data stores. It is not the same as SQL Server table partitioning.

## Why partition data?

* **Improve scalability**. When you scale up a single database system, it will eventually reach a physical hardware limit. If you divide data across multiple partitions, each of which is hosted on a separate server, you can scale out the system almost indefinitely.
* **Improve performance**. Data access operations on each partition take place over a smaller volume of data. Provided that the data is partitioned in a suitable way, partitioning can make your system more efficient. Operations that affect more than one partition can run in parallel. Each partition can be located near the application that uses it to minimize network latency.
* **Improve availability**. Separating data across multiple servers avoids a single point of failure. If a server fails, or is undergoing planned maintenance, only the data in that partition is unavailable. Operations on other partitions can continue. Increasing the number of partitions reduces the relative impact of a single server failure by reducing the percentage of data that will be unavailable. Replicating each partition can further reduce the chance of a single partition failure affecting operations. It also makes it possible to separate critical data that must be continually and highly available from low-value data that has lower availability requirements (log files, for example).
* **Improve security**. Depending on the nature of the data and how it is partitioned, it might be possible to separate sensitive and non-sensitive data into different partitions, and therefore into different servers or data stores. Security can then be specifically optimized for the sensitive data.
* **Provide operational flexibility**. Partitioning offers many opportunities for fine tuning operations, maximizing administrative efficiency, and minimizing cost. For example, you can define different strategies for management, monitoring, backup and restore, and other administrative tasks based on the importance of the data in each partition.
* **Match the data store to the pattern of use**. Partitioning allows each partition to be deployed on a different type of data store, based on cost and the built-in features that data store offers. For example, large binary data can be stored in a blob data store, while more structured data can be held in a document database. For more information, see [Building a polyglot solution] in the patterns & practices guide and [Data access for highly-scalable solutions: Using SQL, NoSQL, and polyglot persistence] on the Microsoft website.

Some systems do not implement partitioning because it is considered a cost rather than an advantage. Common reasons for this rationale include:

* Many data storage systems do not support joins across partitions, and it can be difficult to maintain referential integrity in a partitioned system. It is frequently necessary to implement joins and integrity checks in application code (in the partitioning layer), which can result in additional I/O and application complexity.
* Maintaining partitions is not always a trivial task. In a system where the data is volatile, you might need to rebalance partitions periodically to reduce contention and hot spots.
* Some common tools do not work naturally with partitioned data.

## Designing partitions
Data can be partitioned in different ways: horizontally, vertically, or functionally. The strategy you choose depends on the reason for partitioning the data, and the requirements of the applications and services that will use the data.

> [!NOTE]
> The partitioning schemes described in this guidance are explained in a way that is independent of the underlying data storage technology. They can be applied to many types of data stores, including relational and NoSQL databases.
>
>

### Partitioning strategies
The three typical strategies for partitioning data are:

* **Horizontal partitioning** (often called *sharding*). In this strategy, each partition is a data store in its own right, but all partitions have the same schema. Each partition is known as a *shard* and holds a specific subset of the data, such as all the orders for a specific set of customers in an e-commerce application.
* **Vertical partitioning**. In this strategy, each partition holds a subset of the fields for items in the data store. The fields are divided according to their pattern of use. For example, frequently accessed fields might be placed in one vertical partition and less frequently accessed fields in another.
* **Functional partitioning**. In this strategy, data is aggregated according to how it is used by each bounded context in the system. For example, an e-commerce system that implements separate business functions for invoicing and managing product inventory might store invoice data in one partition and product inventory data in another.

It’s important to note that the three strategies described here can be combined. They are not mutually exclusive, and we recommend that you consider them all when you design a partitioning scheme. For example, you might divide data into shards and then use vertical partitioning to further subdivide the data in each shard. Similarly, the data in a functional partition can be split into shards (which can also be vertically partitioned).

However, the differing requirements of each strategy can raise a number of conflicting issues. You must evaluate and balance all of these when designing a partitioning scheme that meets the overall data processing performance targets for your system. The following sections explore each of the strategies in more detail.

### Horizontal partitioning (sharding)
Figure 1 shows an overview of horizontal partitioning or sharding. In this example, product inventory data is divided into shards based on the product key. Each shard holds the data for a contiguous range of shard keys (A-G and H-Z), organized alphabetically.

![Horizontally partitioning (sharding) data based on a partition key](./images/data-partitioning/DataPartitioning01.png)

*Figure 1. Horizontally partitioning (sharding) data based on a partition key*

Sharding helps you spread the load over more computers, which reduces contention and improves performance. You can scale the system out by adding further shards that run on additional servers.

The most important factor when implementing this partitioning strategy is the choice of sharding key. It can be difficult to change the key after the system is in operation. The key must ensure that data is partitioned so that the workload is as even as possible across the shards.

Note that different shards do not have to contain similar volumes of data. Rather, the more important consideration is to balance the number of requests. Some shards might be very large, but each item is the subject of a low number of access operations. Other shards might be smaller, but each item is accessed much more frequently. It is also important to ensure that a single shard does not exceed the scale limits (in terms of capacity and processing resources) of the data store that's being used to host that shard.

If you use a sharding scheme, avoid creating hotspots (or hot partitions) that can affect performance and availability. For example, if you use a hash of a customer identifier instead of the first letter of a customer’s name, you prevent the unbalanced distribution that results from common and less common initial letters. This is a typical technique that helps distribute data more evenly across partitions.

Choose a sharding key that minimizes any future requirements to split large shards into smaller pieces, coalesce small shards into larger partitions, or change the schema that describes the data stored in a set of partitions. These operations can be very time consuming, and might require taking one or more shards offline while they are performed.

If shards are replicated, it might be possible to keep some of the replicas online while others are split, merged, or reconfigured. However, the system might need to limit the operations that can be performed on the data in these shards while the reconfiguration is taking place. For example, the data in the replicas can be marked as read-only to limit the scope of inconsistences that might occur while shards are being restructured.

> For more detailed information and guidance about many of these considerations, and good practice techniques for designing data stores that implement horizontal partitioning, see [Sharding pattern].
>
>

### Vertical partitioning
The most common use for vertical partitioning is to reduce the I/O and performance costs associated with fetching the items that are accessed most frequently. Figure 2 shows an example of vertical partitioning. In this example, different properties for each data item are held in different partitions. One partition holds data that is accessed more frequently, including the name, description, and price information for products. Another holds the volume in stock and the last ordered date.

![Vertically partitioning data by its pattern of use](./images/data-partitioning/DataPartitioning02.png)

*Figure 2. Vertically partitioning data by its pattern of use*

In this example, the application regularly queries the product name, description, and price when displaying the product details to customers. The stock level and date when the product was last ordered from the manufacturer are held in a separate partition because these two items are commonly used together.

This partitioning scheme has the added advantage that the relatively slow-moving data (product name, description, and price) is separated from the more dynamic data (stock level and last ordered date). An application might find it beneficial to cache the slow-moving data in memory if it is frequently accessed.

Another typical scenario for this partitioning strategy is to maximize the security of sensitive data. For example, you can do this by storing credit card numbers and the corresponding card security verification numbers in separate partitions.

Vertical partitioning can also reduce the amount of concurrent access that's needed to the data.

> Vertical partitioning operates at the entity level within a data store, partially normalizing an entity to break it down from a *wide* item to a set of *narrow* items. It is ideally suited for column-oriented data stores such as HBase and Cassandra. If the data in a collection of columns is unlikely to change, you can also consider using column stores in SQL Server.
>
>

### Functional partitioning
For systems where it is possible to identify a bounded context for each distinct business area or service in the application, functional partitioning provides a technique for improving isolation and data access performance. Another common use of functional partitioning is to separate read-write data from read-only data that's used for reporting purposes. Figure 3 shows an overview of functional partitioning where inventory data is separated from customer data.

![Functionally partitioning data by bounded context or subdomain](./images/data-partitioning/DataPartitioning03.png)

*Figure 3. Functionally partitioning data by bounded context or subdomain*

This partitioning strategy can help reduce data access contention across different parts of a system.

## Designing partitions for scalability
It's vital to consider size and workload for each partition and balance them so that data is distributed to achieve maximum scalability. However, you must also partition the data so that it does not exceed the scaling limits of a single partition store.

Follow these steps when designing partitions for scalability:

1. Analyze the application to understand the data access patterns, such as the size of the result set returned by each query, the frequency of access, the inherent latency, and the server-side compute processing requirements. In many cases, a few major entities will demand most of the processing resources.
2. Use this analysis to determine the current and future scalability targets, such as data size and workload. Then distribute the data across the partitions to meet the scalability target. In the horizontal partitioning strategy, choosing the appropriate shard key is important to make sure distribution is even. For more information, see the [Sharding pattern].
3. Make sure that the resources available to each partition are sufficient to handle the scalability requirements in terms of data size and throughput. For example, the node that's hosting a partition might impose a hard limit on the amount of storage space, processing power, or network bandwidth that it provides. If the data storage and processing requirements are likely to exceed these limits, it might be necessary to refine your partitioning strategy or split data out further. For example, one scalability approach might be to separate logging data from the core application features. You do this by using separate data stores to prevent the total data storage requirements from exceeding the scaling limit of the node. If the total number of data stores exceeds the node limit, it might be necessary to use separate storage nodes.
4. Monitor the system under use to verify that the data is distributed as expected and that the partitions can handle the load that is imposed on them. It's possible that the usage does not match the usage that's anticipated by the analysis. In that case, it might be possible to rebalance the partitions. Failing that, it might be necessary to redesign some parts of the system to gain the required balance.

Note that some cloud environments allocate resources in terms of infrastructure boundaries. Ensure that the limits of your selected boundary provide enough room for any anticipated growth in the volume of data, in terms of data storage, processing power, and bandwidth.

For example, if you use Azure table storage, a busy shard might require more resources than are available to a single partition to handle requests. (There is a limit to the volume of requests that can be handled by a single partition in a particular period of time. See the page [Azure storage scalability and performance targets] on the Microsoft website for more details.)

 If this is the case, the shard might need to be repartitioned to spread the load. If the total size or throughput of these tables exceeds the capacity of a storage account, it might be necessary to create additional storage accounts and spread the tables across these accounts. If the number of storage accounts exceeds the number of accounts that are available to a subscription, then it might be necessary to use multiple subscriptions.

## Designing partitions for query performance
Query performance can often be boosted by using smaller data sets and by running parallel queries. Each partition should contain a small proportion of the entire data set. This reduction in volume can improve the performance of queries. However, partitioning is not an alternative for designing and configuring a database appropriately. For example, make sure that you have the necessary indexes in place if you are using a relational database.

Follow these steps when designing partitions for query performance:

1. Examine the application requirements and performance:
   * Use the business requirements to determine the critical queries that must always perform quickly.
   * Monitor the system to identify any queries that perform slowly.
   * Establish which queries are performed most frequently. A single instance of each query might have minimal cost, but the cumulative consumption of resources could be significant. It might be beneficial to separate the data that's retrieved by these queries into a distinct partition, or even a cache.
2. Partition the data that is causing slow performance:
   * Limit the size of each partition so that the query response time is within target.
   * Design the shard key so that the application can easily find the partition if you are implementing horizontal partitioning. This prevents the query from having to scan through every partition.
   * Consider the location of a partition. If possible, try to keep data in partitions that are geographically close to the applications and users that access it.
3. If an entity has throughput and query performance requirements, use functional partitioning based on that entity. If this still doesn't satisfy the requirements, apply horizontal partitioning as well. In most cases a single partitioning strategy will suffice, but in some cases it is more efficient to combine both strategies.
4. Consider using asynchronous queries that run in parallel across partitions to improve performance.

## Designing partitions for availability
Partitioning data can improve the availability of applications by ensuring that the entire dataset does not constitute a single point of failure and that individual subsets of the dataset can be managed independently. Replicating partitions that contain critical data can also improve availability.

When designing and implementing partitions, consider the following factors that affect availability:

* **How critical the data is to business operations**. Some data might include critical business information such as invoice details or bank transactions. Other data might include less critical operational data, such as log files, performance traces, and so on. After identifying each type of data, consider:
  * Storing critical data in highly-available partitions with an appropriate backup plan.
  * Establishing separate management and monitoring mechanisms or procedures for the different criticalities of each dataset. Place data that has the same level of criticality in the same partition so that it can be backed up together at an appropriate frequency. For example, partitions that hold data for bank transactions might need to be backed up more frequently than partitions that hold logging or trace information.
* **How individual partitions can be managed**. Designing partitions to support independent management and maintenance provides several advantages. For example:
  * If a partition fails, it can be recovered independently without affecting instances of applications that access data in other partitions.
  * Partitioning data by geographical area allows scheduled maintenance tasks to occur at off-peak hours for each location. Ensure that partitions are not too big to prevent any planned maintenance from being completed during this period.
* **Whether to replicate critical data across partitions**. This strategy can improve availability and performance, although it can also introduce consistency issues. It takes time for changes made to data in a partition to be synchronized with every replica. During this period, different partitions will contain different data values.

## Understanding how partitioning affects design and development
Using partitioning adds complexity to the design and development of your system. Consider partitioning as a fundamental part of system design even if the system initially only contains a single partition. If you address partitioning as an afterthought, when the system starts to suffer performance and scalability issues, the complexity increases because you already have a live system to maintain.

If you update the system to incorporate partitioning in this environment, it necessitates modifying the data access logic. It can also involve migrating large quantities of existing data to distribute it across partitions, often while users expect to be able to continue using the system.

In some cases, partitioning is not considered important because the initial dataset is small and can be easily handled by a single server. This might be true in a system that is not expected to scale beyond its initial size, but many commercial systems need to expand as the number of users increases. This expansion is typically accompanied by a growth in the volume of data.

It's also important to understand that partitioning is not always a function of large data stores. For example, a small data store might be heavily accessed by hundreds of concurrent clients. Partitioning the data in this situation can help to reduce contention and improve throughput.

Consider the following points when you design a data partitioning scheme:

* **Where possible, keep data for the most common database operations together in each partition to minimize cross-partition data access operations**. Querying across partitions can be more time-consuming than querying only within a single partition, but optimizing partitions for one set of queries might adversely affect other sets of queries. When you can't avoid querying across partitions, minimize query time by running parallel queries and aggregating the results within the application. This approach might not be possible in some cases, such as when it's necessary to obtain a result from one query and use it in the next query.
* **If queries make use of relatively static reference data, such as postal code tables or product lists, consider replicating this data in all of the partitions to reduce the requirement for separate lookup operations in different partitions**. This approach can also reduce the likelihood of the reference data becoming a "hot" dataset that is subject to heavy traffic from across the entire system. However,   there is an additional cost associated with synchronizing any changes that might occur to this reference data.
* **Where possible, minimize requirements for referential integrity across vertical and functional partitions**. In these schemes, the application itself is responsible for maintaining referential integrity across partitions when data is updated and consumed. Queries that must join data across multiple partitions run more slowly than queries that join data only within the same partition because the application typically needs to perform consecutive queries based on a key and then on a foreign key. Instead, consider replicating or de-normalizing the relevant data. To minimize the query time where cross-partition joins are necessary, run parallel queries over the partitions and join the data within the application.
* **Consider the effect that the partitioning scheme might have on the data consistency across partitions.** Evaluate whether strong consistency is actually a requirement. Instead, a common approach in the cloud is to implement eventual consistency. The data in each partition is updated separately, and the application logic ensures that the updates are all completed successfully. It also handles the inconsistencies that can arise from querying data while an eventually consistent operation is running. For more information about implementing eventual consistency, see the [Data consistency primer].
* **Consider how queries locate the correct partition**. If a query must scan all partitions to locate the required data, there is a significant impact on performance, even when multiple parallel queries are running. Queries that are used with vertical and functional partitioning strategies can naturally specify the partitions. However, horizontal partitioning (sharding) can make locating an item difficult because every shard has the same schema. A typical solution for sharding is to maintain a map that can be used to look up the shard location for specific items of data. This map can be implemented in the sharding logic of the application, or maintained by the data store if it supports transparent sharding.
* **When using a horizontal partitioning strategy, consider periodically rebalancing the shards**. This helps distribute the data evenly by size and by workload to minimize hotspots, maximize query performance, and work around physical storage limitations. However, this is a complex task that often requires the use of a custom tool or process.
* **If you replicate each partition, it provides additional protection against failure**. If a single replica fails, queries can be directed towards a working copy.
* **If you reach the physical limits of a partitioning strategy, you might need to extend the scalability to a different level**. For example, if partitioning is at the database level, you might need to locate or replicate partitions in multiple databases. If partitioning is already at the database level, and physical limitations are an issue, it might mean that you need to locate or replicate partitions in multiple hosting accounts.
* **Avoid transactions that access data in multiple partitions**. Some data stores implement transactional consistency and integrity for operations that modify data, but only when the data is located in a single partition. If you need transactional support across multiple partitions, you will probably need to implement this as part of your application logic because most partitioning systems do not provide native support.

All data stores require some operational management and monitoring activity. The tasks can range from loading data, backing up and restoring data, reorganizing data, and ensuring that the system is performing correctly and efficiently.

Consider the following factors that affect operational management:

* **How to implement appropriate management and operational tasks when the data is partitioned**. These tasks might include backup and restore, archiving data, monitoring the system, and other administrative tasks. For example, maintaining logical consistency during backup and restore operations can be a challenge.
* **How to load the data into multiple partitions and add new data that's arriving from other sources**. Some tools and utilities might not support sharded data operations such as loading data into the correct partition. This means that you might have to create or obtain new tools and utilities.
* **How to archive and delete the data on a regular basis**. To prevent the excessive growth of partitions, you need to archive and delete data on a regular basis (perhaps monthly). It might be necessary to transform the data to match a different archive schema.
* **How to locate data integrity issues**. Consider running a periodic process to locate any data integrity issues such as data in one partition that references missing information in another. The process can either attempt to fix these issues automatically or raise an alert to an operator to correct the problems manually. For example, in an e-commerce application, order information might be held in one partition but the line items that constitute each order might be held in another. The process of placing an order needs to add data to other partitions. If this process fails, there might be line items stored for which there is no corresponding order.

Different data storage technologies typically provide their own features to support partitioning. The following sections summarize the options that are implemented by data stores commonly used by Azure applications. They also describe considerations for designing applications that can best take advantage of these features.

## Rebalancing partitions
As a system matures and you understand the usage patterns better, you might have to adjust the partitioning scheme. For example, individual partitions might start attracting a disproportionate volume of traffic and become hot, leading to excessive contention. Additionally, you might have underestimated the volume of data in some partitions, causing you to approach the limits of the storage capacity in these partitions. Whatever the cause, it is sometimes necessary to rebalance partitions to spread the load more evenly.

In some cases, data storage systems that don't publicly expose how data is allocated to servers can automatically rebalance partitions within the limits of the resources available. In other situations, rebalancing is an administrative task that consists of two stages:

1. Determining the new partitioning strategy to ascertain:
   * Which partitions might need to be split (or possibly combined).
   * How to allocate data to these new partitions by designing new partition keys.
2. Migrating the affected data from the old partitioning scheme to the new set of partitions.

> [!NOTE]
> The mapping of database collections to servers is transparent, but you can still reach the storage capacity and throughput limits of a Cosmos DB account. If this happens, you might need to redesign your partitioning scheme and migrate the data.
>
>

Depending on the data storage technology and the design of your data storage system, you might be able to migrate data between partitions while they are in use (online migration). If this isn't possible, you might need to make the affected partitions temporarily unavailable while the data is relocated (offline migration).

## Offline migration
Offline migration is arguably the simplest approach because it reduces the chances of contention occurring. Don't make any changes to the data while it is being moved and restructured.

Conceptually, this process includes the following steps:

1. Mark the shard offline.
2. Split-merge and move the data to the new shards.
3. Verify the data.
4. Bring the new shards online.
5. Remove the old shard.

To retain some availability, you can mark the original shard as read-only in step 1 rather than making it unavailable. This allows applications to read the data while it is being moved but not to change it.

## Online migration
Online migration is more complex to perform but less disruptive to users because data remains available during the entire procedure. The process is similar to that used by offline migration, except that the original shard is not marked offline (step 1). Depending on the granularity of the migration process (for example, whether it's done item by item or shard by shard), the data access code in the client applications might have to handle reading and writing data that's held in two locations (the original shard and the new shard).

For an example of a solution that supports online migration, see the article [Scaling using the Elastic Database split-merge tool] on the Microsoft website.

## Related patterns and guidance
When considering strategies for implementing data consistency, the following patterns might also be relevant to your scenario:

* The [Data consistency primer] page on the Microsoft website describes strategies for maintaining consistency in a distributed environment such as the cloud.
* The [Data partitioning guidance] page on the Microsoft website provides a general overview of how to design partitions to meet various criteria in a distributed solution.
* The [sharding pattern] as described on the Microsoft website summarizes some common strategies for sharding data.
* The [index table pattern] as described on the Microsoft website illustrates how to create secondary indexes over data. An application can quickly retrieve data with this approach, by using queries that do not reference the primary key of a collection.
* The [materialized view pattern] as described on the Microsoft website describes how to generate pre-populated views that summarize data to support fast query operations. This approach can be useful in a partitioned data store if the partitions that contain the data being summarized are distributed across multiple sites.
* The [Using Azure Content Delivery Network] article on the Microsoft website provides additional guidance on configuring and using Content Delivery Network with Azure.

## More information
* The page [What is Azure SQL Database?] on the Microsoft website provides detailed documentation that describes how to create and use SQL databases.
* The page [Elastic Database features overview] on the Microsoft website provides a comprehensive introduction to Elastic Database.
* The page [Scaling using the Elastic Database split-merge tool] on the Microsoft website contains information about using the split-merge service to manage Elastic Database shards.
* The page [Azure storage scalability and performance targets](https://msdn.microsoft.com/library/azure/dn249410.aspx) on the Microsoft website documents the current sizing and throughput limits of Azure Storage.
* The page [Performing entity group transactions] on the Microsoft website provides detailed information about implementing transactional operations over entities that are stored in Azure table storage.
* The article [Azure Storage table design guide] on the Microsoft website contains detailed information about partitioning data in Azure table storage.
* The page [Using Azure Content Delivery Network] on the Microsoft website describes how to replicate data that's held in Azure blob storage by using the Azure Content Delivery Network.
* The page [What is Azure Search?] on the Microsoft website provides a full description of the capabilities that are available in Azure Search.
* The page [Service limits in Azure Search] on the Microsoft website contains information about the capacity of each instance of Azure Search.
* The page [Supported data types (Azure Search)] on the Microsoft website summarizes the data types that you can use in searchable documents and indexes.
* The page [Azure Redis Cache] on the Microsoft website provides an introduction to Azure Redis Cache.
* The [Partitioning: how to split data among multiple Redis instances] page on the Redis website provides information about how to implement partitioning with Redis.
* The page [Running Redis on a CentOS Linux VM in Azure] on the Microsoft website walks through an example that shows you how to build and configure a Redis node running as an Azure VM.
* The [Data types] page on the Redis website describes the data types that are available with Redis and Azure Redis Cache.

[Availability and consistency in Event Hubs]: /azure/event-hubs/event-hubs-availability-and-consistency
[azure-limits]: /azure/azure-subscription-service-limits
[Azure Content Delivery Network]: /azure/cdn/cdn-overview
[Azure Redis Cache]: https://azure.microsoft.com/services/cache/
[Azure Storage Scalability and Performance Targets]: /azure/storage/storage-scalability-targets
[Azure Storage Table Design Guide]: /azure/storage/storage-table-design-guide
[Building a Polyglot Solution]: https://msdn.microsoft.com/library/dn313279.aspx
[cosmos-db-ru]: /azure/cosmos-db/request-units
[Data Access for Highly-Scalable Solutions: Using SQL, NoSQL, and Polyglot Persistence]: https://msdn.microsoft.com/library/dn271399.aspx
[Data consistency primer]: https://aka.ms/Data-Consistency-Primer
[Data Partitioning Guidance]: https://msdn.microsoft.com/library/dn589795.aspx
[Data Types]: https://redis.io/topics/data-types
[cosmosdb-sql-api]: /azure/cosmos-db/sql-api-introduction
[Elastic Database features overview]: /azure/sql-database/sql-database-elastic-scale-introduction
[event-hubs]: /azure/event-hubs
[Federations Migration Utility]: https://code.msdn.microsoft.com/vstudio/Federations-Migration-ce61e9c1
[guidelines and recommendations for reliable collections in Azure Service Fabric]: /azure/service-fabric/service-fabric-reliable-services-reliable-collections-guidelines
[Index Table Pattern]: ../patterns/index-table.md
[Materialized View Pattern]: ../patterns/materialized-view.md
[Multi-shard querying]: /azure/sql-database/sql-database-elastic-scale-multishard-querying
[Overview of Azure Service Fabric]: /azure/service-fabric/service-fabric-overview
[Partition Service Fabric reliable services]: /azure/service-fabric/service-fabric-concepts-partitioning
[Partitioning: how to split data among multiple Redis instances]: https://redis.io/topics/partitioning
[Performing Entity Group Transactions]: https://msdn.microsoft.com/library/azure/dd894038.aspx
[Redis cluster tutorial]: https://redis.io/topics/cluster-tutorial
[Running Redis on a CentOS Linux VM in Azure]: https://blogs.msdn.microsoft.com/tconte/2012/06/08/running-redis-on-a-centos-linux-vm-in-windows-azure/
[Scaling using the Elastic Database split-merge tool]: /azure/sql-database/sql-database-elastic-scale-overview-split-and-merge
[Using Azure Content Delivery Network]: /azure/cdn/cdn-create-new-endpoint
[Service Bus quotas]: /azure/service-bus-messaging/service-bus-quotas
[service-fabric-reliable-collections]: /azure/service-fabric/service-fabric-reliable-services-reliable-collections
[Service limits in Azure Search]:  /azure/search/search-limits-quotas-capacity
[Sharding pattern]: ../patterns/sharding.md
[Supported Data Types (Azure Search)]:  https://msdn.microsoft.com/library/azure/dn798938.aspx
[Transactions]: https://redis.io/topics/transactions
[What is Event Hubs?]: /azure/event-hubs/event-hubs-what-is-event-hubs
[What is Azure Search?]: /azure/search/search-what-is-azure-search
[What is Azure SQL Database?]: /azure/sql-database/sql-database-technical-overview
