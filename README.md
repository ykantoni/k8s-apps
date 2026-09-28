# k8s-apps

Applications for [proxclus](https://github.com/ykantoni/proxclus),
reconciled by Argo CD. Merge to `main` and Argo CD syncs it.

vm-infra's Terraform creates the root Application `root-k8s-apps` (project
`bootstrap-apps`), which syncs `bootstrap/` of this repository into the
`argocd-apps` namespace. Every file there is one child Application in project
`k8s-apps`.

This is a separate Argo CD pipeline from k8s-infra, with narrower rights:

| | k8s-infra | k8s-apps (this repo) |
| --- | --- | --- |
| Application namespace | `argocd` | `argocd-apps` (only the `k8s-apps` project accepts Applications from it) |
| Destination namespaces | any | listed in vm-infra's `modules/argocd` `apps_destination_namespaces` (`ollama`, `postgres`) |
| Cluster-scoped kinds | any | `Namespace`, `PersistentVolume`, `StorageClass` only |
| Ordering | sync waves | none: every Application retries until its dependencies exist |

## What's here

| Application      | Namespace  | Source                                          | Depends on (k8s-infra)          |
| ---------------- | ---------- | ----------------------------------------------- | ------------------------------- |
| `ollama-storage` | (cluster)  | `charts/ollama-storage` (local PV + StorageClass) | gpu-operator's node label      |
| `ollama`         | `ollama`   | `ollama` chart + `values/ollama`                | gpu-operator, LB-IPAM           |
| `open-webui`     | `ollama`   | `open-webui` chart + `values/open-webui`        | longhorn, LB-IPAM               |
| `postgres`       | `postgres` | `charts/postgres-cluster` + `manifests/postgres` | cnpg-operator, longhorn        |

There are deliberately no sync waves: one app being unhealthy (say, Postgres
waiting on the CNPG operator) must not hold up an unrelated one. Anything that
races a dependency just retries with backoff and converges.

## Layout

```
bootstrap/<app>.yaml        one Argo CD Application per app (namespace argocd-apps, project k8s-apps)
values/<app>/values.yaml    Helm values for chart-repo apps ($values ref)
charts/<chart>/             local charts (their own values.yaml)
manifests/<app>/            SealedSecrets and other raw manifests for that app
pub-cert.pem                Sealed Secrets public cert, for offline sealing
```

## Adding an app

1. `bootstrap/<app>.yaml`: copy `ollama.yaml` (chart, version, namespace);
   keep `metadata.namespace: argocd-apps` and `project: k8s-apps`.
2. `values/<app>/values.yaml`, or a local chart under `charts/`.
3. In vm-infra's `modules/argocd`, add the app's namespace to
   `apps_destination_namespaces` and, if new, its chart repository to
   `k8s_apps_chart_repos`, then apply vm-infra. The `k8s-apps` project
   rejects anything else.
4. Give the app its own namespace (`CreateNamespace=true`), named after it.
   If its pods can't meet RKE2's CIS-default `restricted` Pod Security level,
   set `managedNamespaceMetadata.labels` (see `ollama.yaml`).
5. Anything that needs a new CRD or operator goes to k8s-infra first.

## Secrets

The repository is public: commit only `SealedSecret`s. `lint.yml` fails on any
plain `kind: Secret`. Get `pub-cert.pem` from `just seal-cert` in vm-infra and
commit it here, then:

```bash
kubectl create secret generic <name> -n <namespace> \
  --from-literal=<key>=<value> --dry-run=client -o yaml \
| kubeseal --cert pub-cert.pem -o yaml > manifests/<app>/<name>.sealed.yaml
```

A SealedSecret only decrypts in the namespace, and under the name, it was
sealed for.

## Notes per app

- **ollama** — runs on the GPU node (`nvidia.com/gpu.present=true`,
  `runtimeClassName: nvidia`, one `nvidia.com/gpu`). Its model cache is the
  local PV from `ollama-storage`, at `/var/lib/ollama-models` on that node;
  that directory must exist before the PVC can bind. Namespace is PSA
  `baseline`, since the images run as root.
- **open-webui** — talks to Ollama at
  `http://ollama.ollama.svc.cluster.local:11434`; its own data is on Longhorn.
- **postgres** — a CNPG `Cluster`, 2 instances, 10Gi each on Longhorn. The
  owner's credentials are CNPG-generated (`postgres-app`) unless you commit
  a SealedSecret; see `manifests/postgres/README.md`.
