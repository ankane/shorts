# PgBouncer Setup

In under 5 minutes

## Get Started

Here’s the flow:

```
Web app -> PgBouncer -> Postgres
```

You can install PgBouncer on the same server as Postgres or a separate server. For Amazon RDS, you won’t have shell access to the database server, so you’ll need to spin up another EC2 instance to run PgBouncer.

```
Web app -> EC2 running PgBouncer -> RDS instance
```

Start by launching a new instance of Ubuntu Server 16.04 LTS. Once the server is ready, ssh in. For the latest version of PgBouncer, we’ll use the [official Postgres APT repository](https://wiki.postgresql.org/wiki/Apt).

```sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
sudo apt-get update
sudo apt-get install pgbouncer
```

## Configure PgBouncer

Edit `/etc/pgbouncer/pgbouncer.ini`. The important settings are:

```ini
[databases]
YOUR-DBNAME = host=YOUR-DATABASE-URL port=5432 dbname=YOUR-DBNAME

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

## Start the Service

```sh
sudo service pgbouncer start
```

Then reboot the server and confirm the process comes back up.

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

## Congrats

You’ve successfully set up PgBouncer.
