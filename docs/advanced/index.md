---
sidebar_position: 1
---

# Index

In a database, users can create indexes on tables to accelerate queries. Indexing in RisingWave is similar to traditional databases and is designed to speed up random queries. Users can not build indexes on **tables and materialized views**.

## When to use index

when you need to seach, use it.

## Syntax

```sql
CREATE INDEX index_name ON object_name ( index_column [ ASC | DESC ], [, ...] )
[ INCLUDE ( include_column [, ...] ) ]
[ DISTRIBUTED BY ( distributed_column [, ...] ) ];
```
## Sample Code
Let's assume we have two tables, `customers` and `orders`.

```sql
CREATE TABLE customers (
    c_custkey INTEGER,
    c_name VARCHAR,
    c_address VARCHAR,
    c_nationkey INTEGER,
    c_phone VARCHAR,
    c_acctbal NUMERIC,
    c_mktsegment VARCHAR,
    c_comment VARCHAR,
    PRIMARY KEY (c_custkey)
);

CREATE TABLE orders (
    o_orderkey BIGINT,
    o_custkey INTEGER,
    o_orderstatus VARCHAR,
    o_totalprice NUMERIC,
    o_orderdate DATE,
    o_orderpriority VARCHAR,
    o_clerk VARCHAR,
    o_shippriority INTEGER,
    o_comment VARCHAR,
    PRIMARY KEY (o_orderkey)
);
```

If we want to speed up the query of fetching a customer record by the phone number, we can build an index on the `c_phone` column in the `customers` table. This is a good example.

```sql
CREATE INDEX idx_c_phone on customers(c_phone);

SELECT * FROM customers where c_phone = '123456789';

SELECT * FROM customers where c_phone in ('123456789', '987654321');
```

If we want to speed up the query of fetching all the orders of a customer by the customer key, we can build an index on the `o_custkey` column in the `orders` table.

```sql
CREATE INDEX idx_o_custkey ON orders(o_custkey);

SELECT * FROM customers JOIN orders ON c_custkey = o_custkey 
WHERE c_phone = '123456789';
```

## How to Decide Which Columns to Include?

By default, RisingWave creates an index that includes all columns of a table or a materialized view if you omit the `INCLUDE` clause. This differs from the standard PostgreSQL. Why? RisingWave's design as a cloud-native streaming database includes several key differences from PostgreSQL, including the use of an object store for more cost-effective storage and the desire to make index creation as simple as possible for users who are not experienced with database systems. By including all columns, RisingWave ensures that an index will cover all of the columns touched by a query and eliminates the need for a primary table lookup, which can be slower in a cloud environment due to network communication. However, RisingWave still provides the option to include only specific columns using the `INCLUDE` clause for users who wish to do so.

For example:

If your queries only access certain columns, you can create an index that includes only those columns. The RisingWave optimizer will automatically select the appropriate index for your query.

```sql
-- Create an index that only includes necessary columns
CREATE INDEX idx_c_phone1 ON customers(c_phone) INCLUDE (c_name, c_address);

-- RisingWave will automatically use index idx_c_phone1 for the following query since it only access the indexed columns.
SELECT c_name, c_address FROM customers WHERE c_phone = '123456789';
```


## How to Decide the Index Distribution Key?

RisingWave will use the first index column as the `distributed_column` by default if you omit the `DISTRIBUTED BY` clause. RisingWave distributes the data across multiple nodes and uses the `distributed_column` to determine how to distribute the data based on the index. If your queries intend to use indexes but only provide the prefix of the `index_column`, it could be a problem for RisingWave to determine which node to access the index data from. To address this issue, you can specify the `distributed_column` yourself, ensuring that these columns are the prefixes of the `index_column`.

For example:

```sql
-- Create an index with specified distributed columns
CREATE INDEX idx_c_phone2 ON customers(c_name, c_nationkey) DISTRIBUTED BY (c_name);

SELECT * FROM customers WHERE c_name = 'Alice';
```