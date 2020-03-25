# Installing Presto for Mac

[Presto](https://prestodb.io/) is a “Distributed SQL Query Engine for Big Data” that gives you the ability to join across data stores! :tada:

## Server

The easiest way to install Presto is with [Homebrew](https://brew.sh).

```sh
brew install presto
```

Next, add a connector. Here’s the list of [available ones](https://prestodb.io/docs/current/connector.html).

For PostgreSQL, create `/usr/local/opt/presto/libexec/etc/catalog/mydb.properties` with:

```ini
connector.name=postgresql
connection-url=jdbc:postgresql://localhost:5432/mydbname
connection-user=myuser
connection-password=mysecret
```

And start the server with:

```sh
presto-server run
```

## Client

Presto comes with a CLI

```sh
presto --catalog mydb --schema public
```

And run:

```sql
SHOW TABLES;
```

Try one of your tables with:

```sql
SELECT * FROM mytable;
```

There are also clients in [many different languages](https://prestodb.io/resources.html#libraries) you can use.

:rabbit: :tophat: :sparkles:
