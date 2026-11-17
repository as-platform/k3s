# k3s

🇬🇧 [English version](README.md)

GitOps-репозиторий домашнего кластера K3s платформы [as-platform](https://github.com/as-platform). Единственный источник правды для Argo CD: всё, что работает в кластере, описано здесь и применяется только через Argo CD.

## Структура

```text
.
├── bootstrap/root-app.yaml   # корневое Application (app-of-apps)
├── argocd/
│   ├── projects/             # AppProject
│   └── appsets/              # ApplicationSet
├── infra/<имя>/              # компонент инфраструктуры (обёртка-чарт)
├── apps/<app>/               # values.yaml и catalog-info.yaml приложения
├── catalog/                  # all.yaml, groups.yaml для Backstage
└── renovate.json             # автообновление версий чартов
```

## Как это работает

1. `bootstrap/root-app.yaml` создаёт Application `root`, который смотрит в каталог `argocd/` (`directory.recurse: true`).
2. `argocd/projects/infra.yaml` задаёт AppProject `infra` (sync-wave `-1`).
3. `argocd/appsets/infra.yaml` — ApplicationSet с git-генератором по `infra/*` (sync-wave `1`): на каждый каталог создаётся Application с именем и namespace, равными имени каталога. Включены `automated` (prune + selfHeal), `CreateNamespace=true`, `ServerSideApply=true`.
4. Argo CD управляет и самим собой: компонент `infra/argocd`.

Новый компонент появляется в кластере после коммита каталога `infra/<имя>` в `main`. Репозиторий опрашивается раз в 3 минуты.

## Компоненты `infra/`

| Каталог            | Upstream-чарт                                                      | Версия  |
| ------------------ | ------------------------------------------------------------------ | ------- |
| `argocd`           | argo-cd                                                            | 10.10.5 |
| `backstage`        | собственные шаблоны (Backstage, Postgres, Ingress, ExternalSecret) | 0.1.0   |
| `cert-manager`     | cert-manager                                                       | v1.21.3 |
| `external-secrets` | external-secrets                                                   | 2.14.0  |
| `github-runner`    | собственные шаблоны (KEDA `ScaledJob`, NetworkPolicy, CA, токен)   | 0.1.0   |
| `keda`             | keda                                                               | 2.21.0  |
| `metallb`          | metallb                                                            | 0.16.1  |
| `monitoring`       | kube-prometheus-stack                                              | 92.16.2 |
| `openbao`          | openbao                                                            | 0.30.7  |
| `traefik`          | traefik                                                            | 42.1.3  |
| `zot`              | zot                                                                | 0.1.128 |

## Проверка изменений в `argocd/`

Перед пушем файлы из `argocd/` проверяются на сервере:

```bash
kubectl apply --dry-run=server -f <файл>
```

Битый файл не ломает работающие приложения, но `root` перестаёт синкаться, пока не придёт исправление.

## Восстановление кластера с нуля

Нужно только при полной потере кластера. На ноде под root:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
git clone https://github.com/as-platform/k3s.git && cd k3s
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
helm dependency build infra/argocd
helm install argocd infra/argocd -n argocd --create-namespace --wait --timeout 10m
kubectl create namespace openbao
kubectl apply -f openbao-bootstrap.yaml
kubectl apply -f bootstrap/root-app.yaml
```

`openbao-bootstrap.yaml` — сохранённый Secret `openbao-bootstrap`, подготовленный как в «Восстановление OpenBao с нуля»; он должен существовать до `root-app`. Snapshot OpenBao восстановить после того, как `openbao` станет Healthy, как описано в том же подразделе.

После этого Argo CD подхватывает сам себя через `infra/argocd` и разворачивает остальное. Дальше `helm upgrade` не использовать: все изменения идут через git.

## Секреты

Значения лежат в OpenBao KV v2 по пути `secret/<namespace>/<name>`. В git хранится только ExternalSecret:

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

ClusterSecretStore `openbao` обслуживает только namespace из `spec.conditions` в `infra/external-secrets/templates/cluster-secret-store.yaml` (сейчас: `argocd`, `backstage`, `github-runner`, `monitoring`, `zot`).

OpenBao работает в режиме standalone, хранилище raft на PVC local-path. Seal статический: 32-байтный ключ из Secret `openbao-bootstrap` в namespace `openbao`, создаётся вручную и никогда не управляется Argo CD. UI доступен по `https://openbao.homelab` через Traefik с сертификатом от `homelab-ca`; на хосте нужна строка `192.168.1.50 openbao.homelab` в `/etc/hosts`, как для остальных `*.homelab`. Метод входа в UI — `Username`, пользователь `admin`, пароль — ключ `admin-password` Secret `openbao-bootstrap`. Внутри кластера ESO по-прежнему ходит по обычному HTTP `http://openbao.openbao.svc:8200`. Self-init (блоки `initialize` в `infra/openbao/values.yaml`) создаёт kv-v2 `secret/`, пользователя userpass `admin`, kubernetes auth, а также политику и роль `external-secrets` (только чтение `secret/data/*`, привязка к ServiceAccount `external-secrets` в namespace `external-secrets`).

### Работа с OpenBao

1. Доступ:

   ```bash
   kubectl -n openbao port-forward svc/openbao 8200:8200
   export BAO_ADDR=http://127.0.0.1:8200
   bao login -method=userpass username=admin
   ```

   Пароль — ключ `admin-password` Secret `openbao-bootstrap`. Альтернатива CLI — UI по `https://openbao.homelab` (см. «Секреты»).

2. Добавить секрет: `bao kv put secret/<ns>/<name> key=value`, добавить ExternalSecret в чарт компонента, а новый namespace — в `conditions`.
3. Ротация значения:

   ```bash
   bao kv patch secret/<ns>/<name> key=new
   kubectl -n <ns> annotate externalsecret <name> force-sync=$(date +%s) --overwrite
   ```

   Для `zot-htpasswd` ещё `kubectl -n zot delete pod zot-0`: Zot не перечитывает htpasswd. Для потребителей в `argocd` и `backstage` перезапустить под, если он читает значение при старте. Для ротации `POSTGRES_PASSWORD` в `backstage/backstage` сначала нужен `ALTER USER` в `postgres-0` (Postgres хранит пароль в своём PVC), и только потом `kv patch`, force-sync и перезапуск `backstage`. Для `monitoring/grafana-admin` достаточно перезапустить под Grafana: в `infra/monitoring/values.yaml` персистентность Grafana не настроена, пароль администратора читается из Secret при старте.
4. Политики и роли правятся через `bao policy write` и `bao write auth/kubernetes/role/...`. Блок `initialize` выполняется один раз на свежем хранилище и остаётся в git только для восстановления с нуля.
5. Snapshot:

   ```bash
   bao operator raft snapshot save openbao-$(date +%F).snap
   ```

   Хранить рядом с бэкапом Secret `openbao-bootstrap`. Делать после каждого изменения KV или политик и перед обновлением чарта. Snapshot без статического ключа бесполезен, ключ без snapshot тоже.
6. Ротация статического ключа: смонтировать оба ключа, в блоке `seal "static"` задать `current_key_id` и `current_key` новым, а `previous_key_id` и `previous_key` старым, перекатить под, позже убрать предыдущий ключ.

### Восстановление OpenBao с нуля

1. До `root-app` создать namespace `openbao` и применить Secret из сохранённого `openbao-bootstrap.yaml`, убрав `uid`, `resourceVersion` и `creationTimestamp`.
2. Когда `openbao` Healthy: `bao login`, затем `bao operator raft snapshot restore -force <файл>`, затем снова `bao login` (токены меняются), удалить под `openbao-0`, принудительно синкнуть все ExternalSecret. Второй `bao login` использует пароль администратора, действовавший на момент snapshot, поэтому при ротации пароля в OpenBao держать `admin-password` в `openbao-bootstrap` синхронным.
3. Завершить и пересинкнуть зависшие Application.

Namespace `openbao` не удалять: статический ключ уйдёт вместе с ним, хранилище останется запечатанным навсегда.

### Конфликт владения

Если ESO сообщает о конфликте владения существующим Secret, выполнить `kubectl -n <ns> delete <kind> <name> --cascade=orphan` на старом владельце, затем force-sync. В крайнем случае удалить Secret и дать ESO пересоздать его (uid сменится).

## Правила

- Деплой только через Argo CD, прямой `kubectl apply` на кластер не использовать.
- Исключения из предыдущего правила: Secret `openbao-bootstrap`, `kubectl delete <kind> <name> --cascade=orphan` при разовой миграции с прежнего инструмента секретов, операционные шаги runbook OpenBao на живых объектах (аннотации force-sync, перезапуски подов, разбор конфликтов владения) и стирание PVC OpenBao, пока KV пуст.
- Zot перечитывает `config.json` без перезапуска (проверено для `sync` и `retention`): примерно через минуту после синка Argo CD в логе пода появляется новая запись `configuration settings`.
- Версии чартов обновляет Renovate (по понедельникам до 6:00); чарты Argo CD группируются в один PR.

## Связанные репозитории

- [`homelab-infra`](https://github.com/as-platform/homelab-infra) — Ansible-код, который создаёт кластер K3s, управляемый этим репозиторием
- [`ci-templates`](https://github.com/as-platform/ci-templates) — переиспользуемые workflow, чей `deploy-bump` коммитит теги образов в `apps/`
- [`backstage-templates`](https://github.com/as-platform/backstage-templates) — шаблоны, которые открывают PR с `apps/<имя>/`
- [`backstage`](https://github.com/as-platform/backstage) — портал, который читает `catalog/`

## Лицензия

MIT — см. [`LICENSE`](LICENSE).
