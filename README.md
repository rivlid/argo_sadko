# argo_sadko — GitOps-конфигурация кластера

Эта репа — источник правды по ArgoCD: values для установки самого ArgoCD и все
Application-манифесты. Кластер управляется по схеме:

```
git (gitlab.sadkomed.ru) ──> ArgoCD ──> кластер
```

Руками в кластере ничего не меняется. Любое изменение = коммит в соответствующую
репу, ArgoCD накатывает сам (авто-sync включён у всех приложений).

## Состав репы

| Файл                     | Что это                                                        |
|--------------------------|----------------------------------------------------------------|
| `argocd-helm-values.yaml`| Helm values для установки/обновления самого ArgoCD             |
| `metallb.yaml`           | Application: MetalLB целиком (репа `metallb`, path `sadkomed_bgp`) |
| `ingress-nginx.yaml`     | Application: прод ingress-контроллер, IP 192.168.253.150        |
| `ingress-nginx-test.yaml`| Application: тестовый контроллер, class `nginx-test`, IP 192.168.253.69 |
| `sadko-proxy.yaml`       | Application: reverse-прокси (чарт `sadko_first` из репы `helm`) |

## Связанные репозитории (gitlab.sadkomed.ru)

- `k8s/ingress-nginx.git` — вендоренный официальный чарт ingress-nginx **4.10.1**
  (контроллер v1.10.1). Вендорим, потому что `kubernetes.github.io` /
  `argoproj.github.io` (GitHub Pages CDN) из pod-сети кластера недоступны
  (TLS handshake timeout), при этом сам `github.com` доступен.
- `k8s/helm.git` — чарт `sadko_first` (path `sadko_first`): 60 реверс-прокси,
  каждый = Service + EndpointSlice + Ingress. Backend-адреса — в `values.yaml` чарта.
- репа `metallb` — манифесты MetalLB + конфигурация BGP (пулы, peer, advertisement).

---

# Установка начисто

Предполагается: кластер уже развёрнут (плейбуки `ansible_k8s` 01–06), MetalLB и
ingress ставим НЕ ансиблом (плейбуки 07 и 08 — deprecated), а через ArgoCD, как ниже.

## 1. Установка ArgoCD

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd --create-namespace \
  --version 10.1.2 -f argocd-helm-values.yaml
```

Если `argoproj.github.io` недоступен с хоста установки — взять чарт из кэша helm
(`~/.cache/helm/repository/argo-cd-10.1.2.tgz`) или скачать tgz с github.com
(releases репы argoproj/argo-helm) и ставить из файла.

Пароль admin (UI пока доступен только через port-forward — ingress появится на шаге 4):

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
kubectl port-forward service/argocd-server -n argocd 8080:443
```

### Что и зачем в argocd-helm-values.yaml

| Ключ | Зачем |
|------|-------|
| `configs.params."server.insecure": true` | ArgoCD-server отдаёт HTTP без TLS — терминация TLS на ingress (хост argocd.sadkomed.ru идёт через sadko-proxy) |
| `redis.image.*` | Явный образ redis с docker.io (обход дефолтного registry) |
| `configs.cm."resource.exclusions"` | **Критично.** ArgoCD 3.x по умолчанию исключает из управления `EndpointSlice`, а чарт `sadko_first` создаёт их вручную (в них backend-IP всех прокси). Без переопределения ArgoCD молча НЕ будет применять изменения адресов. Наш список = дефолтный минус Endpoints/EndpointSlice |

**Правило:** любые настройки ArgoCD меняются ТОЛЬКО в этом файле с последующим
`helm upgrade argocd argo/argo-cd -n argocd --version <тек. версия> -f argocd-helm-values.yaml`.
ConfigMap `argocd-cm` руками не редактировать — helm затрёт при следующем upgrade.
После изменения только конфига поды надо перезапустить:

```bash
kubectl rollout restart statefulset argocd-application-controller -n argocd
kubectl rollout restart deploy argocd-repo-server argocd-server -n argocd
```

## 2. Доступ ArgoCD к репозиториям GitLab

Если проекты в GitLab приватные — подключить каждую репу: UI → Settings →
Repositories → Connect repo (https + deploy token), либо секретом с меткой
`argocd.argoproj.io/secret-type: repository`. Если сертификат GitLab не от
публичного CA — добавить CA: Settings → Certificates.

## 3. TLS-секрет для прокси

Чарт `sadko_first` ссылается на секрет `sadkomed-tls` (wildcard *.sadkomed.ru)
в namespace `default`. На чистом кластере создать до синка sadko-proxy:

```bash
kubectl create secret tls sadkomed-tls -n default --cert=fullchain.pem --key=privkey.pem
```

## 4. Применение Application-ов (порядок важен)

```bash
kubectl apply -f metallb.yaml              # 1. MetalLB: пулы, BGP-пиринг
kubectl apply -f ingress-nginx.yaml        # 2. прод-контроллер -> 192.168.253.150
kubectl apply -f ingress-nginx-test.yaml   # 3. тестовый -> 192.168.253.69
kubectl apply -f sadko-proxy.yaml          # 4. все прокси
```

На чистом кластере перед шагом 2 создать namespace (в Application прода нет
CreateNamespace): `kubectl create namespace ingress-nginx`.

Проверка после каждого шага:

```bash
kubectl get application -n argocd                       # все Synced/Healthy
kubectl get svc -n ingress-nginx                        # EXTERNAL-IP 192.168.253.150
kubectl get svc -n ingress-nginx-test                   # EXTERNAL-IP 192.168.253.69
kubectl get ingress -n default | wc -l                  # ~60 ингрессов
curl -kI https://192.168.253.150 -H "Host: grafana.sadkomed.ru"
```

---

# Эксплуатация

- **Добавить/поменять прокси** — правка `values.yaml` в репе `k8s/helm.git`
  (блок `proxies` или `internal`), push → ArgoCD синкает сам. Никаких
  `helm upgrade`/`helm uninstall` для sadko-proxy — selfHeal откатит ручные
  изменения, а uninstall удалит боевые ингрессы.
- **Обновить ingress-nginx** — положить новую версию чарта в `k8s/ingress-nginx.git`
  (helm pull новой версии → закоммитить поверх), push. Сначала обкатать на
  ingress-nginx-test (реки смотрят на одну репу — обновлять через отдельную ветку
  и `targetRevision`, либо последовательно).
- **Обновить ArgoCD** — `helm upgrade ... --version <новая>` с этим же values-файлом.
- **Тонкость sadko-proxy:** kubernetes дописывает эндпойнтам `conditions: {}`,
  поэтому в Application стоит `ignoreDifferences` на `.endpoints[].conditions` —
  без него все EndpointSlice вечно OutOfSync.
- **Новые IP-пулы MetalLB** — в репе metallb; BGPAdvertisement `bgp-253`
  перечисляет пулы явно (если убрать поле `ipAddressPools` — анонсируются все).
