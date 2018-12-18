+++
toc = true
title = "Pagination"
weight = 2
+++

Totally available for pagination queries of MySQL, PostgreSQL and Oracle; partly available for SQLServer pagination query due to its complexity.

## Pagination Performance

### Performance Bottleneck

Pagination with query offset too high can lead to a low data accessibility, take MySQL as an example: 

```sql
SELECT * FROM t_order ORDER BY id LIMIT 1000000, 10
```

This SQL will make MySQL acquire 10 records after skipping 1,000,000 records when it is not able to use indexes. 
Its performance can be seen from it. In sharding databases and sharding tables (suppose two databases), to ensure the data correctness, the SQL will be rewritten as this:

```sql
SELECT * FROM t_order ORDER BY id LIMIT 0, 1000010
```

It also means taking out all the records prior to the offset and only acquire the last 10 records after ordering. 
It will further aggravate the performance bottleneck effect when the database is already slow enough in execution. 
The reason for that is, formerly, the SQL only needs to transmit 10 records to the user end, but now it will transmit `1000010 * 2` records after the rewrite.

### Optimization of ShardingSphere

ShardingSphere has optimized in two ways.

Firstly, it adopts stream process + merger ordering to avoid the excessive occupation of the memory. 
SQL rewrite unavoidably occupies extra bandwidth, but will not lead to sharp increase in memory occupation. 
Most people may assume that ShardingSphere would upload all the `1,000,010 * 2` records to the memory and occupy a large amount of it, which can lead to memory overflow. 
But because the record of each result set has its own order, each comparison of ShardingSphere only acquires current result set record of each shard. 
The record stored in the memory is only the current position pointed by the cursor in the result set of the shard routed to. 
For the item to be sorted, merger ordering only has the time complexity of `O(n)`, with a very low performance consumption.

Secondly, ShardingSphere further optimizes the query that only falls into single shards. 
Requests of this kind can guarantee the correctness of records without rewriting SQLs. 
Under this kind of situation, ShardingSphere will not do that in order to save the bandwidth.

## Pagination Solution Optimization

For LIMIT cannot search for data through indexes, if the ID continuity can be guaranteed, pagination by ID is a better solution:

```sql
SELECT * FROM t_order WHERE id > 100000 AND id <= 100010 ORDER BY id
```

Or use the ID of last record of the former query result to query the next page:

```sql
SELECT * FROM t_order WHERE id > 100000 LIMIT 10
```

## Pagination Sub-query

Oracle and SQLServer pagination both need to be processed by sub-query, ShardingSphere is available for pagination related sub-query.

- Oracle

Available for rownum pagination:

```sql
SELECT * FROM (SELECT row_.*, rownum rownum_ FROM (SELECT o.order_id as order_id FROM t_order o JOIN t_order_item i ON o.order_id = i.order_id) row_ WHERE rownum <= ?) WHERE rownum > ?
```

Unavailable for rownum + BETWEEN pagination for now.

- SQLServer

Available for TOP + ROW_NUMBER() OVER pagination:

```sql
SELECT * FROM (SELECT TOP (?) ROW_NUMBER() OVER (ORDER BY o.order_id DESC) AS rownum, * FROM t_order o) AS temp WHERE temp.rownum > ? ORDER BY temp.order_id
```

Available for OFFSET FETCH pagination after SQLServer 2012:

```sql
SELECT * FROM t_order o ORDER BY id OFFSET ? ROW FETCH NEXT ? ROWS ONLY
```

Unavailable for `WITH xxx AS (SELECT ...)` pagination. 
Because SQLServer automatically generated by Hibernate uses WITH statements, so Hibernate SQLServer pagination or two TOP + sub-query pagination is not available now.

- MySQL, PostgreSQL

Both MySQL and PostgreSQL are available for LIMIT pagination, no need for sub-query:

```sql
SELECT * FROM t_order o ORDER BY id LIMIT ? OFFSET ?
```