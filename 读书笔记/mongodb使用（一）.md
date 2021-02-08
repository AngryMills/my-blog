# mongodb使用（一）

## Nosql 简介：

Not Only Sql，不只是sql。

- 不遵循关系模型

- 不固定的列记录

- 使用自包含的聚合或BLOB

- 不需要对象关系映射和数据规范化

- 没有复杂的功能，例如查询语言、查询计划等

- 支持 ACID

  

## 常见的 nosql 数据库类型：

- k-v 形式 ： Redis、Memcahed
- Document 形式： MongoDB、CouchDB
- Column 形式：Hbase、Cassandra
- Graph 形式：Neo4J、InfoGrid

基于键值对：（Dynamo论文 https://zhuanlan.zhihu.com/p/148788043）

将数据存储为哈希表，键唯一，值可以使字符串、JSON、BLOB 等。

基于列：

每列都单独处理，单列数据库的值连续存储。

在聚合查询(SUM、COUNT等)提供了高性能。

面向文档：

存储和检索为键值对，值部分为文档。文档以JSON或XML格式存储。

它不应用于需要多种操作或针对不同聚合结构进行查询的复杂交易。

基于图片：

图形类数据库存储实体以及这些实体之间的关系。



## CAP定理和最终一致性ACID

分布式：CAP

- Consistency (一致性)：即更新操作成功并返回客户端后，所有节点在同一时间的数据完全一致。
- Available（可用性）：服务一直可用，而且是正常响应时间。
- Partition Tolerance (分区容错性)：分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性或可用性的服务。

最终一致性： ACID

- 原子性(atomicity): 指所有在事务中的操作要么都成功，要么都不成功，所有的操作都不可分割，没有中间状态。
- 一致性(consistency): 指的是逻辑上的一致性，即所有操作是符合现实当中的期望的。
- 隔离性(isolation): 即不同事务之间的相互影响和隔离的程度。
- 持久性(durability): 数据持久化到系统中。



## Nosql的优点和缺点















