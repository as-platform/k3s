# k3s

🇷🇺 [Русская версия](README_RUS.md)

The GitOps repository of the [as-platform](https://github.com/as-platform) homelab cluster on K3s. The single source of truth for Argo CD: everything running in the cluster is described here and applied only through Argo CD.

## Structure

```text
.
├── bootstrap/root-app.yaml   # root Application (app-of-apps)
├── argocd/
│   ├── projects/             # AppProject
│   └── appsets/              # ApplicationSet
├── infra/<name>/             # an infrastructure component (wrapper chart)
├── apps/<app>/               # an application's values.yaml and catalog-info.yaml
├── catalog/                  # all.yaml, groups.yaml for Backstage
└── renovate.json             # automatic chart version updates
```

## How it works

1. `bootstrap/root-app.yaml` creates the `root` Application, which watches the `argocd/` directory (`directory.recurse: true`).
2. `argocd/projects/infra.yaml` defines the `infra` AppProject (sync-wave `-1`).
3. `argocd/appsets/infra.yaml` is an ApplicationSet with a git generator over `infra/*` (sync-wave `1`): for each directory it creates an Application whose name and namespace equal the directory name. `automated` (prune + selfHeal), `CreateNamespace=true` and `ServerSideApply=true` are enabled.
4. Argo CD manages itself too: the `infra/argocd` component.

A new component appears in the cluster after the `infra/<name>` directory is committed to `main`. The repository is polled every 3 minutes.

## `infra/` components

| Directory          | Upstream chart                                               | Version |
| ------------------ | ------------------------------------------------------------ | ------- |
| `argocd`           | argo-cd                                                      | 10.10.5 |
| `backstage`        | own templates (Backstage, Postgres, Ingress, ExternalSecret) | 0.1.0   |
| `cert-manager`     | cert-manager                                                 | v1.21.3 |
| `external-secrets` | external-secrets                                             | 2.14.0  |
| `github-runner`    | own templates (KEDA `ScaledJob`, NetworkPolicy, CA, token)   | 0.1.0   |
| `keda`             | keda                                                         | 2.21.0  |
| `metallb`          | metallb                                                      | 0.16.1  |
| `monitoring`       | kube-prometheus-stack                                        | 92.16.2 |
| `openbao`          | openbao                                                      | 0.30.7  |
| `traefik`          | traefik                                                      | 42.1.3  |
| `zot`              | zot                                                          | 0.1.128 |

## Checking changes in `argocd/`

Before pushing, files from `argocd/` are validated on the server:

```bash
kubectl apply --dry-run=server -f <file>
```

A broken file does not break the running applications, but `root` stops syncing until a fix arrives.

## Restoring the cluster from scratch

Needed only after a complete loss of the cluster. On the node, as root:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
git clone https://github.com/as-platform/k3s.git && cd k3s
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm dependency build infra/argocd
helm install argocd infra/argocd -n argocd --create-namespace --wait --timeout 10m
kubectl apply -f bootstrap/root-app.yaml
```

After that Argo CD picks itself up through `infra/argocd` and deploys everything else. Do not use `helm upgrade` afterwards: all changes go through git.

## Secrets

Values live in OpenBao KV v2 at `secret/<namespace>/<name>`. Git holds only an ExternalSecret:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: <name>
  namespace: <ns>
  annotations:
    argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: openbao
  target:
    name: <name>
    creationPolicy: Owner
    deletionPolicy: Retain
  dataFrom:
    - extract:
        key: <ns>/<name>
```

The ClusterSecretStore `openbao` serves only the namespaces listed in `spec.conditions` of `infra/external-secrets/templates/cluster-secret-store.yaml` (now: `argocd`, `backstage`, `github-runner`, `monitoring`, `zot`).

OpenBao runs standalone with raft storage on a local-path PVC. The seal is static: a 32-byte key from the Secret `openbao-bootstrap` in the `openbao` namespace, created by hand and never managed by Argo CD. There is no UI and no Ingress, access is HTTP inside the cluster. Self-init (`initialize` blocks in `infra/openbao/values.yaml`) creates the kv-v2 mount `secret/`, the userpass user `admin`, kubernetes auth, and the policy and role `external-secrets` (read-only on `secret/data/*`, bound to the ServiceAccount `external-secrets` in the `external-secrets` namespace).

### Working with OpenBao

1. Access:

   ```bash
   kubectl -n openbao port-forward svc/openbao 8200:8200
   export BAO_ADDR=http://127.0.0.1:8200
   bao login -method=userpass username=admin
   ```

   The password is the key `admin-password` of the Secret `openbao-bootstrap`.

2. Add a secret: `bao kv put secret/<ns>/<name> key=value`, add an ExternalSecret to the component chart, and add the namespace to `conditions` if it is new.
3. Rotate a value:

   ```bash
   bao kv patch secret/<ns>/<name> key=new
   kubectl -n <ns> annotate externalsecret <name> force-sync=$(date +%s) --overwrite
   ```

   For `zot-htpasswd` also run `kubectl -n zot delete pod zot-0`: Zot does not reread htpasswd. Restart the consumer pod in `argocd` and `backstage` if it reads the value at start.
4. Edit policies and roles with `bao policy write` and `bao write auth/kubernetes/role/...`. The `initialize` block runs once on a fresh storage and stays in git only for restoring from scratch.
5. Snapshot:

   ```bash
   bao operator raft snapshot save openbao-$(date +%F).snap
   ```

   Keep it next to the backup of the `openbao-bootstrap` Secret. Take one after every KV or policy change and before a chart upgrade. A snapshot without the static key is useless, and the key without a snapshot too.
6. Static key rotation: mount both keys, set `current_key_id` and `current_key` to the new one and `previous_key_id` and `previous_key` to the old one in the `seal "static"` block, roll the pod, drop the previous key later.

### Restoring OpenBao from scratch

1. Before `root-app`, create the `openbao` namespace and apply the Secret from the saved `openbao-bootstrap.yaml` after removing `uid`, `resourceVersion` and `creationTimestamp`.
2. When `openbao` is Healthy: `bao login`, then `bao operator raft snapshot restore -force <file>`, then `bao login` again (tokens change), delete the pod `openbao-0`, force-sync every ExternalSecret.
3. Terminate and resync the stuck Applications.

Never delete the `openbao` namespace: the static key goes with it and the storage can never be opened again.

### Ownership conflict

If ESO reports an ownership conflict on an existing Secret, run `kubectl -n <ns> delete <kind> <name> --cascade=orphan` on the old owner, then force-sync. As a last resort delete the Secret and let ESO recreate it (the uid changes).

## Rules

- Deploy only through Argo CD; do not use a direct `kubectl apply` on the cluster.
- Exceptions to the previous rule: the `openbao-bootstrap` Secret, `kubectl delete <kind> <name> --cascade=orphan` during the one-time migration from the previous secret tooling, and wiping the OpenBao PVC while the KV is still empty.
- Zot reloads `config.json` without a restart (checked for `sync` and `retention`): about a minute after Argo CD syncs, a new `configuration settings` entry appears in the pod log.
- Chart versions are updated by Renovate (Mondays before 6:00); the Argo CD charts are grouped into a single PR.

## Related repositories

- [`homelab-infra`](https://github.com/as-platform/homelab-infra) — the Ansible code that creates the K3s cluster this repository manages
- [`ci-templates`](https://github.com/as-platform/ci-templates) — the reusable workflows whose `deploy-bump` commits image tags to `apps/`
- [`backstage-templates`](https://github.com/as-platform/backstage-templates) — the templates that open a pull request adding `apps/<name>/`
- [`backstage`](https://github.com/as-platform/backstage) — the portal that reads `catalog/`

## License

MIT — see [`LICENSE`](LICENSE).
