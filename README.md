# DB Credentials Rotation Using Vault Dynamic secrets

## Prerequetes

To setup flask application on Minikube with vault secret injection can be followed here:

We are moving one step further to use the dynamic secrets for DB connection instead of static credentials. After complete the steps provided in above link. Dont launch the application instead follow these steps:

# Steps:

```
kubectl exec -it vault-0 -- /bin/sh

vault secrets enable database

```
Create a database connection:
Replace the vaule of your_db_name & your_db_name with your actual values.
```
vault write database/config/mydb \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@your_db_host:5432/your_db_name?sslmode=disable" \
    allowed_roles="my-role" \
    username="db_username" \
    password="db_password"

```

Create a database role.
```
vault write database/roles/my-role \
    db_name=mydb \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

Update your ACL policy by adding database permission in it.

```
path "internal/data/database/config" {
  capabilities = ["list", "read"]
}

path "/internal/*" {
  capabilities = ["list", "read"]
}

path "database/creds/my-role" {
  capabilities = [ "create", "read", "update", "delete", "list" ]
}
```

# Launch an application

We are using helm to deploy the application.

If you are creating from scratch then use below commnad to create the chart.

```
helm create chart/my-flask-app
```

To install the chart.

```
cd chart

helm install my-flask-app ./my-flask-app
```

To upgrade the chart

```
helm upgrade my-flask-app ./my-flask-app
```