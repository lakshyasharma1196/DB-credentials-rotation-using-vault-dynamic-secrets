# Vault-integration-on-flask-app

Here we are deploying a FLASK-app connecting to a remote postgres database. We have used helm chart to deploy the application. And using vault side car inhector to provide the environment variables.

# Prerequisites

This tutorial requires the Kubernetes command-line interface (CLI) and the Helm CLI installed, Minikube, and additional configuration to bring it all together.

Docker image should be pushed to docker hub.

```
docker build -f docker/Dockerfile -t my-flask-app .
docker tag my-flask-app your_username/my-flask-app
docker push your_username/my-flask-app
```

To run the docker image locally

```
docker run -p 5000:5000 -e USERNAME=your_db_username -e PASSWORD=your_db_password -e DB_HOST=your_db_host -e DB_NAME=your_db_name my-flask-app

```


# Steps

# Clone the repo to install vault:

 ```
 git clone https://github.com/hashicorp/vault-guides.git
 cd vault-guides/operations/provision-vault/kubernetes/minikube/vault-agent-sidecar
 ```

# Initate the minikube:
 
 ```
 minikube start
 minikube status
 minikube dashboard
 ```

# Install the Vault Helm chart

```
helm repo add hashicorp https://helm.releases.hashicorp.com

helm repo update

helm install vault hashicorp/vault --set "server.dev.enabled=true"

```

Check the vault pods:

```
kubectl get pods
NAME                                    READY   STATUS    RESTARTS   AGE
vault-0                                 1/1     Running   0          80s
vault-agent-injector-5945fb98b5-tpglz   1/1     Running   0          80s
```

## Note: Donot forget to unseal the vault.

```
kubectl exec vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > cluster-keys.json
```

```
jq -r ".unseal_keys_b64[]" cluster-keys.json
```

```
VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" cluster-keys.json)
```

```
kubectl exec vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

```
jq -r ".root_token" cluster-keys.json
```

This will give the root token , can be used while login in later step.

# Vault Dashboard

```
kubectl exec -it vault-0 -- /bin/sh

vault login

```

You will get token. Copy this token.

```
kubectl port-forward vault-0 8200:8200
```

Open browser and got to localhost:8200

A ui will open, paste the token and login.


# Set a secret in Vault

```
kubectl exec -it vault-0 -- /bin/sh
```

Now Run the below commands in vault shell
```
vault secrets enable -path=internal kv-v2

vault kv put internal/database/config USERNAME="your_db_username" PASSWORD="your_db_password" DB_HOST="your_db_host" DB_NAME="your_db_name"

```

Check the secrets are added:

```
vault kv get internal/database/config

exit
```

# Configure Kubernetes authentication

```
kubectl exec -it vault-0 -- /bin/sh

vault auth enable kubernetes
```


```
vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"
```

```
vault policy write my-flask-app - <<EOF
path "internal/data/database/config" {
  capabilities = ["list", "read"]
}

path "/internal/*" {
  capabilities = ["list", "read"]
}
EOF
```

```
vault write auth/kubernetes/role/my-flask-app \
    bound_service_account_names=my-flask-app \
    bound_service_account_namespaces=default \
    policies=my-flask-app \
    ttl=24h

exit
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

helm install my-flask-app ./hello-world
```

To upgrade the chart

```
helm upgrade my-flask-app ./my-flask-app
```

