# Securing Database Traffic with PgBouncer and Amazon RDS

Securing database traffic inside your network can be a great step for defense in depth. It’s also a necessity for [Zero Trust Networks](https://www.amazon.com/Zero-Trust-Networks-Building-Untrusted/dp/1491962194).

Both Amazon RDS and PgBouncer have built-in support for TLS, but it’s a little bit of work to get it set up. This tutorial will show you how.

## Direct Connections

The first step is to make sure all direct connections are secure. Luckily, Amazon RDS has a parameter named `rds.force_ssl` for this. Once it’s applied, you’ll see an error if you try to connect without TLS. You can test this out with:

```sh
psql "postgresql://user:secret@dbhost:5432/ssltest?sslmode=disable
```

You’ll see an error like `FATAL:  no pg_hba.conf entry ... SSL off` if everything is configured correctly.

There are a number of possible values for `sslmode`, which you can [read about here](https://www.postgresql.org/docs/current/static/libpq-ssl.html). The most secure (and one we want) is `verify-full`, as it provides protection against both eavesdropping and man-in-the-middle attacks. This mode requires you to provide a root certificate to verify against. AWS makes this certificate available on [their website](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#PostgreSQL.Concepts.General.SSL).

```sh
wget https://s3.amazonaws.com/rds-downloads/rds-combined-ca-bundle.pem
```

To use it with `psql`, run:

```sh
psql "postgresql://user:secret@dbhost:5432/ssltest?sslmode=verify-full&sslrootcert=rds-combined-ca-bundle.pem"
```

Once connected, you should see an `SSL connection` line before the first prompt.

There’s also an extension you can use (useful for non-`psql` connections).

```sql
CREATE EXTENSION IF NOT EXISTS sslinfo;
SELECT ssl_is_used();
```

Now direct connections are good, so let’s secure connections from PgBouncer to the database.

## PgBouncer to the Database

Follow [this guide](PgBouncer-Setup.md) to set up PgBouncer. Once that’s completed, there are two settings to add to `/etc/pgbouncer/pgbouncer.ini`:

```ini
server_tls_sslmode = verify-full
server_tls_ca_file = /path/to/rds-combined-ca-bundle.pem
```

Restart the service

```sh
sudo service pgbouncer restart
```

And test it

```sh
psql "postgresql://user:secret@bouncerhost:6432/ssltest"
```

The connection should succeed and the server should report SSL is used.

```sql
SELECT ssl_is_used();
```

We’ve now successfully encrypted traffic between the bouncer and the database!

However, you’ll notice the `psql` prompt does not have an `SSL connection` line as it did before. You can also use `sslmode=disable` to successfully connect, and programs like `tcpdump` or [tshark](https://www.wireshark.org/docs/man-pages/tshark.html) will show unencrypted traffic between the client and the bouncer. You can test this out with:

```sh
sudo tcpdump -i lo -X -s 0 'port 6432'
```

Run commands in `psql` and you’ll see plaintext statements printed.

## Clients to PgBouncer

This last flow is the trickiest. PgBouncer 1.7+ supports TLS, but we need to create keys and certificates for it. For this, we’ll create a private [PKI](https://en.wikipedia.org/wiki/Public_key_infrastructure) with Vault.

Install the [latest version of Vault](https://www.vaultproject.io/downloads.html) and jq

```sh
sudo apt-get install unzip jq
wget https://releases.hashicorp.com/vault/0.9.0/vault_0.9.0_linux_amd64.zip
unzip vault_0.9.0_linux_amd64.zip
sudo mv vault /usr/local/bin
```

Start Vault (we use development mode for this tutorial)

```sh
vault server -dev
```

Create a PKI secret backend

```sh
export VAULT_ADDR='http://127.0.0.1:8200'

vault mount pki
vault mount-tune -max-lease-ttl=87600h pki

vault write pki/root/generate/internal common_name=myvault.com ttl=87600h

vault write pki/config/urls issuing_certificates="http://127.0.0.1:8200/v1/pki/ca" \
    crl_distribution_points="http://127.0.0.1:8200/v1/pki/crl"

vault write pki/roles/pgbouncer \
    allowed_domains="bouncerhost" \
    allow_subdomains="false" max_ttl="72h"
```

And issue certificates

```sh
data=`vault write -format=json pki/issue/pgbouncer common_name=bouncerhost`

jq -r '.data.certificate' <<< $data > cert.pem
jq -r '.data.private_key' <<< $data > key.pem
jq -r '.data.issuing_ca' <<< $data > ca.pem
```

We now have the three files we need. Add the key and certificate to `/etc/pgbouncer/pgbouncer.ini`:

```ini
client_tls_sslmode = require # not verify-full, unless you use client certificate validation
client_tls_key_file = /path/to/key.pem
client_tls_cert_file = /path/to/cert.pem
```

And restart the service. To connect, we once again use `verify-full` but this time with the root certificate we generated above:

```psql
psql "postgresql://user:secret@bouncerhost:6432/ssltest?sslmode=verify-full&sslrootcert=ca.pem"
```

Confirm the `SSL connection` line is printed and `sslmode=disable` no longer works.

We’ve now successfully encrypted traffic end-to-end!
