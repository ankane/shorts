# PgBouncer for Amazon RDS

:fire: In under 5 minutes

## Get Started

RDS does not give you shell access to database servers, so you need to spin up another EC2 instance to run PgBouncer.

Here’s the flow:

```
Web app -> EC2 running PgBouncer -> RDS instance
```

Start by launching a new instance of Ubuntu Server 14.04 LTS. Once the server is ready, ssh in and run:

```sh
sudo apt-get install pgbouncer
```

## Configure PgBouncer

Edit `/etc/pgbouncer/pgbouncer.ini`. The important settings are:

```ini
[databases]
YOUR-DBNAME = host=YOUR-RDS-URL.rds.amazonaws.com port=5432 dbname=YOUR-DBNAME

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

If you need more than 1,000 connections to PgBouncer, you’ll need to increase file limits.

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

Load PgBouncer on startup.

Append to `/etc/default/pgbouncer`:

```
START=1
```

## Start Services

```sh
su postgres -c 'service pgbouncer start'
```

## Test

```sh
psql -h 127.0.0.1 -p 6432 -d YOUR-DBNAME -U USERNAME1
```

## App Changes

Be sure to disable prepared statements, as they will not work with PgBouncer in transaction mode.

## Statement Timeouts

To use a [statement timeout](http://www.postgresql.org/docs/9.4/static/runtime-config-client.html#GUC-STATEMENT-TIMEOUT), run:

```sql
ALTER ROLE USERNAME1 SET statement_timeout = 5000;
```

## TODO

- shell script to do all of this
