# Adventures and Lessons in Postgres [Draft]

At Instacart, we depend heavily on Postgres. As our database cluster has grown, we’ve learned a number of important lessons along the way.

This post aims to share our knowledge so you don’t have to learn the hard way. Most of this is packaged into [PgHero](https://github.com/ankane/pghero), our open-source performance dashboard for Postgres.

Here’s the journey behind it.

## Queries

One of the earliest issues we hit was spikes in CPU usage. The problem was, we didn’t know which queries were causing it. Luckily, Postgres has an extension called [pg_stat_statements](https://www.postgresql.org/docs/current/static/pgstatstatements.html) to track the most time-consuming queries. You can read how to set it up [here](https://github.com/ankane/pghero/blob/master/guides/Query-Stats.md).

To successfully correlate queries with high CPU usage, we needed to track them over time. We created a table and set up a cron job to record them every 5 minutes.

Once we identified queries, we had to decide what to do with them. Some were a result a of missing indexes. Others were a result of many small queries. For these, we fixed [N+1 queries](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/) and cached others.

### Timeouts

In summer 2015, our database ran out of space. The space was recovered when we rebooted, leading us to believe it was temp space.

Later the same day, we caught a query that looked like this:

```
`SELECT * FROM large_table INNER JOIN other_large_table ...`
```

Typically, this query had a `WHERE` condition, but it was missing when a certain code path was taken.

*A single bad query brought our database to its knees.*

To prevent this, set aggressive timeouts on non-superusers. This way, inefficient queries fail and have limited impact. You can set statement timeouts with:

```
`ALTER ROLE myuser SET statement_timeout = '5s';`
```

### Scaling Out

As traffic continued to grow, we needed more CPU than we could get with a single instance, so we decided to add a replica. We started off by selectively moving reads.

Replicas can lag, so if you insert a record and immediately try to read it from the replica, it may or may not be there. Your app needs to handle this situation. One approach is stick to the primary for a set period after writes. This doesn’t guarantee the data will be there if replica lag is high, but can reduce issues.

One error you might see is:

> ERROR: canceling statement due to conflict with recovery

You can set `hot_standby_feedback = on` on the replica to fix this.

### Planning

Sometimes, a query will suddenly start taking much longer than previously. This is often due to planner statistics getting out of date. You can use the `ANALYZE` function to update the statistics.

```
`ANALYZE VERBOSE <table>;`
```

Postgres regularly auto analyzes on its own, so this issue comes up infrequently. It’s still a good idea to run it after adding new indexes, as well as writes that affect significant portions of a table.

## Connections

As your app grows in popularity, you’ll likely need to spin up more web servers and background workers to handle the load. This will increase the number of connections to the database.
In winter 2015, our performance tanked. It happened on a Thursday, and didn’t get better on Friday. Or Saturday. Or Sunday. We had to restart the database each day to keep it going. We eventually found we had 1500 connections. Each connection takes ~10 MB memory, so 1500 required 15 GB. Even for the largest Postgres instances, it’s recommended to keep connections under 500.

Luckily, we weren’t the first people to run into this issue. Marko Kreen created [PgBouncer](https://pgbouncer.github.io/) to solve it. PgBouncer sits in front of your database. It takes advantage of the fact that most of the time, connections are idle. PgBouncer pools these together so Postgres connections can be shared. You can have thousands of connections to the bouncer sharing tens of connections to Postgres.

At 5 am Monday morning, we deployed our first bouncer. To make sure we didn’t run into the issue again, we set `max_connections = 500` in our Postgres configuration.

Instructions to install PgBouncer are [here](https://github.com/ankane/shorts/blob/master/PgBouncer-Setup.md). Statements can queue on the bouncer when all connections are active, so be sure to monitor the `cl_waiting` metric.

## Space

As your data grows, space can begin to become a concern. More space = more money. We also use Amazon RDS, which has a 6 TB limit. There are often a few quick wins you can get with space.

First, remove indexes you don’t need. Postgres keeps track of index usage makes this data available in the `pg_stat_user_indexes` view. If you have replicas, check all the instances to make sure an index isn’t used before removing.

Next, look for duplicate indexes. If you have a multicolumn index on `store_id, created_at`, you don’t need a separate index on `store_id`. Postgres can use the first index for any queries that use `store_id`.

Once these are addressed, look at your largest tables and indexes. Two good options are:

1. Move tables to another database / data store
2. Archive data

It’s best to decide this for each table. We’ve done both over the years.

### Moving Out

As with most apps, we started with a single database. Next, we moved our product data to a separate database. Since then, we’ve split into many more databases based on business domain. The biggest drawback to this approach is you lose the ability to join across tables that are now in different DBs. You can typically do this in your data warehouse for analytics, but need to rewrite some application code for transactional queries.

We created [pgsync](https://github.com/ankane/pgsync) to make this easier.

You can also move tables to another data store. We recently started using [Cassandra](http://cassandra.apache.org/).

### Archiving

Another option is to archive older data. Specifically, we want to backup and drop older data from the database.
A good strategy for doing this is partitioning. With partitioning, a table is split into smaller physical tables. The advantage of this is you can easily drop a partition and reclaim space, which you can’t do if you just delete the data. There are different ways to partition, but we’re interested time-based partitioning.

Let’s say we have an `audits` table. We can create partitions for each month, named `audits_201706`, `audits_201707`, etc. Audits in the same month will be written to the same partition. You can still query all audits at once through the `audits` table. If you want to keep three months of audits, set up a cron job to archive partitions older than three months.

We created [pgslice](https://github.com/ankane/pgslice) to make this easier.

Another option is to shard your data. You can do this manually or with a product like [Citus](https://www.citusdata.com/), but we won’t cover this here.

### Large Indexes

For large indexes, recreating them can reclaim space if they’ve accumulated a lot of bloat. Just add a duplicate index, then remove the original one.

Another option is to use [partial indexes](https://www.postgresql.org/docs/current/static/indexes-partial.html). Partial indexes only index a subset of a table, so they’re smaller.

If you have large text indexes and you only need equality matching, you can use a [custom hash index](https://medium.com/@ankane/large-text-indexes-in-postgres-5d7f1677f89f). Overall, don’t worry too much about indexes unless they are huge.

## Pitfalls

### Transaction ID Wraparound

In summer 2015, we encountered a very scary message.

> WARNING: database "instacart" must be vacuumed within 8914761 transactions
HINT: To avoid a database shutdown, execute a database-wide VACUUM in "instacart".

Our database is going to shutdown?!

With the way Postgres works internally, each table must be vacuumed every two billion transactions. If this doesn’t happen, the database will shutdown with one million transactions to go to protect itself from data loss. You can read a [detailed description here](https://www.postgresql.org/docs/current/static/routine-vacuuming.html#VACUUM-FOR-WRAPAROUND).

Most of the time, autovacuum will handle this for you. In our case, a corrupted index prevented one table from being vacuumed. The solution was to add a new index, then drop the existing one.

While we haven’t come as close to the brink since, we have seen situations where autovacuum lags behind after large updates. No manual action has been required yet, but it’s definitely something to keep an eye on.

### Integer Overflow

One day, we started seeing errors on one of our heavily written tables.

> ERROR: integer out of range

We use integer primary keys for most tables, and it turns out we exceeded the max value for integers (~2.1 billion). We could not longer write to the table until we upgraded the column to a `bigint`. Since the table was unusable, we were able to upgrade in place.

However, as other tables approached the limit, we didn’t want to incur a large period of downtime to make the switch to `bigint`. In these cases, the solution was to add a new `bigint` column to the table, backfill it, and swap it. This process must be repeated for columns that reference the table as well.

For new tables, use `bigint` for primary keys from the start. As they say:

> Friends don’t let friends use INT as a primary key. — [@schneems](https://twitter.com/schneems/status/731167572096253952)

## Schema Migrations

Changing the schema can be one of the riskier things we do as developers. There are a number of operations that can cause downtime. Common ones with Postgres are:

* adding an index non-concurrently
* adding a column with a non-null default value to an existing table
* changing the type of a column

Sometimes people forgot the rules, or a new hire never saw them, so we decided to enforce them with code. We created a project called [Strong Migrations](https://github.com/ankane/strong_migrations), which virtually eliminated the issue.

Another thing to be aware of is locking. If a statement like `ALTER TABLE` can’t acquire a lock in a timely manner, other statements get stuck behind it. To prevent this, it’s a good idea to set a lock timeout for the database user that runs migrations.

```
`ALTER ROLE myuser SET lock_timeout = '10s';`
```

## Tuning

Lastly, Postgres has a number of settings you can tune. [PostgreSQL When It’s Not Your Job](http://thebuild.com/presentations/not-your-job.pdf) has great recommendations.

## Conclusion

Our journey with Postgres has been a great one, even with the mistakes we’ve made. To summarize:

1. Use pg_stat_statements to identify resource-intensive queries
2. Set aggressive statement timeouts on all non-superusers
3. Use replicas to scale reads
4. Use PgBouncer if you have over 500 connections
5. Remove duplicate indexes and unused indexes to save space
6. Use partitioning to archive data
7. Watch out for transaction ID wraparound and integer overflow
8. Perform schema migrations in ways that don’t cause downtime
9. Spend a little bit of time to tune your settings

Hopefully some of this knowledge allows you to spend less time on database management and more time on building something great.
