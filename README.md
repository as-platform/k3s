# k3s

🇷🇺 [Русская версия](README_RUS.md)

A GitOps repository for a home K3s cluster. The single source of truth for Argo CD: everything running in the cluster is described here and applied only through Argo CD.

## Structure

```
k3s/
├── bootstrap/root-app.yaml   root Application (app-of-apps)
├── argocd/
│   ├── projects/             AppProject
│   └── appsets/              ApplicationSet
├── infra/<name>/             an infrastructure component (wrapper chart)
├── apps/<app>/               an application's values.yaml and catalog-info.yaml
├── catalog/                  all.yaml, groups.yaml for Backstage
└── renovate.json             automatic chart version updates
```

## How it works

1. `bootstrap/root-app.yaml` creates the `root` Application, which watches the `argocd/` directory (`directory.recurse: true`).
2. `argocd/projects/infra.yaml` defines the `infra` AppProject (sync-wave `-1`).
3. `argocd/appsets/infra.yaml` is an ApplicationSet with a git generator over `infra/*` (sync-wave `1`): for each directory it creates an Application whose name and namespace equal the directory name. `automated` (prune + selfHeal), `CreateNamespace=true` and `ServerSideApply=true` are enabled.
4. Argo CD manages itself too: the `infra/argocd` component.

A new component appears in the cluster after the `infra/<name>` directory is committed to `main`. The repository is polled every 3 minutes.

## `infra/` components


| Directory        | Upstream chart                                    | Version |
| ---------------- | ------------------------------------------------- | ------- |
| `argocd`         | argo-cd                                           | 10.10.5 |
| `cert-manager`   | cert-manager                                      | v1.21.3 |
| `github-runner`  | own templates (KEDA `ScaledJob`, NetworkPolicy, CA, token) | 0.1.0   |
| `keda`           | keda                                              | 2.21.0  |
| `metallb`        | metallb                                           | 0.16.1  |
| `monitoring`     | kube-prometheus-stack                             | 92.16.2 |
| `sealed-secrets` | sealed-secrets                                    | 2.20.0  |
| `traefik`        | traefik                                           | 42.1.3  |
| `zot`            | zot                                               | 0.1.128 |

## Checking changes in `argocd/`

Before pushing, files from `argocd/` are validated on the server:

```
kubectl apply --dry-run=server -f <file>
```

A broken file does not break the running applications, but `root` stops syncing until a fix arrives.

## Restoring the cluster from scratch

Needed only after a complete loss of the cluster. On the node, as root:

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
git clone https://github.com/as-platform/k3s.git && cd k3s
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm dependency build infra/argocd
helm install argocd infra/argocd -n argocd --create-namespace --wait --timeout 10m
kubectl apply -f bootstrap/root-app.yaml
```

After that Argo CD picks itself up through `infra/argocd` and deploys everything else. Do not use `helm upgrade` afterwards: all changes go through git.

## Secrets

Secrets are stored in git only in encrypted form (Sealed Secrets). The controller's namespace is `sealed-secrets`, so `kubeseal` must be run like this:

```
kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets-controller
```

## Rules

- Deploy only through Argo CD; do not use a direct `kubectl apply` on the cluster.
- Zot reloads `config.json` without a restart (checked for `sync` and `retention`): about a minute after Argo CD syncs, a new `configuration settings` entry appears in the pod log.
- Chart versions are updated by Renovate (Mondays before 6:00); the Argo CD charts are grouped into a single PR.
