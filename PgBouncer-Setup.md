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

Start by launching a new instance of Ubuntu Server 16.04 LTS. Be sure to open inbound TCP port 6432 in the security group associated with your instance, and confirm that it's reachable from any IPs you'll be using with it.

(You might also want to lock down the security group inbound TCP rule that governs your RDS instance, so that it can accept incoming postgres connections only from the IP that the pgbouncer ec2 instance is on, but be sure that you've reserved an IP for this instance, first.)

Once the server is ready, ssh in. For the latest version of PgBouncer, we’ll use the [official Postgres APT repository](https://wiki.postgresql.org/wiki/Apt).

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
YOUR-DBNAME = host=YOUR-HOST port=5432 dbname=YOUR-DBNAME auth_user=YOUR-DB-USERNAME

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
"USERNAME1" "md5PASSWORD1"
"USERNAME2" "md5PASSWORD2"
```

Use the same credentials as your database server, but instead of the password you'll need the md5 hash of the password. The md5 password hash is just the md5sum of `password + username` with the string `md5` prepended. For instance, to derive the hash for user `username1` with password `password`, you can do:

```bash
# ubuntu, debian, etc.
echo -n "passwordusername1" | md5sum
# macOS, openBSD, etc.
md5 -s "passwordusername1"
```

If the above commands give you an output string like `d75bb2be2d7086c6148944261a00f605`, then the `/etc/pgbouncer/userlist.txt` file would look as follows:

```
"username1" "md5d75bb2be2d7086c6148944261a00f605"
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

## Increase File Limits

If you need more than 1,000 connections to PgBouncer, you’ll need to increase file limits.

Append to `/etc/default/pgbouncer`:

```sh
ulimit -n 16384
```

Restart the service with:

```sh
sudo service pgbouncer restart
```

To confirm it worked, find the process ID and run:

```sh
cat /proc/<pid>/limits
```

`Max open files` should reflect the value above.

## App Changes

Be sure to disable prepared statements, as they will not work with PgBouncer in transaction mode.

## Statement Timeouts

To use a [statement timeout](http://www.postgresql.org/docs/9.4/static/runtime-config-client.html#GUC-STATEMENT-TIMEOUT), run:

```sql
ALTER ROLE USERNAME1 SET statement_timeout = 5000;
```

## Congrats

You’ve successfully set up PgBouncer.
