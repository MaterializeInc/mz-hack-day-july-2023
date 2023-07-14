# Electives

We've put together an unordered collection of "electives": work that builds on what we've covered, and introduces new and more exploratory content.
These electives are available for you to choose from, in any order, as your appetite allows.
There are fewer "correct answers" here, but each should have the opportunity to learn and do something interesting.

1. [`WITH MUTUALLY RECURSIVE` (e.g. auction bidder cycles)](#recursive-queries)
1. [Ecosystem integrations (e.g. dbt, Metabase)](#ecosystem-integrations)
1. [Query optimization, performance diagnosis](#query-diagnosis-and-optimization)
1. [Build a game](#build-a-game).
1. Sources, sinks (?)
1. E-commerce stack

## Recursive Queries

Materialize has a unique `WITH MUTUALLY RECURSIVE` syntactic block that allows you the full power of recursive and iterative query constructions.
This section will get you experience using this construct, through a worked example with our auction data.

Folks running an auction site are plausibly understandably interested in fraud.
There are a lot of ways this can happen, but one of the ways folks have flagged is when you have *cycles* of transactions amount accounts.
For example, A bids on something B sells, B bids on something C sells, and C bids on something A sells.


---

**CHALLENGE**:
Our `bids` collection has a column `buyer` and our `auctions` collection has a column `seller`.
They are connected by the `bids.auction_id` and `auctions.id` columns.
Write a query that looks for cycles of accounts that bids on auctions by accounts that bid on auctions by accounts that ... on auctions by the original account.
It may help to start with a query that considers pairs of accounts, then triples, etc in order to see how the pattern generalizes.


<details>
<summary>Example challenge answers</summary>

To sniff out the potential cycles, we start with a (non-recursive) definition of a single link, and then repeatedly expand it.
```sql
WITH MUTUALLY RECURSIVE
    -- directed link between two accounts.
    link (source bigint, target bigint) AS (
        SELECT bids.id as source, auctions.seller as target
        FROM bids, auctions
        WHERE bids.auction_id = auctions.id
    ),
    -- directed chain between two accounts
    chain (source bigint, target bigint) AS (
        SELECT chain.source, link.target
        FROM chain, link
        WHERE chain.target = link.source
        UNION
        SELECT * FROM link
    )
-- those accounts that loop back to themselves.
SELECT source
FROM chain
WHERE chain.source = chain.target;
```
</details>


Can you extend the query to report the length of the cycle?
Can you extend the query to report the maximum amount of money that might flow along each cycle, defined as the least amount bid on each cycle.

You may find the `(RETURN AT RECURSION LIMIT n)` clause to help you debug your queries.

<details>
<summary>Example challenge answers</summary>

```sql
WITH MUTUALLY RECURSIVE
    -- directed link between two accounts, with bid amount.
    link (source bigint, target bigint, amount integer, hops integer) AS (
        SELECT bids.id as source, auctions.seller as target, amount, 1
        FROM bids, auctions
        WHERE bids.auction_id = auctions.id
    ),
    candidates (source bigint, target bigint, amount integer, hops integer) AS (
        SELECT
            chain.source,
            link.target,
            LEAST(chain.amount, link.amount) as amount,
            chain.hops + link.hops as hops
        FROM chain, link
        WHERE chain.target = link.source
        UNION ALL
        SELECT * FROM link
    ),
    -- directed chain between two accounts, with minimum bid and chain length.
    chain (source bigint, target bigint, amount integer, hops integer) AS (
        SELECT DISTINCT ON (source, target) source, target, amount, hops
        FROM candidates
        -- Ordeing by `hops` ascending prevents unbounded increase.
        ORDER BY source, target, amount DESC, hops ASC
    )
-- those accounts that loop back to themselves.
SELECT source, amount, hops
FROM chain
WHERE chain.source = chain.target;
```

</details>

Can you write a query from the output above that rediscovers the cycle from the evidence of the amount and hops?
Can you modify the query to leave breadcrumbs behind to make this discovery efficient and interactive?

---



## Ecosystem integrations

So far, you've been interacting with Materialize using a SQL shell. Is that all there is to it? — you might be asking yourself. Definitely not! To the outside world, Materialize presents as a PostgreSQL database (via `pgwire`), which means it integrates with many tools in the data ecosystem out-of-the-box.

### dbt

Materialize integrates with dbt through the [`dbt-materialize`](https://github.com/MaterializeInc/materialize/tree/main/misc/dbt-materialize) adapter. In the unlikely case you haven't heard about it: [dbt](https://www.getdbt.com/product/what-is-dbt/) is commonly used as a SQL transformation layer to define, run and manage data transformations against a database following software engineering best practices like version control, testing and CI/CD. Using dbt with Materialize comes with some _woo aaa_ moments, like not having to re-run your transformations on a schedule, or being able to continually test them.

One fun challenge is to convert the set of SQL transformations you've run so far in this Hack Day into a dbt project, and play around with it. Follow [this guide](https://materialize.com/docs/manage/dbt/) to get started!

### Metabase

You can also serve the results being maintained in Materialize to a BI tool, and build dashboards that don't break bank every time you refresh them. We suggest you [sign up for a free trial of Metabase](https://store.metabase.com/free-trial) and try to create some visualizations based on your auction data. Follow [this guide](https://materialize.com/docs/serve-results/metabase/) to get started!

## Query Diagnosis and Optimization

Materialize can represent the same workload multiple ways, with different performance trade-offs.
This section will get you experience framing the same task different ways, diagnosing their performance and costs, and choosing between alternatives.

Let's imagine that in our auction data we want to provide a service that, when requested, informs a user of all other bids that are currently outbidding them in some auction.
Let's start by preparing views that would determine this information.
```sql
-- Collect bids that are for currently active auctions
CREATE VIEW active_bids AS
SELECT
    bids.id,
    bids.buyer,
    bids.auction_id,
    bids.amount,
    bids.bid_time
FROM bids
JOIN auctions ON bids.auction_id = auctions.id
WHERE auctions.end_time > mz_now();
```

```sql
-- Bids that are beating an active bid.
CREATE VIEW out_bids AS
SELECT
    a1.id,
    a1.buyer,
    a1.auction_id,
    a1.amount,
    a1.bid_time,
    a2.buyer AS other_buyer,
    a2.amount AS other_amount
FROM active_bids a1, active_bids a2
WHERE a1.auction_id = a2.auction_id
  AND a1.amount < a2.amount
  AND a1.buyer != a2.buyer;
```
These views are stacked on top of each other, and queries from `out_bids` should contain the results we are looking for.

---
**CHALLENGE**: Create two new clusters `pull` and `push` and use each of them to try out direct access to the view (`SELECT` from it) and indexed access (`CREATE INDEX` on `out_bids`), when looking on behalf of a specific user (say `500`).
Compare the response latencies between the two, and also the memory use of each cluster (through the Console cluster dashboard).
Use `EXPLAIN` to validate your understanding of why the two approaches have different characteristics.

<details>
<summary>Example challenge answers</summary>

Let's start with a cluster that does not index the data, and just re-evaluates the query from scratch each time.
```sql
-- PULL: Re-evaluate from scratch.
SET CLUSTER = pull;
SELECT * FROM out_bids WHERE buyer = 500;
EXPLAIN SELECT * FROM out_bids WHERE buyer = 500;
```

Next, let's consider a cluster that indexes the results.
This should result in prompt response times, but should also use substantially more resources.
Check if you can see the memory use through the Console cluster dashboard.
```sql
-- PUSH: Maintain for all bids in all open auctions.
SET CLUSTER = push;
CREATE INDEX out_bids_by_buyer ON out_bids (buyer);
SELECT * FROM out_bids WHERE buyer = 500;
EXPLAIN SELECT * FROM out_bids WHERE buyer = 500;
```
</details>

Can you come up with a hybrid approach that maintains *some* information as it changes, less than the `push` cluster but more than `pull`, and still responds almost as promptly as the `push` cluster? Hint: call your new cluster `pushpull`.

<details>
<summary>Example challenge answers</summary>

This cluster contains two indexes that make the join execution efficient, without expanding the data as much as an index on all of `out_bids` would.

```sql
-- PUSHPULL: Maintain active_bids, evaluate out_bids.
SET CLUSTER = pushpull;
CREATE INDEX active_by_buyer ON active_bids (buyer);
CREATE INDEX active_by_auction ON active_bids (auction_id);
SELECT * FROM out_bids WHERE buyer = 500;
EXPLAIN SELECT * FROM out_bids WHERE buyer = 500;
```


</details>

---

## Build a game!

Materialize supports multiple interactive sessions, mutable tables, and the ability to frame application logic as views over those tables.
In principle, this is sufficient to build an interactive game you can play with your neighbor.

As an example, [this post](https://github.com/frankmcsherry/blog/blob/master/posts/2022-07-06.md) describes building Minesweeper logic in Materialize.
The underlying tables are guess locations, and the views derived report on the counts of mines in adjacent cells, enough to play the game as if using a 1980s BBS.

Alternately, Sudoku lends itself to concise SQL rules both to assess guesses, but also to provide hints for locations that must contain certain values. Can you take those prompts and wrap them in a `WITH MUTUALLY RECURSIVE` block to try and *solve* Sudoku problems using your rules?