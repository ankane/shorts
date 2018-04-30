# TPC-DS with Postgres

[TPC-DS](http://www.tpc.org/tpcds/) is a database benchmark.

```sh
git clone https://github.com/gregrahn/tpcds-kit.git
cd tpcds-kit/tools
make OS=MACOS
```

Create the database and load the schema

```sh
createdb tpcds
psql tpcds -f tpcds.sql
```

Generate data

```sh
./dsdgen -FORCE -VERBOSE
```

Load the data

```sh
for i in `ls *.dat`; do
  table=${i/.dat/}
  echo "Loading $table..."
  sed 's/|$//' $i > /tmp/$i
  psql tpcds -q -c "TRUNCATE $table"
  psql tpcds -c "\\copy $table FROM '/tmp/$i' CSV DELIMITER '|';"
done
```

Generate queries

```sh
./dsqgen -DIRECTORY ../query_templates -INPUT ../query_templates/templates.lst \
  -VERBOSE Y -QUALIFY Y -DIALECT netezza
```

Run queries

```sh
psql tpcds -c "ANALYZE VERBOSE"
psql tpcds < query_0.sql
```

## Bonus: Add Indexes with Dexter

Install the latest version of [Dexter](https://github.com/ankane/dexter)

```sh
gem install specific_install
gem specific_install https://github.com/ankane/dexter.git
```

And run

```sh
for i in `seq 1 10`; do
  dexter tpcds query_0.sql --input-format sql --create
done
```
