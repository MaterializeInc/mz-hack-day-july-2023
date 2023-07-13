# Advanced Concepts

Materialize's core value proposition is about keeping SQL views up to date for you.
Materialize does this in a way that is meant to be very predictable and reassuring, providing the strongest of database consistency levels.
However, at the very same time it does some clever things on the side that you often expect to come into conflict with that structure: improving performance, flexibility, and scalability. 

We will hit on a few of these, many of which play a part in bringing a real application to life.
1. [Serializability, Linearizability, and Transactions](#serializability-linearizability-and-transactions)
1. [Isolating and scaling workloads with clusters and materialized views](#clusters-and-materialized-views)
1. [`EXPLAIN`-ing query plans](#explain-ing-querys)
1. [Optimization: Focusing on recent data](#temporal-filters-and-mz_now)
1. [Optimization: Reducing append-only data](#append-only-optimizations)
1. [Optimization: Using indexes to make `JOIN` efficient](#join-planning-and-indexes)

## Serializability, Linearizability, and Transactions

All collections of data in Materialize change as they advance through time.
Materialize records these changes with explicit timestamps, called "virtual time".
All `SELECT` queries in Materialize execute at a specific virtual time.
The produced results are exactly correct for the input data at this specific time.
Materialize behaves *as if* it proceeded time-by-time processing input changes and answering queries.
This property is called ["serializability"](https://jepsen.io/consistency/models/serializable) in database lingo.

You can access the logical time using the `mz_now()` function.
```sql
SELECT mz_now(), COUNT(*) FROM auctions;
```

Serializability is a very powerful property, but one you may not notice until you don't have it.
With serializability, all views are always "in sync". 
If you compute the same thing two different ways, their content will always be identical.
If you collate independently defined views, their results are always internally consistent.
If you and your colleagues produce results at the same `mz_now()`, your numbers will all add up.

Without serializability, it is harder to be certain of your conclusions and to react with confidence.

You can see what timestamp would be used for a query with the [`EXPLAIN TIMESTAMP`](https://materialize.com/docs/sql/explain/#timestamps) command.
The command also explains (in some detail) the reasons Materialize would choose a certain timestamp.

```sql
EXPLAIN TIMESTAMP FOR SELECT COUNT(*) FROM auctions;
```

### Linearizability

Although the property of serializability ensures that you'll always see a consistent moment in time, it does not guarantee that this time moves forward.
Strictly speaking, you'll need ["linearizability"](https://jepsen.io/consistency/models/linearizable) for this, also called ["strict serializability"](https://jepsen.io/consistency/models/strict-serializable) when it is combined with serializability.
This is a stronger isolation level, which ensures that the order of `mz_now()` assigned to each command comports with the real-world order of commands: if one command starts after another finishes, it will execute at a strictly greater `mz_now()` time.

Without linearizability, you can see anomalies where you see results in one view that are, mysteriously, not evident in another view.
For example you might count your data and see that it is non-zero, and then aggregate it and see that there is a `NULL` aggregate (corresponding to no data).
This is mysterious, but not illegal under the definition of serializability.
You need strict serializability, which adds linearizability, in order to prevent this anomaly.

Fortunately, Materialize defaults to strict serializability, and you'll only see its absence if you weaken the transaction isolation to the `serializable` level.
```sql
SET transaction_isolation = 'serializable';
```
You may also notice an increase in the snappiness of queries, which is the main advantage of allowing Materialize to answer queries potentially out of order.

To re-enable strict serializability you set the isolation level to `strict serializable`.
```sql
SET transaction_isolation = 'strict serializable';
```

---

**CHALLENGE**: Set your isolation level to `'serializable'` and see if you can frame two queries that produce `mz_now()` values in the wrong order. Make sure to put `mz_now()` as one of the things you select. As a hint, compare the times you see selecting from tables and base sources from the times you see selecting out of indexed views.

Set your isolation level back to `'strict serializable'` and see what happens when you run the same sorts of queries. Do you see the ordering anomalies (we hope not), and if not what do you see instead?

---


### Transactions

Another way to ensure that you receive consistent results is to use transactions.
A transaction is a collection of commands, started with `BEGIN` and concluded with `COMMIT`. All commands between these two will execute at the same logical timestamp.

Transactions are good for short-term coupling of commands, but not for long term interation with Materialize.
Transactions will eventually time out, and it's not until you `COMMIT` a transaction that Materialize commits to the interaction being valid.

```sql
-- Be certain that two queries produce consistent results.
BEGIN;
SELECT mz_now(), COUNT(*) FROM auctions;
SELECT mz_now(), MAX(end_time) FROM auctions;
COMMIT;
```

Materialize does not support all commands within transactions, for technical reasons that we can totally discuss if you are interested!

## Clusters and Materialized Views

You and your colleagues may want to collaborate on the same data, in the same Materialize instance.
At the same time, you and your colleagues may have different standards for things like "is it ok to hose the system" or "how important is response time for my use case".
You might benefit from **performance isolation**: independent resources whose use cannot negatively impact others.

Materialize provides the abstraction of a [cluster](https://materialize.com/docs/sql/create-cluster/) to isolate performance. 
A cluster corresponds to a private collection of resources that nonetheless share access to the same input tables, sources, and views.
Clusters use the same timestamps, and work across independent clusters still provides serializable and strict serializable guarantees.
Each user session has a `cluster` session variable, and commands will be routed to that cluster.
Each Materialize instance has a `default` cluster and each session defaults to using it.

When you create a new cluster you can choose to provision it with specific named resources, which is helpful if you want to manually alter these resources, or you can use the `MANAGED` keyword to task Materialize with this responsibility.
```sql
-- Create a new cluster with a single replica.
CREATE CLUSTER c1 REPLICAS (
    r1 (SIZE = 'xsmall')
);
-- Aim new work at this cluster.
SET cluster = 'c1';
```

### Clusters and Indexes and Materialized Views

Indexes are local to a cluster.
The resources consumed by an index, and the value provided to queries by the index, are restricted to the cluster where the index was created.
If your use case requires indexes that come at some cost, memory and computation, it may make sense to provision an isolated cluster to ensure you get the intended benefits of the indexes *and* do not impose on other work.

To share derived results between clusters Materialize provides [materialized views](https://materialize.com/docs/sql/create-materialized-view/).
These are views whose results are recorded to persistent storage, like a table or source, and are available across clusters.
Unlike indexes, they are not resident in memory nor randomly accessible by key columns.
Like indexes, they allow you to avoid re-computing the same views.

A common workflow has one cluster perform data ingestion, cleaning, and normalization work common to all users, and to materialize the results.
Independent use cases then have their own clusters where they load and index the subsets of this data relevant to the use cases.
All interactions with each of the use cases are still serializable and strict serializable, even if their derived results are mashed up.

### Rescaling Clusters

The work done on a cluster is mirrored on each of its replicas.
This means that when you add replicas, the same work is done twice, and the system responds with whichever results it receives first (they should be identical, so no worries).
This means that you can change the replica allocation to a cluster seamlessly, by adding a new replica, waiting for it to catch up, and then removing the old replica.

---

**CHALLENGE**: Having created a new cluster, see if you can overwhelm it with complex queries and/or indexes. 
Joining data with itself and indexing the results is often a good way to do this.
Confirm that you still have interactive, performant access to the `default` cluster.
Clean up the overwhelmed cluster by dropping the expensive indexes or canceling the expensive queries

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

**CHALLENGE:** Introduce a `mz_now() <= end_time` constraint to focus on recent auctions.
Notice where that predicate appears, and the newly reported `Source` output. Use `EXPLAIN` with your modified view definition bounding `bids` (you can use `EXPLAIN VIEW <view name>` to get explanations for existing views). Confirm that both `auctions` and `bids` have a `Source` output that correspond to the bounds you introduced.

---

You can also use our dataflow visualizer. 
From the Materialize Console, navigate to "Connect", and then the "External tools" tab.
You can `https://` to the HOST there, use the USER to log in, and provide an App password as generated below.
This gives you access to several dataflow visualizers that can show you the shape of the dataflows, numbers of records that have moved along each edge, and number of records retained in each arrangement.

This tool can be very helpful to see the difference in rendered query plans, as the information comes directly from the execution layer.

You should also be able to use an experimental visualizer through the web console: clicking through clusters, to indexes, should have a "visualize" button to the right: this will present the same content in a similar form, without the navigation to a mysterious address.

## Temporal Filters and `mz_now()`

As our auction example continues the volume of data will only increase.
Ad-hoc queries will take increasing amounts of time to run; our indexes will grow without bound.

However, auctions have an `end_time` column, indicating when they conclude.
We can focus our attention on auctions that are still open using a `WHERE` clause that compares `end_time` with the current `mz_now()` time.

```sql
-- Select auctions that ended only recently.
SELECT * FROM auctions WHERE mz_now() <= end_time + INTERVAL '1 minute';
```
Written this way, Materialize can also see how the results change as a function of either `mz_now()` or `auctions`.
```sql
-- Select auctions that ended only recently.
CREATE VIEW recent_auctions AS
SELECT * 
FROM auctions 
WHERE mz_now() <= end_time + INTERVAL '1 minute';

-- We can keep the view up to date despite `mz_now()`.
CREATE DEFAULT INDEX ON recent_auctions;
```

The maintained index will maintain only those records that satisfy the predicate, and retire records that no longer satisfy the predicate.
Its resource requirements can avoid growing without bound.
This can be especially helpful when your data are collections of events, which are never explicitly retracted and instead only age out.

---

**CHALLENGE**: Our `max_bid_by_auction` view uses all of `auctions`. Rewrite it to use only those `auctions` that have not yet ended. 

When you create an index on the updated `max_bid_by_auction` view, will that index have a bounded footprint? What if anything might allow us to forget about records in `bids`? Will we just have to maintain all bids forever? What assumptions would allow you to update the view again in a way that will maintain only recent bids? How can you validate such an assumption?

Use the dataflow visualizer to observe the number of records maintained in each arrangement, when each are filtered to only recent events.

---


## Append-only Optimizations

Quite often, your continually changing data are often append-only: records are added and never removed.
This would eventually overwhelm conventional databases, but it needn't overwhelm Materialize when it is instructed to maintain specific queries.

The opportunities here are subtle. 
If we are tracking the MAX of a column in an append-only collection, we can be clever and only retain the maximum value, rather than all possible values (which we might need if you could remove the current maximum).
If we are keeping the `LIMIT 1` of an append-only collection, we can retain the best row itself rather than an entire collection.
These tricks apply as well to operations on derived views as much as on raw collections.

Let's take the example of maintaining the top bidder for each auction.
We currently do this in `max_bid_by_auction`, though our instruction to Materialize is to first join `bids` and `auctions`, then determine the best bid for each auction.
However, to do this we have indexed `bids` by `auction_id` up above; that's a lot of data and its just growing.
Do we really need to keep *all* of that bid data, if it only ever grows?

Alternately, we could *first* reduce `bids` and then join it with `auctions`.
```sql
-- Retain the top bid for each auction.
CREATE VIEW max_bid_by_auctionid AS
SELECT DISTINCT ON(auction_id) bids.*
FROM bids
ORDER BY auction_id, bids.amount DESC;

-- Notice: just a join; no `DISTINCT ON` here.
CREATE VIEW max_bid_by_auction_lean AS
SELECT auction_id, amount, mbbai.id as bid_id
FROM max_bid_by_auctionid mbbai, auctions
WHERE bid_time <= end_time
  AND mbbai.auction_id = auctions.id;
```

---

**CHALLENGE**: Use the dataflow visualizer to observe the number of records maintained in the arrangement for `max_bid_by_auctionid`. You should also be able to use `EXPLAIN PHYSICAL PLAN` to observe that the `TopK` operator uses a special variant: `MonotonicTop1`.

What happens if you first apply a temporal filter (`WHERE mz_now() <= ...`) to `bids`?
Does that make sense?

---

## Join planning and Indexes

We saw earlier how indexes (from `CREATE INDEX`) are helpful for maintaining results, and providing random access for `SELECT` queries with `WHERE column = literal` constraints.
In addition, they can make joins substantially more efficient, because the joins can re-use existing indexes instead of creating their own.

```sql
-- Examine the initial plan for the query.
EXPLAIN SELECT * FROM max_bid_by_auction;
```

```sql
-- Create an index on auctions by their primary key.
CREATE INDEX auctions_by_id ON auctions (id);
```

```sql
-- Re-examine the plan for the query.
EXPLAIN SELECT * FROM max_bid_by_auction;
```
Notice the output now contains an entry for `Used Indexes`.
This tells us that we are able to make use of the index.

To be entirely certain *how* we are using the index, we can dig deeper with `EXPLAIN PHYSICAL PLAN`
```sql
EXPLAIN PHYSICAL PLAN FOR SELECT * FROM max_bid_by_auction;
```
Notice this line, which communicates that we are directly passing arranged data in.
```
Get::PassArrangements materialize.public.auctions
```

By contrast, the `bids` collection is subjected to an `ArrangeBy` operator, meaning we will have to build an index as part of the query.
By volume, `bids` is substantially larger than `auctions`.
While our `auctions_by_id` index is helpful, it doesn't make the query pop yet.

We could create an index on `bids`, but instead let's create an index on `max_bid_by_auctionid`, which will let us compare the two approaches.
```sql
CREATE INDEX mbbai ON max_bid_by_auctionid (auction_id);
```

We can now look at the plan for `max_bid_by_auction_lean`, and see that it only passes through arrangements; it does not build any of its own.
```sql
EXPLAIN PHYSICAL PLAN FOR SELECT * FROM max_bid_by_auction_lean;
```

Performing this `SELECT` should cause no additional data to be arranged, and maintaining the view will require nothing more than the space for the results.
Maintaining `COUNT(*)` of the results, for example, should have a nominal impact.

---

**CHALLENGE**: Use a dataflow visualizer to check out the plan for a maintained `SELECT COUNT(*) FROM max_bid_by_auction_lean`. 