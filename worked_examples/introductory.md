# Introducing Materialize

Materialize is a SQL database / data warehouse, one that should feel both familiar (because of SQL) but also surprisingly new at a few key moments.
Our goal is this tour is to get you situated on familiar SQL ground, and then build up to the surprising and we hope eye-opening moments.

Here's the sequence of introductory concepts, linked (and exciting call-out moments!):
1. [Create a table, insert, select](#creating-updating-and-inspecting-tables)
1. [Create a load generator source](#intermission-a-load-generating-source-of-data)
1. [Create a view](#building-up-business-logic-as-views)
1. [Create an index (you can index views!)](#using-indexes-to-make-views-fast)
1. [`SUBSCRIBE` (you can get a view changefeed!)](#subscribing-to-changes)



## Creating, updating, and inspecting tables.

SQL is based around the idea that you have tables and tables contain data.
You can create, modify, and inspect tables.
Let's start with that in Materialize.

```sql
-- Creates a table of pairs of integer and string.
CREATE TABLE test(a int, b string);
```

```sql
-- Introduce two values into `test`.
INSERT INTO test VALUES (1, 'hello'), (3, 'world');
```

```sql
-- Read the current values of out `test`.
SELECT * FROM test;
```

You can modify table contents with `INSERT`, `UPDATE`, and `DELETE`.
You can issue more complicated `SELECT` queries, with all (or almost all) of your favorite SQL fragments.

## Intermission: A load generating source of data

You may not have a live collection of data easily at hand.
In that case, and to put us all on the same page, we'll fire up a load generating source.

```sql
-- Create a load generator source.
CREATE SOURCE auction_house
FROM LOAD GENERATOR AUCTION
FOR ALL TABLES 
WITH (SIZE = '3xsmall');
```
This creates a collection of "sources": SQL tables that are populated and updated exogenously.
You can read from these sources, but you cannot write to them (with e.g. `UPDATE` or `DELETE`).

You can see the available sources with Materialize's `show sources` command.
Notice how we have created a whole bundle of them all at once; they all related to each other other!
```
materialize=> show sources;
          name          |      type      |  size   
------------------------+----------------+---------
 accounts               | subsource      | 
 auction_house          | load-generator | 3xsmall
 auction_house_progress | progress       | 
 auctions               | subsource      | 
 bids                   | subsource      | 
 organizations          | subsource      | 
 users                  | subsource      | 
(7 rows)

materialize=> 
```
Those sources with type `subsource` contain the generated data.
The `load-generator` source is a reference to the bundle of sources (and is not itself queryable), and the `progress` source provides consistency information (and is an advanced topic).

You can inspect each `subsource` with Materialize's `show columns` command.
```
materialize=> show columns from auctions;
   name   | nullable |           type           
----------+----------+--------------------------
 id       | f        | bigint
 seller   | f        | bigint
 item     | f        | text
 end_time | f        | timestamp with time zone
(4 rows)

materialize=> 
```
Of course, you could also `SELECT * FROM auctions` to see the column namse.
As more auctions are introduced over time, this will become an increasingly unappealing way to get the column information.

## Building up business logic as views

Let's start to build up more interesting business logic than counting the number of records.

Records in `bids` reference records in `auctions` though their `auction_id` column.
We can connect bids with their auctions using a SQL join.
```sql
-- Enrich each bid with its associated auction.
CREATE VIEW bids_with_auction AS
SELECT bids.*, auctions.seller, auctions.item, auctions.end_time
FROM bids, auctions
WHERE bids.auction_id = auctions.id;
```

With such a view, we can ask for the greatest bid associated with each auction.
```sql
-- Extract the maximum bid for each auction
CREATE VIEW max_amount_by_auction AS
SELECT auction_id, MAX(amount)
FROM bids_with_auction
WHERE bid_time <= end_time
GROUP BY auction_id;
```

You can use some non-standard PostgreSQL modifications to return the bid itself.
```sql
-- Extract the maximum bid and its id for each auction.
CREATE VIEW max_bid_by_auction AS
SELECT DISTINCT ON(auction_id) auction_id, amount, id as bid_id
FROM bids_with_auction
WHERE bid_time <= end_time
ORDER BY auction_id, amount DESC;
```

There are other connection points among the relations.
Each of `bids.buyer` and `auctions.seller` reference a row in `accounts`.
In turn `accounts.org_id` references a row in `organizations`.
We don't have a reason to join these together at the moment, but they are available for you to play around with.

With views defined, you can select out of the views as a handy shortcut.
```sql
SELECT * FROM max_bid_by_auction
```
A view is only a name given to a query definition, so each time you select from the view Materialize will re-execute your query.

## Using indexes to make views fast

SQL indexes conventionally make access to tables fast.
In Materialize you can also index *views*, and make access to those views fast.
Importantly, the results are as if you used the view: always up to date, never stale cached content.
```sql
CREATE DEFAULT INDEX ON max_bid_by_auction;
```
At this point we can read results directly out of the index, without re-evaluating the view.
```sql
SELECT * FROM max_bid_by_auction;
```
This should go substantially faster than querying the view without the index. 
The results are resident in memory ready to go, rather than re-evaluated when you ask.

The data are also readily available for fast input to other queries.
```sql
SELECT COUNT(*) FROM max_bid_by_auction;
```
Although we are doing more than just reading out of `max_bid_by_auction`, we can connect what we have for it in to the count query and save ourself a lot of time and effort.

All indexes in Materialize are on a set of columns of the relation.
When you `SELECT` constraining those columns to literal values, Materialize will do an efficient indexed look-up rather than a full scan. 
In the case of `max_bids_by_auction`, the default index uses `auction_id` (as Materialize can see that it is a unique key for the relation).

```sql
-- Perform an indexed read from `max_bid_by_auction`.
SELECT * FROM max_bid_by_auction WHERE auction_id = 7;
```

You can specify the columns an index uses in the extended `CREATE INDEX` command.
```sql
CREATE INDEX mba_by_bid ON max_bid_by_auction (bid_id);
```
This now allows us to perform indexed access by bid identifiers.
```sql
-- Perform and indexed read from `max_bid_by_auction`.
SELECT * FROM max_bid_by_auction WHERE bid_id = 13;
```

Indexes do take up memory, as well as impose an ongoing computational burden.
As the data change, Materialize is continually updating all of your indexes proactively.
Materialize's indexes allow you to "pre-pay" computation so that when you need results fast, they are ready for you.

## Subscribing to changes

You may have noticed that the results to certain queries change.
At the same time, it's potentially pretty annoying to have to visually scan `max_bid_by_auction` looking for changed results.
Materialize augments `SELECT` with a subscription counterpart `SUBSCRIBE`, which provides the results `SELECT` would give you, followed by a continually running changefeed.

```sql
-- You'll want to use `psql` to observe this.
COPY (SUBSCRIBE max_bid_by_auction) TO stdout;
```

The changefeed is continually updated, but it may seem ambiguous at times.
Are there no new results because there are no changes, or because something is stuck?
The `WITH (PROGRESS)` option adds in a heartbeart that comminicates when updates for each timestamp are complete.

```sql
-- You'll want to use `psql` to observe this.
COPY (SUBSCRIBE max_bid_by_auction WITH(PROGRESS = true)) TO stdout;
```

`SUBSCRIBE` is especially handy when you have a large result set that experiences local changes.
