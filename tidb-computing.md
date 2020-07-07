---
title: TiDB database computation
summary: Understand the computation layer of the TiDB database.
category: introduction
aliases: ['/docs/dev/tidb-computing/']
---

# Computation of TiDB database

Based on the distributed storage capability provided by TiKV, TiDB builds the computing engine that combines superior transaction processing with good data analysis capabilities. This article starts with a data mapping algorithm to describe how TiDB maps data from database tables to TiKV's (Key, Value) key-value pairs, then a description of how TiDB manages meta-information, and finally a description of the main architecture of the TiDB SQL layer.

For the computation layer dependent storage schemes, this article only introduces TiKV based row storage structures. For analytic services, TiDB introduces a column storage scheme as a TiKV extension - [TiFlash](/tiflash/tiflash-overview.md).

## Mapping of table data to Key-Value

This section describes the scheme for mapping data to (Key, Value) key-value pairs in TiDB. Data here consists of the following two main aspects:

- Data for each row in the table, hereinafter referred to as table data.
- Data for all indexes in the table, hereinafter referred to as index data.

### Mapping of table data to Key-Value

In a relational database, a table may have many columns. To map the data from each column in a row to a (Key, Value) key-value pair, you need to consider how to construct the Key. First of all, OLTP scenarios have a large number of operations such as adding, deleting, changing and searching for single or multiple rows, which require the database to read a line of data quickly. Therefore, it is best to have a unique ID (either explicit or implicit) for the corresponding key to facilitate a quick location. Second, many OLAP queries require a full table scan. If you can encode the keys of all rows in a table into a range, the whole table can be efficiently scanned by range queries.

Based on the above considerations, the mapping of table data to Key-Value in TiDB is designed as follows:

- To ensure that data from the same table is kept together for easy searching, TiDB assigns a table ID to each table denoted by `TableID`. Table ID is an integer that is unique throughout the cluster.
- TiDB assigns a row ID, represented by `RowID`, to each row of data in the table. The row ID is also an integer, unique within the table. For row ID, TiDB has made a small optimization, if a table has integer type primary key, TiDB will use primary key value as the row ID of this row of data.

Each row of data is encoded as a (Key, Value) key-value pair according to the following rule:

```
Key:   tablePrefix{TableID}_recordPrefixSep{RowID}
Value: [col1, col2, col3, col4]
```

`tablePrefix` and `recordPrefixSep` are both special string constants used to distinguish other data in Key space. The exact value is given in the summary that follows.

### Mapping of Indexed Data to Key-Value

TiDB supports both primary keys and secondary indexes (both unique and non-unique indexes). Similar to the table data mapping scheme, TiDB assigns an index ID to each index in the table indicated by `IndexID`.

For primary keys and unique indexes, it is needed to quickly locate the corresponding RowID based on the key value, so it is encoded as follows (Key, Value) Key-value pairs.

```
Key:   tablePrefix{tableID}_indexPrefixSep{indexID}_indexedColumnsValue
Value: RowID
```

For ordinary secondary indexes that do not need to satisfy the uniqueness constraint, a single key may correspond to multiple rows. It needs to query corresponding RowID according to the range of keys. Therefore, it is encoded as a (Key, Value) key-value pair according to the following rule:

```
Key:   tablePrefix{TableID}_indexPrefixSep{IndexID}_indexedColumnsValue_{RowID}
Value: null
```

### Summary of mapping relationships

`tablePrefix`, `recordPrefixSep`, and `indexPrefixSep` in all of the above encoding rules are string constants that are used to distinguish between other data in Key space, defined as follows:

```
tablePrefix     = []byte{'t'}
recordPrefixSep = []byte{'r'}
indexPrefixSep  = []byte{'i'}
```

Also note that in the above schemes, regardless of table data or index data key encoding scheme, all rows in a table have the same key prefix, and all the data of an index also has the same prefix. Data with the same prefixes are thus arranged together in TiKV's Key space. Therefore, by carefully designing the encoding scheme of the suffix part to ensure that the pre and post-encoding comparisons remain the same, the table data or index data can be stored in the TiKV in an ordered manner. With this encoding, all rows of data in a table are arranged orderly by `RowID` in the TiKV's Key space, and the data for a particular index will also be arranged sequentially in the Key space according to the specific value of the index data (the `indexedColumnsValue`).

### Example of Key-Value mapping relationship

Finally, a simple example is used to understand the Key-Value mapping relationship of TiDB. Suppose the following table exists in TiDB.

```sql
CREATE TABLE User {
     ID int,
     Name varchar(20),
     Role varchar(20),
     Age int,
     PRIMARY KEY (ID),
     KEY idxAge (Age)
};
```

Suppose there are 3 rows of data in the table.

```
1, "TiDB", "SQL Layer", 10
2, "TiKV", "KV Engine", 20
3, "PD", "Manager", 30
```

First, each row of data is mapped to a (Key, Value) key-value pair, and the table has an `int` type primary key, so the value of `RowID` is the value of this primary key. Suppose the table has `TableID` of 10, then its table data stored on TiKV is:

```
t10_r1 --> ["TiDB", "SQL  Layer", 10]
t10_r2 --> ["TiKV", "KV  Engine", 20]
t10_r3 --> ["PD", " Manager", 30]
```

In addition to the primary key, the table has a non-unique ordinary secondary index, `idxAge`. Suppose the `IndexID` is 1, then its index data stored on TiKV is:

```
t10_i1_10_1 --> null
t10_i1_20_2 --> null
t10_i1_30_3 --> null
```

The above example shows the mapping rule from a relational model to a Key-Value model in TiDB, and the consideration behind the selection of this scheme.

## Meta-information management

Each `Database` and `Table` in TiDB has meta information, aka its definition and various attributes. This information also needs to be persistent, and TiDB stores this information in TiKV as well.

Each `Database` / `Table` is assigned an unique ID. As the unique identifier, when encoded as Key-Value, this ID is encoded in the Key with the `m_` prefix. This constructs a Key, and stores the serialized meta information in Value.

Besides, TiDB also uses a dedicated (Key, Value) key pair to store the latest version number of the current all tables structure information. This key-value pair is global, and its version number is increased by 1 each time the state of the DDL operation changes. TiDB stores this key-value pair persistently in the PD Server with a key of "/tidb/ddl/global_schema_version", and Value is the version number value of int64 type. Refers to Google F1's  Online Schema change algorithm, TiDB keeps a background thread constantly checking whether the version number of the table structure information stored in the PD Server changes, and ensuring to get the changes of version in a certain time.

## Introduction to the SQL layer

TiDB's SQL layer, TiDB Server, is responsible for translating the SQL into Key-Value operation to the common distributed Key-Value storage layer TiKV, assembling the results returned by TiKV, and returning the query results to the client ultimately.

The nodes at this layer are stateless, the nodes themselves do not store data, and the nodes are completely equal.

### SQL algorithm

The simplest solution is through the [mapping of table data to Key-Value](#mapping-of-table-data-to-key-value) as described in the previous section scheme, mapping SQL queries to KV queries, and then acquires the corresponding data through the KV interface and performs various computations.

For example, `select count(*) from user where name = "TiDB"` such a SQL statement. It needs to read all the data in the table, then check if the `name` field is `TiDB`, and if so, returns this line. The process is as follows:

1. construct the Key Range: all `RowID` in a table are in `[0,  MaxInt64)` range. According to the row data `Key` encoding rule, using `0` and `MaxInt64` can construct a `[StartKey, EndKey)` range that is left-included, right-excluded.
2. scan Key Range: read the data in TiKV according to the key range constructed above.
3. filter data: for each row of data read, calculate `name = "TiDB"` expression. Returns up this line if true, otherwise discards this line of data.
4. calculate `Count(*)`: for each line that meets the requirements, accumulate to the result of `Count(*)`.

**The entire process is illustrated as follows:**

![naive sql flow](/media/tidb-computing-native-sql-flow.jpeg)

This solution is intuitive and feasible, but has some obvious problems in a distributed database scenario.

- As the data is being scanned, each row is read out of TiKV via a KV operation at least once RPC overhead, which can be very high if there is a lot of data to scan.
- Not all rows meet the filter criteria `name = "TiDB"`. If the conditions are not met, they are unnessary to be read out.
- The value of the rows that meet the requirements doesn't mean anything, in fact, all needed here is the information of how many rows of data.

### Distributed SQL operations

To solve the above problem, the computation should need to be as close to the storage node as possible to avoid a large number of RPC calls. First of all, the SQL predicate condition `name = "TiDB"` should be pushed down to the storage node for computation, so that only valid rows are returned, avoiding meaningless network transfers. Then, the aggregation function `Count(*)` can also be pushed down to the storage nodes for pre-aggregation, and each node only has to return a result of `Count(*)`, and the SQL layer will sum up the `Count(*)` result returned by each node.

The following is a schematic representation of the data returned layer by layer.

![dist sql flow](/media/tidb-computing-dist-sql-flow.png)

### SQL layer architecture

With the above example, I hope you have a basic understanding of how SQL statements are handled. In fact, TiDB's SQL layer is much more complex, with many modules and layers. The following diagram lists the important modules and calling relationships:

![tidb sql layer](/media/tidb-computing-tidb-sql-layer.png)

The user's SQL request is sent to TiDB Server either directly or via `Load Balancer`. TiDB Server will parse `MySQL Protocol Packet`, getting the content of requests, parsing syntax and semantic analysis of SQL, developing and optimizing query plans, executing a query plan , fetching and processing the data. The data is all stored in the TiKV cluster, so in this process, TiDB Server needs to interact with the TiKV and gets the data. Finally, TiDB Server needs to return the query results to the user.