# Explore a Kafka data source

Materialize allows you to work with streaming data from multiple external data sources, like [Kafka](https://materialize.com/docs/sql/create-source/kafka/) and [PostgreSQL](https://materialize.com/docs/sql/create-source/postgres/). To get you started with a more realistic source, we set up a Kafka cluster with some sample data that you can use to explore and build.

<p align="center">
<img width="650" alt="demo_overview" src="https://github.com/MaterializeInc/mz-hack-day-july-2023/assets/23521087/3c1e80cf-66c8-40be-bd84-52b925499dd2">
</p>

## How to connect

Grab the credentials for the Hack Day Kafka broker from [1Password](https://share.1password.com/s#7Nj-rs0i59KwJQV47kSGQRAKW35dsbOINMuGyswEypw). We've conveniently included the DDL required to create the source in Materialize and start ingesting data, but you can also choose to use the source creation UI in the [web console](https://console.materialize.com/).

### Creating sources in Materialize

1. Now that you have the details to connect, you can create a source for each Kafka topic you'd like to use.

```sql
-- Topic: mysql.shop.purchases
CREATE SOURCE purchases
  FROM KAFKA CONNECTION confluent_cloud (TOPIC 'mysql.shop.purchases')
  FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY CONNECTION schema_registry
  ENVELOPE DEBEZIUM
  WITH (SIZE = '3xsmall');
```

```sql
-- Topic: mysql.shop.items
CREATE SOURCE items
  FROM KAFKA CONNECTION confluent_cloud (TOPIC 'mysql.shop.items')
  FORMAT AVRO USING CONFLUENT SCHEMA REGISTRY CONNECTION schema_registry
  ENVELOPE DEBEZIUM
  WITH (SIZE = '3xsmall');
```

2. To list the sources you just created:

```sql
SHOW SOURCES;
```

There are two additional topics at your disposal: `mysql.shop.users` and `pageviews` .

## Transforming data using SQL

With data being ingested into Materialize, you can start building SQL statements to transform it into something actionable. Following are some sample queries you can try, in case you need a jumpstart! We encourage you not only to reach for your favourite SQL patterns, but also to intentionally try to break things!

### Sample workflow

1. Create a [view](https://materialize.com/docs/get-started/key-concepts/#non-materialized-views) `item_purchases` that calculates rolling aggregations for each item in the inventory:

	```sql
	CREATE VIEW item_purchases AS
	      SELECT
	        item_id,
	        SUM(quantity) AS items_sold,
	        SUM(purchase_price) AS revenue,
	        COUNT(id) AS orders,
	        MAX(created_at::timestamp) AS latest_order
	      FROM purchases
	      GROUP BY item_id;
	```

2. To keep the results of this view incrementally maintained **in memory**, you can create an [index](https://materialize.com/docs/get-started/key-concepts/#indexes):

	```sql
	CREATE INDEX item_purchases_idx ON item_purchases (item_id);
	```

3. To see the results:

	```sql
	SELECT * FROM item_purchases LIMIT 10;

    --Lookups will be fast because of the index on item_id!
    SELECT * FROM item_purchases WHERE item_id = 768;
	```

4.  Create a [materialized view](https://materialize.com/docs/get-started/key-concepts/#materialized-views) `item_summary` that joins `item_purchases` and `items` based on the item identifier:

	```sql
	CREATE MATERIALIZED VIEW item_summary AS
	    SELECT
	        ip.item_id AS item_id,
	        i.name AS item_name,
	        i.category AS item_category,
	        SUM(ip.items_sold) AS items_sold,
	        SUM(ip.revenue) AS revenue,
	        SUM(ip.orders) AS orders,
	        ip.latest_order AS latest_order
	     FROM item_purchases ip
	     JOIN items i ON ip.item_id = i.id
	     GROUP BY item_id, item_name, item_category, latest_order;
	```

5. To see the results:

	```sql
	SELECT * FROM item_summary ORDER BY revenue DESC LIMIT 5;
	```

6. Create a view `item_summary_5min` that keeps track of any items that had orders in the past 5 minutes using a [temporal filter](https://materialize.com/docs/transform-data/patterns/temporal-filters/):

	```sql
	CREATE VIEW item_summary_5min AS
    	SELECT *
    	FROM item_summary
    	WHERE mz_now() >= latest_order
    	AND mz_now() < latest_order + INTERVAL '5' MINUTE;
	```
