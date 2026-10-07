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
| `ingress-nginx.yaml`     | Application: прод ingress-контроллер, class `nginx`, IP 192.168.253.150 |
| `ingress-nginx-test.yaml`| Application: тестовый контроллер, class `nginx-test`, IP 192.168.253.69 |
| `proxy-150.yaml`         | Application: реверс-прокси прод-контроллера (чарт `sadko_first`, values-файл `values-proxy-150.yaml`) |
| `truenas-iscsi.yaml`     | Application: democratic-csi (TrueNAS iSCSI, драйвер `freenas-api-iscsi`), namespace `democratic-csi`, StorageClass `truenas-iscsi` (RWO, не default) |

## Связанные репозитории (gitlab.sadkomed.ru)

- `k8s/ingress-nginx.git` — вендоренный официальный чарт ingress-nginx **4.10.1**
  (контроллер v1.10.1). Вендорим, потому что `kubernetes.github.io` /
  `argoproj.github.io` (GitHub Pages CDN) из pod-сети кластера недоступны
  (TLS handshake timeout), при этом сам `github.com` доступен.
- `k8s/helm.git` — чарт `sadko_first`: реверс-прокси, каждый = Service +
  EndpointSlice + Ingress.
- репа `metallb` — манифесты MetalLB + конфигурация BGP (пулы, peer, advertisement).
- `k8s/democratic-csi.git` — вендоренный чарт democratic-csi **0.15.1**
  (`democratic-csi.github.io` — тот же GitHub Pages CDN). Образ драйвера
  v1.9.5 задан в `truenas-iscsi.yaml`. Обновление: `helm pull --version <новая> --untar`
  на ansible13 → закоммитить поверх.

## Устройство чарта sadko_first: один values-файл = один контроллер

```
sadko_first/
├── Chart.yaml
├── templates/               # общие шаблоны для всех контуров
├── values.yaml              # ТОЛЬКО общие настройки (namespace, tlsSecret,
│                            # defaults) + proxies: [] и internal: [] (ПУСТЫЕ)
└── values-proxy-150.yaml    # прод: ingressClassName nginx + все прокси + internal
```

Правила:

- Базовый `values.yaml` helm читает всегда; списки в нём **обязаны быть пустыми** —
  вся конкретика живёт в файлах контроллеров.
- Файл контроллера подключается только явно, через `helm.valueFiles` в Application.
  Посторонние `values-*.yaml` в каталоге ни на что не влияют.
- При наложении values-файлов списки **заменяются целиком** (не сливаются) —
  поэтому контуры не пересекаются.
- Новый контур (тест, контроллеры замены прокси-серверов) =
  `values-<имя>.yaml` в чарте + `<имя>.yaml` Application здесь. Прод не трогается.
- **Добавить/изменить прокси** = правка values-файла нужного контроллера + push.
  Никаких `helm upgrade` руками.

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
| `configs.params."server.insecure": true` | ArgoCD-server отдаёт HTTP без TLS — терминация TLS на ingress (хост argocd.sadkomed.ru идёт через proxy-150) |
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
в namespace `default`. На чистом кластере создать до синка proxy-150:

```bash
kubectl create secret tls sadkomed-tls -n default --cert=fullchain.pem --key=privkey.pem
```

## 3a. Секрет democratic-csi (API-ключ TrueNAS)

Конфиг драйвера содержит API-ключ TrueNAS, поэтому в git его нет — Secret
создаётся руками до синка `truenas-iscsi`. Файл с ключом создавать вне git-реп
и удалить сразу после создания секрета.

```bash
kubectl create namespace democratic-csi
kubectl label namespace democratic-csi pod-security.kubernetes.io/enforce=privileged
kubectl -n democratic-csi create secret generic truenas-iscsi-driver-config \
  --from-file=driver-config-file.yaml=/root/driver-config.yaml && rm /root/driver-config.yaml
```

Шаблон `driver-config.yaml` (подставить ключ; `{{ parameters... }}` — шаблоны
самого democratic-csi, оставить как есть):

```yaml
driver: freenas-api-iscsi
instance_id:
httpConnection:
  protocol: https
  host: 10.69.0.251
  port: 443
  apiKey: "<API_KEY>"
  allowInsecure: true
zfs:
  datasetProperties:
    "org.freenas:description": "{{ parameters.[csi.storage.k8s.io/pvc/namespace] }}/{{ parameters.[csi.storage.k8s.io/pvc/name] }}"
  datasetParentName: ssd/k8s/iscsi/v
  detachedSnapshotsDatasetParentName: ssd/k8s/iscsi/s
  zvolCompression:
  zvolDedup:
  zvolEnableReservation: false      # тонкие тома; пул защищён ssd/_reserve
  zvolBlocksize: 8K                 # на весь драйвер, не на StorageClass
iscsi:
  targetPortal: "10.69.0.251:3260"
  targetPortals: []
  interface:
  namePrefix: csi-
  nameSuffix: "-k8s"
  targetGroups:
    - targetGroupPortalGroup: 1      # ID из midclt, не из UI
      targetGroupInitiatorGroup: 2
      targetGroupAuthType: None
      targetGroupAuthGroup:
  extentCommentTemplate: "{{ parameters.[csi.storage.k8s.io/pvc/namespace] }}/{{ parameters.[csi.storage.k8s.io/pvc/name] }}"
  extentInsecureTpc: true
  extentXenCompat: false
  extentDisablePhysicalBlocksize: false
  extentBlocksize: 4096
  extentRpm: "SSD"
  extentAvailThreshold: 0
```

Важно: TrueNAS 25.04 отзывает API-ключ при использовании по http — только `https` +
`allowInsecure: true`.

## 4. Применение Application-ов (порядок важен)

```bash
kubectl apply -f metallb.yaml              # 1. MetalLB: пулы, BGP-пиринг
kubectl apply -f ingress-nginx.yaml        # 2. прод-контроллер -> 192.168.253.150
kubectl apply -f ingress-nginx-test.yaml   # 3. тестовый -> 192.168.253.69
kubectl apply -f proxy-150.yaml            # 4. все прокси прод-контроллера
kubectl apply -f truenas-iscsi.yaml        # 5. democratic-csi (нужен секрет из 3a)
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
kubectl -n democratic-csi get pods -o wide              # 1 controller + node на каждом воркере
kubectl get csidriver,sc | grep truenas
```

---

# Эксплуатация

- **Добавить/поменять прокси** — правка `values-proxy-150.yaml` (или файла другого
  контура) в репе `k8s/helm.git`, push → ArgoCD синкает сам. `helm upgrade`/`helm
  uninstall` не использовать: selfHeal откатит ручные изменения. В `helm list`
  приложения ArgoCD не видны — это нормально: Арго не создаёт helm-релизов, он
  рендерит чарт и применяет манифесты напрямую; история — в UI (History and Rollback).
- **Правка общих файлов** (`templates/*`, базовый `values.yaml`) применяется ко
  ВСЕМ контурам сразу — сначала обкатать на тестовом.
- **Удаление Application** — только **Non-cascading** (в UI) или `kubectl delete
  application <имя> -n argocd`. Foreground/Background каскадно удаляют все ресурсы
  приложения — на проде это положит трафик.
- **Переименование Application** — удалить старое (Non-cascading!), применить файл
  с новым именем: ресурсы переживают это без прерывания, новое приложение их
  усыновляет. Держать оба одновременно нельзя — будут драться за tracking-лейбл.
- **Обновить ingress-nginx** — положить новую версию чарта в `k8s/ingress-nginx.git`
  (helm pull новой версии → закоммитить поверх), push. Сначала обкатать на
  ingress-nginx-test (оба смотрят в одну репу — обновлять через отдельную ветку и
  `targetRevision`, либо последовательно).
- **Обновить ArgoCD** — `helm upgrade ... --version <новая>` с этим же values-файлом.
- **Тонкость EndpointSlice:** kubernetes дописывает эндпойнтам `conditions: {}`,
  поэтому в Application прокси стоит `ignoreDifferences` на
  `.endpoints[].conditions` — без него все слайсы вечно OutOfSync.
- **Новые IP-пулы MetalLB** — в репе metallb; BGPAdvertisement `bgp-253`
  перечисляет пулы явно (если убрать поле `ipAddressPools` — анонсируются все).
- **Запуск чарта без ArgoCD** (новый кластер, DR — НЕ боевой кластер под Арго):
  `helm install proxy-150 ./sadko_first -n default -f values-proxy-150.yaml`;
  предпросмотр рендера: `helm template ... -f <values-файл>`.
