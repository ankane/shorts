# TPC-H with Postgres

[TPC-H](http://www.tpc.org/tpch/) is a database benchmark.

```sh
git clone https://github.com/gregrahn/tpch-kit.git
cd tpch-kit/dbgen
make -f Makefile.osx
```

Create the database and load the schema

```sh
createdb tpch
psql tpch -f dss.ddl
```

Generate data

```sh
./dbgen -vf -s 1
```

Load the data

```sh
for i in `ls *.tbl`; do
  table=${i/.tbl/}
  echo "Loading $table..."
  sed 's/|$//' $i > /tmp/$i
  psql tpch -q -c "TRUNCATE $table"
  psql tpch -c "\\copy $table FROM '/tmp/$i' CSV DELIMITER '|'"
done
```

Generate queries

```sh
mkdir /tmp/queries
for i in `ls queries/*.sql`; do
  tail -r $i | sed '2s/;//' | tail -r > /tmp/$i
done

DSS_QUERY=/tmp/queries ./qgen | sed 's/limit -1//' | sed 's/day (3)/day/' > queries.sql
```

Run queries

```sh
psql tpch -c "ANALYZE VERBOSE"
psql tpch < queries.sql
```

## Bonus: Add Indexes with Dexter

Install [Dexter](https://github.com/ankane/dexter)

```sh
gem install pgdexter
```

And run

```sh
for i in `seq 1 5`; do
  dexter tpch queries.sql --input-format sql --create
done
```
