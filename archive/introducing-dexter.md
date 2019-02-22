# Introducing Dexter, the Automatic Indexer for Postgres

<p style="text-align: center;"><img src="/images/dexter.jpg" alt="Dexter" /></p>

Your database knows which queries are running. It also has a pretty good idea of which indexes are best for a given query. And since indexes don’t change the results of a query, they’re really just a performance optimization. So why do we always need a human to choose them?

Introducing [Dexter](https://github.com/ankane/dexter). Dexter indexes your database for you. You can still do it yourself, but Dexter will do a pretty good job.

Dexter works in two phases:

1. Collect queries
2. Generate indexes

We’ll walk through each of them.

### Phase 1: Collect

You can stream Postgres log files directly to Dexter. Dexter finds lines like:

```txt
LOG:  duration: 14.077 ms  statement: SELECT * FROM ratings WHERE user_id = 3;
```

And parses out the query and duration. It uses fingerprinting to group queries. Queries with the same parse tree but different values are grouped together. For instance, both of the following queries have the same fingerprint.

```sql
SELECT * FROM ratings WHERE user_id = 2;
SELECT * FROM ratings WHERE user_id = 3;
```

The data is aggregated to get the total execution time by fingerprint. You can get similar information from the [pg_stat_statements view](https://www.postgresql.org/docs/current/static/pgstatstatements.html), except queries in the view are normalized. This means, you get:

```sql
SELECT * FROM ratings WHERE user_id = ?;
```

instead of

```sql
SELECT * FROM ratings WHERE user_id = 3;
```

However, we need the actual values to determine costs in the next step. To prevent over-indexing, you can set a threshold for the total execution time before a query is considered for indexing.

### Phase 2. Generate

To generate indexes, Dexter creates hypothetical indexes to try to speed up the slow queries we’ve just collected. Hypothetical indexes show how a query’s execution plan would change if an actual index existed. They take virtually no time to create, don’t require any disk space, and are only visible to the current session. You can read more about [hypothetical indexes here](https://rjuju.github.io/postgresql/2015/07/02/how-about-hypothetical-indexes.html).

The main steps Dexter takes are:

1.  Filter out queries on system tables and other databases
2.  Analyze tables for up-to-date planner statistics if they haven’t been analyzed recently
3.  Get the initial cost of queries
4.  Create hypothetical indexes on columns that aren’t already indexes
5.  Get costs again and see if any hypothetical indexes were used

While fairly straightforward, this approach is extremely powerful, as it uses the Postgres query planner to figure out the best index(es) for a query. Hypothetical indexes that were used AND significantly reduced cost are selected to be indexes.

To be safe, indexes are only logged by default. This allows you to use Dexter for index suggestions if you want to manually verify them first. When you let Dexter create indexes, they’re created concurrently to limit the impact on database performance.

```txt
2017-06-25T17:52:22+00:00 Index found: ratings (user_id)
2017-06-25T17:52:22+00:00 Creating index: CREATE INDEX CONCURRENTLY ON ratings (user_id)
2017-06-25T17:52:37+00:00 Index created: 15243 ms
```

### Trade-offs and Limitations

The big advantage of indexes is faster data retrieval. On the flip side, indexes add overhead to write operations, like INSERT, UPDATE, and DELETE, as indexes must be updated as well. Indexes also take up disk space.

Because of this, you may not want to index write-heavy tables. Dexter does not currently try to identify these tables automatically, but you can pass them in by hand.

As for other limitations, Dexter does not try to create multicolumn indexes (edit: this is no longer the case). Dexter also assumes the search_path for queries is the same as the user running Dexter. You’ll still need to create unique constraints on your own. Dexter also requires the [HypoPG](https://github.com/HypoPG/hypopg) extension, which isn’t available on some hosted providers like Heroku and Amazon RDS.

* * *

It’s time to make forgotten indexes a problem of the past.

[Add Dexter to your team](https://github.com/ankane/dexter) today.

### Thanks

This software wouldn’t be possible without [HypoPG](https://github.com/HypoPG/hypopg), which allows you to create hypothetical indexes, and [pg_query](https://github.com/lfittl/pg_query), which allows you to parse and fingerprint queries. A big thanks to Dalibo and [Lukas Fittl](https://medium.com/@LukasFittl) respectively.
