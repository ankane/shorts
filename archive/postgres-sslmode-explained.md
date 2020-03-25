# Postgres SSLMODE Explained

When you connect to a database, Postgres uses the `sslmode` parameter to determine the security of the connection. There are many options, so here’s an analogy to web security:

- `disable` is HTTP
- `verify-full` is HTTPS

All the other options fall somewhere in between, and by design, make less guarantees of security than HTTPS in your browser does.

<p style="text-align: center;"><img src="/images/sslmode-screenshot.png" alt="Screenshot" /></p>

This includes the default `prefer`. The [Postgres docs](https://www.postgresql.org/docs/current/libpq-ssl.html) have a great table explaining this:

<p style="text-align: center;"><img src="/images/sslmode-table.png" alt="Table" /></p>

Other modes like `require` are still useful in protecting against passive attacks (sniffing), but are vulnerable to active attacks that can compromise your credentials. Tarjei Husøy created [postgres-mitm](https://thusoy.com/2016/mitming-postgres) to demonstrate this.

## Defense

The best way to protect a database is to limit inbound traffic. Require a VPN or SSH tunneling through a [bastion host](https://medium.com/@bill_73959/understanding-bastions-hosts-6ccd457e41ac) to connect. This ensures connections are always secure, and even if database credentials are compromised, an attacker won’t be able to access the database.

If this is not feasible, always use `verify-full`. This includes from code, psql, SQL clients, and other tools like [pgsync](https://github.com/ankane/pgsync) and [pgslice](https://github.com/ankane/pgslice).

You can specify `sslmode` in the connection URI:

```text
postgresql://user:pass@host/dbname?sslmode=verify-full&sslrootcert=ca.pem
```

Or use environment variables.

```sh
PGSSLMODE=verify-full PGSSLROOTCERT=ca.pem
```

Libraries for most programming languages have options as well.

```ruby
PG.connect(sslmode: "verify-full", sslrootcert: "ca.pem")
```

## Certificates

To verify an SSL/TLS certificate, the client checks it against a root certificate. Your browser ships with root certificates to verify HTTPS websites. Postgres doesn’t come with any root certificates, so to use `verify-full`, you must specify one.

Here are root certificates for a number of providers:

Provider | Certificate | Docs
--- | --- | ---
Amazon RDS | [Download](https://s3.amazonaws.com/rds-downloads/rds-ca-2019-root.pem) | [View](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#PostgreSQL.Concepts.General.SSL)
Google Cloud SQL | In Account | [View](https://cloud.google.com/sql/docs/postgres/connect-admin-ip)
Digital Ocean | In Account | [View](https://www.digitalocean.com/docs/databases/how-to/clusters/secure-clusters/)
Citus Data | [Download](https://console.citusdata.com/citus.crt) | [View](https://docs.citusdata.com/en/v8.0/cloud/security.html)

There’s no way to use `verify-full` with Heroku Postgres, so use caution when connecting from networks you don't fully trust. Instead of `heroku pg:psql`, use:

```sh
heroku run psql \$DATABASE_URL
```

This securely connects to a dyno before connecting to the database.

If you use PgBouncer, [set up secure connections](https://ankane.org/securing-pgbouncer-amazon-rds) for it as well.

## Conclusion

Hopefully this helps you understand connection security a bit better.

<div style="margin-top: 2rem;"></div>

Updates

- August 2019: Added Digital Ocean
