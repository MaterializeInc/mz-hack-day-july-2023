# Advanced Concepts

## Serializability, Linearizability, and Transactions

All `SELECT` queries in Materialize execute at a specific "logical time".
The produced results are exactly correct for the inputs at this specific time.
This property is called "serializability" in database lingo.

You can access the logical time using the `mz_now()` function.
```sql
SELECT COUNT(*), mz_now() FROM auctions;
```

Serializability is a very powerful, but one you may not notice until you don't have it.
With serializability, all views are always "in sync". 
If you compute the same thing two different ways, their content will always be identical.
If you collate independently defined views, their results are always internally consistent.

### Linearizability

Although serializability ensures that you'll always see a consistent moment in time, it does not guarantee that this time moves forward.
Strictly speaking, you'll need "linearizability" for this, or "strict serializability" when combined with serializability.
This is a stronger isolation level, which ensures that the order of `mz_now()` assigned to each command comports with the real-world order of commands: if one command starts after another finishes, it will execute at a strictly greater `mz_now()` time.

Without linearizability, you can see anomalies where you see results in one view that are, mysteriously, not evidence in another view.
For example you might count your data and see that it is non-zero, and then aggregate it and see that there is a `NULL` aggregate (corresponding to no data).
This is mysterious, but not illegal under the definition of serializability.
You need strict serializability, which adds linearizability, in order to prevent this anomaly.

Materialize defaults to strict serializability, and you'll only see its absence if you weaken the transaction isolation to the `serializable` level.
```sql
SET transaction_isolation = 'serializable';
```
You may also notice an increase in the snappiness of queries, which is the main advantage of allowing Materialize to answer queries potentially out of order.

To re-enable strict serializability you set the isolation level to `strict serializable`.
```sql
SET transaction_isolation = 'strict serializable';
```

### Transactions

Another way to ensure that you receive consistent results is to use transactions.
A transaction is a collection of commands, started with `BEGIN` and concluded with `COMMIT`. All commands between these two will execute at the same logical timestamp.

Transactions are good for short-term coupling of commands, but not for long term interation with Materialize.
Transactions will eventually time out, and it's not until you `COMMIT` a transaction that Materialize commits to the interaction being valid.

```sql
BEGIN;
SELECT mz_now(), COUNT(*) FROM auctions;
SELECT mz_now(), MAX(end_time) FROM auctions;
COMMIT;
```

## Temporal Filters and `mz_now()`

As our auction example continues the volume of data will only increase.
Ad-hoc queries will take increasing amounts of time to run; our indexes will grow without bound.

However, auctions have an `end_time` column, indicating when they conclude.
We can focus our attention on auctions that are still open using a `WHERE` clause that compares `end_time` with the current time.

```sql
-- Select auctions that ended only recently.
SELECT * FROM auctions WHERE end_time >= now() - INTERVAL '1 minute';
```
In addition to `auctions` changing, the value of `now()` continually changes as well.
We can rewrite the `WHERE` predicate that shows when the results change as a function of `now()`.
```sql
-- Rewritten, we see how the predicate changes when `now()` changes.
SELECT * FROM auctions WHERE now() <= end_time + INTERVAL '1 minute';
```
Written this way, *Materialize* can also see how the results change as a function of `now()`.
Except, we ask you to use `mz_now()` instead, because we are only able to use Materialize's timestamp rather than the system time of the computer (what PostgreSQL wants `now()` to mean).
```sql
-- Change to `mz_now()` because `now()` is taken.
CREATE VIEW recent_auctions AS
SELECT * 
FROM auctions 
WHERE mz_now() <= end_time + INTERVAL '1 minute';

-- We are able to index the view because we compare using `mz_now()`.
CREATE DEFAULT INDEX ON recent_auctions;
```

The maintained index will maintain only those records that satisfy the predicate, and retire records that no longer satisfy the predicate.
Its resource requirements can avoid growing without bound.
This can be especially helpful when your data are collections of events, which are never explicitly retracted and instead only age out.

---

**CHALLENGE**: Our `max_bid_by_auction` view uses all of `auctions`. Rewrite it to use only those `auctions` that have not yet ended. 

When you create an index on the updated `max_bid_by_auction` view, will that index have a bounded footprint? What if anything might allow us to forget about records in `bids`? Will we just have to maintain all bids forever? What assumptions would allow you to update the view again in a way that will maintain only recent bids? How can you validate such an assumption?

---

## `EXPLAIN`-ing querys

It isn't always abvious what will happen when you `SELECT` or `CREATE VIEW`.
The [`EXPLAIN` command](https://materialize.com/docs/sql/explain/) can show you what Materialize plans to do with your command.
```sql
-- The query plan Materialize will use for the `SELECT` command.
EXPLAIN 
SELECT DISTINCT ON(auction_id) auction_id, amount, id as bid_id
FROM bids_with_auction
WHERE bid_time <= end_time
ORDER BY auction_id, amount DESC;
```
The `EXPLAIN` command reveals detailed information about the structure of the query plan.
The output is a tree whose root (at the top) are the query results, and whose branches are the inputs that will be brought together to compute the result. 
In this case, it shows us that the query involves a join, and a "top k" operator:
```
Optimized Plan
Explained Query:
  Finish order_by=[#0 asc nulls_last, #1 desc nulls_first] output=[#0..=#2]
    TopK group_by=[#0] order_by=[#1 desc nulls_first] limit=1
      Project (#1, #2, #0)
        Filter (#3 <= #5)
          Join on=(#1 = #4) type=differential
            ArrangeBy keys=[[#1]]
              Project (#0, #2..=#4)
                Get materialize.public.bids
            ArrangeBy keys=[[#0]]
              Project (#0, #3)
                Get materialize.public.auctions
```

--- 

**CHALLENGE:** Introduce the `mz_now()` constraint from the temporal filter section.
Notice where that predicate appears, and the newly reported `Source` output. Use `EXPLAIN` with your modified view definition bounding `bids` (you can use `EXPLAIN VIEW <view name>` to get explanations for existing views). Can you confirm that both `auctions` and `bids` have a `Source` output that correspond to the bounds you introduced.

---

## Join planning and Indexes

Indexes (from `CREATE INDEX`) were helpful for maintaining results, and providing random access for `SELECT` queries with `WHERE column = literal` constraints.
In addition, they can make joins substantially more efficient, because the joins can re-use the indexes instead of creating their own.

```sql
-- Examine the initial plan for the query.
EXPLAIN VIEW max_bid_by_auction;
```

```sql
-- Create an index on auctions by their primary key.
CREATE INDEX auctions_by_id ON auctions (id);
```

```sql
-- Re-examine the plan for the query.
EXPLAIN VIEW max_bid_by_auction;
```
Notice the output now containts an entry for `Used Indexes`.
This tells us that we are able to make use of the index.

By volume, `bids` is substantially larger than `auctions`.
While our `auctions_by_id` index is helpful, it doesn't make the query pop yet.
To do that, we'll also need an index on `bids` by `auction_id`.
```sql
CREATE INDEX bids_by_auction_id ON bids (auction_id);
```
Let's look at the plan again.
```sql
EXPLAIN VIEW max_bid_by_auction;
```
There are two changes here, one clear and one more subtle.
First, we have an additional used index: `bids_by_auction_id`.
Second, the `Project` stage is lifted away from `Get materialize.public.bids`: Materialize concludes that it is more efficient to use the existing index than to re-read all the data and immediately project down to the relevant columns.

## Append-only Optimizations

Quite often, your continually changing data are often append-only: records are added and never removed.
This would eventually overwhelm conventional databases, but needn't overwhelm Materialize when it is instructed to maintain specific queries.

**Show off the TopK operator** 

**Push down the TopK operator**

## Clusters and Materialized Views

**Overwhelm `default`**

**Cancel query**

**Create cluster; split work**

**Rescale cluster**