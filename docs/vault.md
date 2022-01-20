# Using Hashicorp Vault with Redis Connect in K8s

It is a common practice to use Hashicorp Vault to manage sensitive information, such as database credentials, instead of Kubernetes Secrets, Config Maps and/or environment variables.

This document guides you on how to use Redis Connect with database credentials, managed by Vault. Specifically - plugins for the database secrets engine.

We'll use Postgress plugin https://www.vaultproject.io/docs/secrets/databases/postgresql as an example, but the usage for other plugins is very similar.

## Prerequisites

### Vault

We assume that k8s cluster and HCP Vault already installed and configured. You can follow one of the official Vault guides https://www.vaultproject.io/docs/platform/k8s#getting-started-with-vault-and-kubernetes to install and configure Vault on k8s.

### Postgres

TODO: describe setting up postgress or provide link to docs...

Then follow the Vault documentation https://www.vaultproject.io/docs/secrets/databases/postgresql to create `database/config/my-postgresql-database` connection and `database/roles/my-role`.

Alternatevely - https://www.atlantbh.com/keeping-secrets-secure-with-vault-inside-a-kubernetes-cluster/ provides easy to follow guide to setup both Vault and Postgres.

You should be able to request temp. Vault managed credentials with the command:
```bash
vault read database/creds/my-role
Key                Value
---                -----
lease_id           database/creds/my-role/xSxn4CbSgFhNgkWmSFWPIrNP
lease_duration     1h
lease_renewable    true
password           iTI5kq-Eh0Zv4Q8XXXX
username           v-root-my-role-nhoiOG13bv2eXXXXXXXX-1234567890
```

## Create Vault policy
```bash
vault policy write redis-connect - <<EOF
path "database/creds/my-role" {
  capabilities = ["read"]
}
EOF
```

## Create k8s service account
```bash
kubectl create sa redis-connect
```

## Bind k8s service account to the vault policy
```bash
vault write auth/kubernetes/role/redis-connect \
    bound_service_account_names=redis-connect \
    bound_service_account_namespaces=default \
    policies=redis-connect \
    ttl=24h
```

At this point, every pod, holding the `redis-connect` Service Account would be able to retreive database credential, managed by the Vault.

## Injecting secrets into Redis Connect pod via Vault Agent Containers

TODO: instead of the abstract ubuntu pod describe changes to the redis-connect yaml files?

Maybe using `kubectl patch`?

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-connect
  labels:
    app: redis-connect
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/role: "redis-connect"
    # File export 
    #vault.hashicorp.com/agent-inject-secret-credentials.txt: "database/creds/my-role"

    vault.hashicorp.com/agent-inject-secret-config: 'database/creds/my-role'
    # Environment variable export template
    vault.hashicorp.com/agent-inject-template-config: |
      {{ with secret "database/creds/my-role" -}}
        export REDISCONNECT_SOURCE_PASSWORD="{{ .Data.password }}"
        export REDISCONNECT_SOURCE_USERNAME="{{ .Data.username }}"
      {{- end }}
spec:
  serviceAccountName: redis-connect
  containers:
    - name: redis-connect
      image: ubuntu
      command: ["/bin/bash","-c","source /vault/secrets/config && /bin/sleep 1000"]
```

`source /vault/secrets/config` loads credentials into environment variables. It has to be run before your usual `command`

Troubleshooting: if ubuntu container gets into  `Back-off restarting failed container` - remove the `source /vault/secrets/config &&` from command and examine the vault-agent sidecar container logs:

```bash
kubectl logs redis-connect -c vault-agent
```
