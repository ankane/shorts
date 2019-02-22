# Vault for PKI

Here’s how to use Vault for public key infrastructure.

---

**Update:** Vault now has a [great article](https://learn.hashicorp.com/vault/secrets-management/sm-pki-engine) on this

---

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

vault write pki/roles/yourrole \
    allowed_domains="yourhost" \
    allow_subdomains="false" max_ttl="72h"
```

And issue certificates

```sh
data=`vault write -format=json pki/issue/yourrole common_name=yourhost`

jq -r '.data.certificate' <<< $data > cert.pem
jq -r '.data.private_key' <<< $data > key.pem
jq -r '.data.issuing_ca' <<< $data > ca.pem
```
