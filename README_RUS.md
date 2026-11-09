# k3s

🇬🇧 [English version](README.md)

GitOps-репозиторий домашнего кластера K3s. Единственный источник правды для Argo CD: всё, что работает в кластере, описано здесь и применяется только через Argo CD.

## Структура

```
k3s/
├── bootstrap/root-app.yaml   корневое Application (app-of-apps)
├── argocd/
│   ├── projects/             AppProject
│   └── appsets/              ApplicationSet
├── infra/<имя>/              компонент инфраструктуры (обёртка-чарт)
├── apps/<app>/               values.yaml и catalog-info.yaml приложения
├── catalog/                  all.yaml, groups.yaml для Backstage
└── renovate.json             автообновление версий чартов
```

## Как это работает

1. `bootstrap/root-app.yaml` создаёт Application `root`, который смотрит в каталог `argocd/` (`directory.recurse: true`).
2. `argocd/projects/infra.yaml` задаёт AppProject `infra` (sync-wave `-1`).
3. `argocd/appsets/infra.yaml` — ApplicationSet с git-генератором по `infra/*` (sync-wave `1`): на каждый каталог создаётся Application с именем и namespace, равными имени каталога. Включены `automated` (prune + selfHeal), `CreateNamespace=true`, `ServerSideApply=true`.
4. Argo CD управляет и самим собой: компонент `infra/argocd`.

Новый компонент появляется в кластере после коммита каталога `infra/<имя>` в `main`. Репозиторий опрашивается раз в 3 минуты.

## Компоненты `infra/`


| Каталог          | Upstream-чарт                                     | Версия  |
| ---------------- | ------------------------------------------------- | ------- |
| `argocd`         | argo-cd                                           | 10.10.5 |
| `backstage`      | собственные шаблоны (Backstage, Postgres, Ingress, SealedSecret) | 0.1.0   |
| `cert-manager`   | cert-manager                                      | v1.21.3 |
| `github-runner`  | собственные шаблоны (KEDA `ScaledJob`, NetworkPolicy, CA, токен) | 0.1.0   |
| `keda`           | keda                                              | 2.21.0  |
| `metallb`        | metallb                                           | 0.16.1  |
| `monitoring`     | kube-prometheus-stack                             | 92.16.2 |
| `sealed-secrets` | sealed-secrets                                    | 2.20.0  |
| `traefik`        | traefik                                           | 42.1.3  |
| `zot`            | zot                                               | 0.1.128 |

## Проверка изменений в `argocd/`

Перед пушем файлы из `argocd/` проверяются на сервере:

```
kubectl apply --dry-run=server -f <файл>
```

Битый файл не ломает работающие приложения, но `root` перестаёт синкаться, пока не придёт исправление.

## Восстановление кластера с нуля

Нужно только при полной потере кластера. На ноде под root:

```
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
git clone https://github.com/as-platform/k3s.git && cd k3s
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm dependency build infra/argocd
helm install argocd infra/argocd -n argocd --create-namespace --wait --timeout 10m
kubectl apply -f bootstrap/root-app.yaml
```

После этого Argo CD подхватывает сам себя через `infra/argocd` и разворачивает остальное. Дальше `helm upgrade` не использовать: все изменения идут через git.

## Секреты

Секреты хранятся в git только в зашифрованном виде (Sealed Secrets). Namespace контроллера — `sealed-secrets`, поэтому `kubeseal` нужно запускать так:

```
kubeseal --controller-namespace sealed-secrets --controller-name sealed-secrets-controller
```

## Правила

- Деплой только через Argo CD, прямой `kubectl apply` на кластер не использовать.
- Версии чартов обновляет Renovate (по понедельникам до 6:00); чарты Argo CD группируются в один PR.