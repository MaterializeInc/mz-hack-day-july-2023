# Electives

## Recursive Queries

Materialize has a unique `WITH MUTUALLY RECURSIVE` syntactic block that allows you the full power of recursive and iterative query constructions.
Recursive queries can sound intimidating, so we'll build up an example using our auction data.

Folks running an auction site are plausibly understandably interested in fraud. 
There are a lot of ways this can happen, but one of the ways folks have flagged is when you have *cycles* of transactions amount accounts. 
For example, A bids on something B sells, B bids on something C sells, and C bids on something A sells.


---

**CHALLENGE**: 
Our `bids` collection has a column `buyer` and our `auctions` collection has a column `seller`.
They are connected by the `bids.auction_id` and `auctions.id` columns.
Write a query that looks for cycles of accounts that bids on auctions by accounts that bid on auctions by accounts that ... on auctions by the original account.

---


## Integration with BI tools

## Integration with `dbt`

## Query Diagnosis and Optimization