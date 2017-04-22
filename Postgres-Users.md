# Three Types of Postgres Users, and How to Create Them

Setting up database users for an app can be challenging if you don’t do it often. Good permissions add a layer of security and can minimize the chances of developer mistakes. Following the [principle of least privilege](https://en.wikipedia.org/wiki/Principle_of_least_privilege), users should have the privileges they need and nothing more.

The three types we’ll cover are:

Type | Read | Write | Modify
--- | --- | --- | ---
migrations | ✓ | ✓ | ✓
app | ✓ | ✓ |
analytics | ✓ | |

Translating to SQL terms:

- read is `SELECT`
- write is `INSERT`, `UPDATE`, and `DELETE`
- modify is schema changes, like `CREATE TABLE` and `DROP TABLE`

Before we jump into it, there’s something you should know about new databases.

## New Databases

After creating a new database, all users can access it and create tables in the `public` schema. This isn’t what we want. To fix this, run:

```sql
REVOKE ALL ON DATABASE mydb FROM public;

REVOKE ALL ON SCHEMA public FROM public;
```

Be sure to replace `mydb` with your database name.

## Roles

PostgreSQL uses the concept of *roles* to manage permissions. From [the docs](https://www.postgresql.org/docs/current/static/user-manag.html):

> A role can be thought of as either a database user, or a group of database users

A user is simply a role with a password and permission to log in.

The approach we’ll take is to create a group and add users to it. This makes it easy to rotate credentials in the future: just add a second user to the group, update your apps, and then remove the original user.

## Migrations

First, we need a group to manage the schema. You could use a superuser, but this isn’t a great idea, as superusers can access all databases, change permissions, and create new roles.

```sql
CREATE ROLE migrations;

GRANT CONNECT ON DATABASE mydb TO migrations;

GRANT ALL ON SCHEMA public TO migrations;

ALTER ROLE migrations SET lock_timeout TO '5s';
```

We also set a lock timeout so migrations don’t disrupt normal database activity while attempting to acquire a lock.

Create a user with:

```sql
CREATE ROLE migrator WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE migrations;
```

You can generate a nice password with:

```sh
echo $(cat /dev/urandom | LC_CTYPE=C tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1)
```

## App

Next, let’s create a group for our app. It shouldn’t need to modify the schema or truncate tables. We also want to set a statement timeout to prevent long running queries from degrading database performance.

```sql
CREATE ROLE app;

GRANT CONNECT ON DATABASE mydb TO app;

GRANT USAGE ON SCHEMA public TO app;

GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app;

GRANT SELECT, USAGE ON ALL SEQUENCES IN SCHEMA public TO app;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT, USAGE ON SEQUENCES TO app;

ALTER ROLE app SET statement_timeout TO '30s';
```

**Note:** The default privileges statements reference the group used for migrations.

Then, create a user with:

```sql
CREATE ROLE myapp WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE app;
```

## Analytics

Finally, let’s create a group to be used for data analysis, reporting, and business intelligence (BI) tools. These users are often referred to as a *read-only users*.

```sql
CREATE ROLE analytics;

GRANT CONNECT ON DATABASE mydb TO analytics;

GRANT USAGE ON SCHEMA public TO analytics;

GRANT SELECT ON ALL TABLES IN SCHEMA public TO analytics;

ALTER DEFAULT PRIVILEGES FOR ROLE migrations IN SCHEMA public
    GRANT SELECT ON TABLES TO analytics;

ALTER ROLE analytics SET statement_timeout TO '3min';
```

Once again, creating a user is relatively straightforward.

```sql
CREATE ROLE bi WITH LOGIN ENCRYPTED PASSWORD 'secret' IN ROLE analytics;
```

This should give you a nice foundation.
