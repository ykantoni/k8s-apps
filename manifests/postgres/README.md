# postgres SealedSecrets

Synced into `postgres` by the `postgres` Application (second source). Only
`*.yaml`/`*.yml`/`*.json` files here are applied.

Optional: `app-owner.sealed.yaml`, the initdb owner's credentials. Without
it, CNPG generates a random password into `postgres-app`. To choose it
yourself (from the repo root, `pub-cert.pem` present):

```bash
kubectl create secret generic app-owner -n postgres \
  --type=kubernetes.io/basic-auth \
  --from-literal=username=app \
  --from-literal=password='<choose one>' \
  --dry-run=client -o yaml \
| kubeseal --cert pub-cert.pem -o yaml \
> manifests/postgres/app-owner.sealed.yaml
```

then set `bootstrap.initdb.secretName: app-owner` in
`charts/postgres-cluster/values.yaml`. `username` must match
`bootstrap.initdb.owner`. initdb only runs when the cluster is first created.
