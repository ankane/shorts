# Installing Presto for Mac

[Presto](https://prestodb.io/) is a “Distributed SQL Query Engine for Big Data”.

## Server

The easiest way to install Presto is with [Homebrew](http://brew.sh).

```sh
brew install presto
```

Next, add a connector. Here’s the list of [available ones](https://prestodb.io/docs/current/connector.html).

For PostgreSQL, create `/usr/local/Cellar/presto/0.150/libexec/etc/catalog/mydb.properties` with:

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

## Resources

- [List of clients](https://prestodb.io/resources.html)
- [Installation instructions](https://prestodb.io/docs/current/installation/deployment.html)
