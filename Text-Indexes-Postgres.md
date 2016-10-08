# Large Text Indexes in Postgres

An index on a sufficiently large `text` column can take up more space than the table itself. If you only need to check for equality, you can significantly reduce the size of the index.

At first glance, a hash index seem perfect for this. However, you shouldn’t use them since [they are not currently WAL-logged](https://www.postgresql.org/docs/current/static/indexes-types.html). Instead, use an expression index:

```sql
CREATE INDEX CONCURRENTLY ON table_name (CAST(md5(column_name) AS uuid));
```

Cast to a `uuid` since it’s 16 bytes - the [perfect size](http://dba.stackexchange.com/questions/115271/what-is-the-optimal-data-type-for-an-md5-field) to store an md5 hash.


Add an extra condition to your queries so the index is used.

```sql
SELECT * FROM table_name WHERE column_name = 'some_value'
  AND md5(column_name)::uuid = md5('some_value')::uuid; -- add this
```

Keep the original equality comparison in the unlikely chance of a hash collision.

Finally, confirm it worked:

```sql
EXPLAIN ANALYZE
  SELECT * FROM table_name WHERE column_name = 'some_value'
  AND md5(column_name)::uuid = md5('some_value')::uuid;
```

This should show the new index being used.

```
Index Scan using table_name_md5_idx on table_name  (cost=0.58..8.60 rows=1 width=172) (actual time=0.012..0.012 rows=0 loops=1)
  Index Cond: ((md5(column_name)::uuid = '9619030c-750b-4300-95b1-2d365196de91'::uuid)
  Filter: (column_name = 'some_value'::text)
Planning time: 0.062 ms
Execution time: 0.027 ms
```

If it’s not, run `ANALYZE VERBOSE table_name;` and try again.

This reduced the size of one of our indexes by **7x!** :slot_machine:
