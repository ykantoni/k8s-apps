# postgres ExternalSecrets

Synced into `postgres` by the `postgres` Application (second source). Only
`*.yaml`/`*.yml`/`*.json` files here are applied.

Optional: `app-owner.yaml`, the initdb owner's credentials. Without it, CNPG
generates a random password into `postgres-app`. To choose it yourself,
write it to OpenBao (token: see k8s-infra's `manifests/openbao/README.md`):

```bash
kubectl -n openbao exec -it openbao-0 -- sh -c \
  'BAO_TOKEN=<token> bao kv put secret/postgres/app-owner \
     username=app password=<choose one>'
```

commit `manifests/postgres/app-owner.yaml`:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: app-owner
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: app-owner
    template:
      type: kubernetes.io/basic-auth
  dataFrom:
    - extract:
        key: postgres/app-owner
```

then set `bootstrap.initdb.secretName: app-owner` in
`charts/postgres-cluster/values.yaml`. `username` must match
`bootstrap.initdb.owner`. initdb only runs when the cluster is first created.
