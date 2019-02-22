# Trying Out Vault for Postgres Credentials

Install [Vault](https://www.vaultproject.io/), as well as JQ for JSON parsing

```sh
brew install vault jq
```

Start the dev server

```sh
vault server -dev
```

Then open another window. For this demo, we’ll create a new Postgres database.

```sh
createdb myapp
```

Create a Postgres user for Vault to manage other users

```sh
psql -c "CREATE USER vault WITH CREATEROLE ENCRYPTED PASSWORD 'secret';" myapp
```

And create a role to grant to temporary users. This is where you should configure privileges (omitted).

```sh
psql -c "CREATE ROLE app;" myapp
```

Configure Vault. We set a default TTL of 10 seconds for users to test.

```sh
export VAULT_ADDR='http://127.0.0.1:8200'

vault mount database

vault write database/config/postgresql \
    plugin_name=postgresql-database-plugin \
    allowed_roles="app" \
    connection_url="postgresql://vault:secret@localhost:5432/myapp?sslmode=disable"

vault write database/roles/app \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';" \
    default_ttl="10s" \
    max_ttl="24h"
```

Fetch temporary credentials

```sh
vault read -format=json database/creds/app
```

Save the result as environment variables (with JQ)

```sh
data=`vault read -format=json database/creds/app`
export PGUSER=`echo $data | jq -r '.data.username'`
export PGPASSWORD=`echo $data | jq -r '.data.password'`
```

Test the new user

```sh
psql -c "SELECT current_user;" myapp
```

Wait 10 seconds and re-run the command to confirm the user no longer exists
