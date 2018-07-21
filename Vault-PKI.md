# Vault PKI

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
