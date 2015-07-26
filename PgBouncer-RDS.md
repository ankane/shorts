# PgBouncer for Amazon RDS

:fire: In under 5 minutes

## Get Started

RDS does not give you shell access to database servers, so you need to spin up another EC2 instance to run PgBouncer. [Stunnel](http://en.wikipedia.org/wiki/Stunnel) is used to provide a secure tunnel to your RDS instance.

Here’s the flow:

```
Web app -> EC2 running PgBouncer and Stunnel -> RDS instance
```

More specifically:

```
Web app -> PgBouncer on port 6432 -> Stunnel on port 5432 -> RDS instance
```

Start by launching a new instance of Ubuntu Server 14.04 LTS. Once the server is ready, ssh in and run:

```sh
sudo apt-get install stunnel4
sudo apt-get install pgbouncer
```

## Configure Stunnel

Replace `/etc/stunnel/stunnel.conf` with:

```ini
client = yes

[rds-postgres]
protocol = pgsql
accept = 127.0.0.1:5432
connect = YOUR-RDS-URL.rds.amazonaws.com:5432
options = NO_TICKET
retry = yes
```

## Configure PgBouncer

Edit `/etc/pgbouncer/pgbouncer.ini`. The important settings are:

```ini
[databases]
YOUR-DBNAME = host=127.0.0.1 port=5432 dbname=YOUR-DBNAME

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction
server_reset_query =
```

[View all settings](http://pgbouncer.projects.pgfoundry.org/doc/config.html)

Create `/etc/pgbouncer/userlist.txt` with:

```
"USERNAME1" "PASSWORD1"
"USERNAME2" "PASSWORD2"
```

Use the same credentials as your database server.

## Increase File Limits

Append to `/etc/security/limits.conf`:

```
postgres         soft    nofile          4096
postgres         hard    nofile          10240
```

Append to both `/etc/pam.d/common-session` and `/etc/pam.d/common-session-noninteractive`:

```
session required pam_limits.so
```

## Startup

Load Stunnel and PgBouncer on startup.

Append to `/etc/default/stunnel4`:

```
ENABLED=1
```

Append to `/etc/default/pgbouncer`:

```
START=1
```

## Start Services

```sh
service stunnel4 start
su postgres -c 'service pgbouncer start'
```

## Test

```sh
psql -h 127.0.0.1 -p 6432 -d YOUR-DBNAME -U USERNAME1
```

Next, ensure your connection is secure. In the `psql` console, run:

```sql
CREATE EXTENSION IF NOT EXISTS sslinfo;
SELECT ssl_is_used();
```

To connect from another server, use:

```sh
psql -h YOUR-EC2-HOSTNAME -p 6432 -d YOUR-DBNAME -U USERNAME1
```

## Statement Timeouts

To use a [statement timeout](http://www.postgresql.org/docs/9.4/static/runtime-config-client.html#GUC-STATEMENT-TIMEOUT), run:

```sql
ALTER ROLE USERNAME1 SET statement_timeout = 5000;
```

## TODO

- shell script to do all of this

## Thanks

Made possible thanks to:

- [Connecting to Postgres with STunnel](https://helveticode.com/2014/01/11/rds-postgresql-with-stunnel/)
