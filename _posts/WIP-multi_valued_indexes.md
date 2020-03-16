---
layout: post
title: Storage and Indexed access of denormalized columns (arrays) on MySQL 8.0, via multi-valued indexes
tags: [databases,data_types,indexes,mysql]
---

Another "missing and missed" functionality in MySQL is a data type for arrays.

While MySQL is not there yet, it's now possible to cover a significant use case: storing denormalized columns and accessing them via index.

In this article I'll give some context about denormalized data and indexes, including the workaround for such functionality on MySQL 5.7, and describe how this is (rather) cleanly accomplished on MySQL 8.0.

References:

- select multiple rows/TABLE (8.0.19): https://dev.mysql.com/doc/refman/8.0/en/create-table-select.html
- multi-valued index (8.0.17): https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-multi-valued
  - worklog: https://dev.mysql.com/worklog/task/?id=8763
    - high-level https://dev.mysql.com/worklog/task/?id=8763#tabs-8763-4
    - low-level: https://dev.mysql.com/worklog/task/?id=8763
- https://www.compose.com/articles/take-a-dip-into-postgresql-arrays
- https://www.quora.com/In-database-design-what-exactly-is-the-difference-between-inverted-index-and-a-normal-index
- https://www.geeksforgeeks.org/inverted-index
- https://www.postgresql.org/docs/current/gin-implementation.html
- https://programmer.ink/think/mysql8.0.17-introduction-to-multi-valued-indexes.html
- https://dev.mysql.com/doc/refman/8.0/en/fulltext-fine-tuning.html#fulltext-optimize

WL#8955: Add support for multi-valued indexes
- https://dev.mysql.com/worklog/task/?id=8955#tabs-8955-4
  > In general, multi-valued index is a regular functional index, with the exception that it requires additional handling under the hood on INSERT/UPDATE for multi-valued key parts.
  > A key with multi-valued key part won't support ordering
  > There are two limitations on how much keys per single data row one can index. It's max number of keys and total keys length. Both are dictated by the storage engine via new handler::ha_mv_key_capacity() function. In case of InnoDB it's 655335 and 10000 bytes respectively. InnoDB limits it by how much data can be stored in an single undo log page. To give an idea how much it is: currently InnoDB allows to index 1250 integer values at most.
  > Currently mult-valued index supports only two charsets - binary and utf8mb4_0900_as_cs. The reason is that they are most close to how memcmp() compares data (this is what our JSON type implementation uses to compare strings).



- https://dev.mysql.com/doc/refman/8.0/en/range-optimization.html
- https://dev.mysql.com/doc/internals/en/optimizer-range-join-type.html
- https://dev.mysql.com/doc/internals/en/optimizer-partition-pruning.html
- https://github.com/mysql/mysql-server/blob/ea7d2e2d16ac03afdd9cb72a972a95981107bf51/sql/opt_range.cc#L281

> An index with a multi-valued key part does not support ordering and therefore cannot be used as a primary key. For the same reason, a multi-valued index cannot be defined using the ASC or DESC keyword.

> The maximum number of values per record for a multi-valued index is determined by the amount of data than can be stored on a single undo log page, which is 65221 bytes (64K minus 315 bytes for overhead), which means that the maximum total length of key values is also 65221 bytes. The maximum number of keys depends on various factors, which prevents defining a specific limit. Tests have shown a multi-valued index to permit as many as 1604 integer keys per record, for example. When the limit is reached, an error similar to the following is reported: ERROR 3905 (HY000): Exceeded max number of values per record for multi-valued index 'idx' by 1 value(s).

WRITEME: explain requirement for 8.0.17

WRITEME: data types

WRITEME: performance considerations

Contents:

- [Inverted indexes and the PostgreSQL GIN data structure](#inverted-indexes-and-the-postgresql-gin-data-structure)
- [Denormalized columns, and a/the MySQL 5.7 workaround](#denormalized-columns-and-athe-mysql-57-workaround)
- [A big warning on InnoDB fulltext indexes: don't use them unless strictly necessary](#a-big-warning-on-innodb-fulltext-indexes-dont-use-them-unless-strictly-necessary)
- [Limits!](#limits)
  - [array size](#array-size)
  - [int size](#int-size)
- [UNSIGNED ARRAY columns](#unsigned-array-columns)

## Inverted indexes and the PostgreSQL GIN data structure

http://rachbelaid.com/introduction-to-postgres-physical-storage/

A document can be represented as an array - an array of tokens (words). However, this doesn't imply that arrays and documents are optimally handled by the same data structures and algorithms.

Documents, which are, at the very least, considered different from basic arrays in the fact that they're longer, are (more) optimally indexed via the so-called "inverted indexes".

PostgreSQL has two options for fulltext indexes; the one relevant in this context is the ["GIN" (Generalized inverted) index](https://www.postgresql.org/docs/current/gin-implementation.html).

A high-level representation is the following:

WRITEME: pgsql image

This complex data representation is composed of:

- the main data structure is B-tree, used for navigating the index;
- the "posting trees" are B-trees as well, 

## Denormalized columns, and a/the MySQL 5.7 workaround

One of the uses cases for indexed denormalized columns is date warehousing tables; it's typical to store the foreign keys of a 1-to-many relationship in a single field.

However, in MySQL this is not directly possible.

Storing a string of separated integers (I'll assume that foreign key values are integers, but the concept applies in general), and put an index, doesn't work: indexes on strings can cover only the leftmost prefix, and, additionally, they're not very efficient.

WRITEME: inverted indexes

WRITEME: performance

## A big warning on InnoDB fulltext indexes: don't use them unless strictly necessary

WRITME: testing

```sql
-- WRITEME: bug reference
-- WRITEME: sorted? no; normalized? yes

-- The alias column names are optional; by default, they're named `column<n>`.
--
DROP TABLE IF EXISTS warehousing_table;
CREATE TABLE warehousing_table
(
    id           INT NOT NULL PRIMARY KEY,
    customer_ids JSON NOT NULL
)
SELECT * FROM
(
  VALUES 
    ROW(1, '[1, 2]'),
    ROW(2, '[2, 3, 4]'),
    ROW(3, '[5,3,1]')
) v (id, customer_ids);

ALTER TABLE warehousing_table ADD KEY customer_ids ( (CAST(customer_ids -> '$' AS UNSIGNED ARRAY)) );

TABLE warehousing_table;

-- +----+--------------+
-- | id | customer_ids |
-- +----+--------------+
-- |  1 | [1, 2]       |
-- |  2 | [2, 3, 4]    |
-- |  3 | [5, 3, 1]    |
-- +----+--------------+

SELECT id, JSON_EXTRACT(customer_ids, '$[1]') `array_item_1` FROM warehousing_table;

-- +----+--------------+
-- | id | array_item_1 |
-- +----+--------------+
-- |  1 | 2            |
-- |  2 | 3            |
-- |  3 | 3            |
-- +----+--------------+

SELECT id, customer_ids FROM warehousing_table WHERE 1 MEMBER OF (customer_ids -> '$');

-- +----+--------------+
-- | id | customer_ids |
-- +----+--------------+
-- |  1 | [1, 2]       |
-- |  3 | [5, 3, 1]    |
-- +----+--------------+

SELECT id, customer_ids FROM warehousing_table WHERE JSON_OVERLAPS(customer_ids -> '$', CAST('[1, 2]' AS JSON));

-- +----+--------------+
-- | id | customer_ids |
-- +----+--------------+
-- |  1 | [1, 2]       |
-- |  2 | [2, 3, 4]    |
-- |  3 | [5, 3, 1]    |
-- +----+--------------+

EXPLAIN FORMAT=TREE SELECT id, customer_ids FROM warehousing_table WHERE 2 MEMBER OF (customer_ids -> '$');

-- -> Filter: json'2' member of (cast(json_extract(warehousing_table.customer_ids,_utf8mb4'$') as unsigned array))  (cost=0.35 rows=1)
--     -> Index lookup on warehousing_table using customer_ids (cast(json_extract(warehousing_table.customer_ids,_utf8mb4'$') as unsigned array)=json'2')  (cost=0.35 rows=1)

EXPLAIN FORMAT=TREE SELECT id, customer_ids FROM warehousing_table WHERE JSON_OVERLAPS(customer_ids -> '$', CAST('[1, 2]' AS JSON));

-- -> Filter: json_overlaps(cast(json_extract(warehousing_table.customer_ids,_utf8mb4'$') as unsigned array),json'[1, 2]')  (cost=2.31 rows=4)
--     -> Index range scan on warehousing_table using customer_ids  (cost=2.31 rows=4)

EXPLAIN FORMAT=TREE SELECT id, customer_ids FROM warehousing_table WHERE JSON_OVERLAPS(customer_ids -> '$', CAST('[1]' AS JSON));

-- -> Filter: json_overlaps(cast(json_extract(warehousing_table.customer_ids,_utf8mb4'$') as unsigned array),json'[1]')  (cost=1.16 rows=2)
--     -> Index range scan on warehousing_table using customer_ids  (cost=1.16 rows=2)

SHOW INDEXES FROM warehousing_table WHERE Key_name = 'customer_ids'\G

--         Table: warehousing_table
--    Non_unique: 1
--      Key_name: customer_ids
--  Seq_in_index: 1
--   Column_name: NULL
--     Collation: A
--   Cardinality: 3
--      Sub_part: NULL
--        Packed: NULL
--          Null: YES
--    Index_type: BTREE
--       Comment: 
-- Index_comment: 
--       Visible: YES
--    Expression: cast(json_extract(`customer_ids`,_utf8mb4\'$\') as unsigned array)
```

## Limits!

WRITEME: limits

### array size

> Currently, the limits from InnoDB is somehow bounded by undo log size. Since the undo log of INSERT should be written in a single undo log page, so basically, above two factors are restricted by the maximum undo log page size.
> However, it's a not realistic to have a true estimation on the multi-value data length and maximum size of the data array before InnoDB knows the real data. Because the data length of other dynamic fields and each data array size are not known beforehand, so there is no exact estimation for each different row.
> [...]
> Based on this, it should assume nearly the whole undo page is available for one multi-value data, except some hard-coded header content, and the maximum length of the array can be calculated against the multi-value field with the shortest data length in the table.

WRITEME: limits, using CTE+GROUP_CONCAT

### int size

- https://tools.ietf.org/html/rfc7159#section-6

> This specification allows implementations to set limits on the range and precision of numbers accepted.

- https://dev.mysql.com/doc/refman/8.0/en/json.html#json-comparison

> Two JSON arrays are equal if they have the same length and values in corresponding positions in the arrays are equal.

## UNSIGNED ARRAY columns

WRITEME: UNSIGNED ARRAY columns

