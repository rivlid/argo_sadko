# argo_sadko — GitOps-конфигурация кластера

Эта репа — источник правды по ArgoCD: values для установки самого ArgoCD и все
Application-манифесты. Кластер управляется по схеме:

```
git (gitlab.sadkomed.ru) ──> ArgoCD ──> кластер
```

Руками в кластере ничего не меняется. Любое изменение = коммит в соответствующую
репу, ArgoCD накатывает сам (авто-sync включён у всех приложений).

## Состав репы

```
argo_sadko/
├── README.md
├── argocd-helm-values.yaml   # values для установки/обновления самого ArgoCD
├── infra/                    # Application-ы инфраструктуры — применены в кластере
└── legacy/                   # старая схема 192.168.253.x — НЕ ПРИМЕНЯТЬ
```

| Файл | Что это |
|------|---------|
| `argocd-helm-values.yaml` | Helm values для установки/обновления самого ArgoCD |
| `infra/metallb.yaml` | Application `metallb-config`: MetalLB целиком (репа `metallb`, path `sadkomed_bgp`) |
| `infra/ingress-nginx-20.yaml` | Application: **прод** ingress-контроллер, class `nginx-20`, IP 192.168.69.20, 3 реплики |
| `infra/ingress-nginx-69.yaml` | Application: **тестовый** контроллер, class `nginx-69`, IP 192.168.69.69, 2 реплики |
| `infra/proxy-20.yaml` | Application: маршруты прод-контроллера (чарт `sadko_first`, `values-proxy-20.yaml`) |
| `infra/proxy-69-69.yaml` | Application: маршруты тестового контроллера (чарт `sadko_first`, `values-proxy-69-69.yaml`) |
| `infra/truenas-iscsi.yaml` | Application: democratic-csi (TrueNAS iSCSI, драйвер `freenas-api-iscsi`), namespace `democratic-csi`, StorageClass `truenas-iscsi` (RWO, не default) |

### legacy/ — не применять

`legacy/ingress-nginx.yaml`, `legacy/ingress-nginx-test.yaml`, `legacy/proxy-150.yaml`,
`legacy/proxy-69.yaml` — первая схема (контроллеры на 192.168.253.150 / 192.168.253.69,
классы `nginx` / `nginx-test`). В кластере **не применены**, хранятся как справка.
`kubectl apply -f legacy/` поднимет второй прод-контроллер и продублирует все прокси.
Связанные хвосты той же схемы: values-файлы `values-proxy-150.yaml` / `values-proxy-69.yaml`
в чарте `sadko_first` и пул `pool-253-150` в репе metallb.

## Текущее состояние кластера (2026-10-08)

Application-ы в ArgoCD (все Synced/Healthy): `ingress-nginx-20`, `ingress-nginx-69`,
`metallb-config`, `proxy-20`, `proxy-69-69`, `truenas-iscsi`.

| Контур | Контроллер | Класс | IP (пул MetalLB) | Маршруты |
|--------|-----------|-------|------------------|----------|
| прод | `ingress-nginx-20` | `nginx-20` | 192.168.69.20 (`pool-69-20`) | `proxy-20` → `values-proxy-20.yaml`: ~60 внешних прокси + internal (headlamp, grafana-k8s, argocd) |
| тест | `ingress-nginx-69` | `nginx-69` | 192.168.69.69 (`pool-69-69`) | `proxy-69-69` → `values-proxy-69-69.yaml`: sema, elma-test + internal (argo-test) |

### Как связаны MetalLB, контроллер и маршруты

Явных ссылок «пул ↔ контроллер ↔ прокси» нет — всё вяжется неявно:

1. **Контроллер → пул MetalLB — по IP.** Аннотация `metallb.io/loadBalancerIPs: 192.168.69.20`
   на Service контроллера; MetalLB сам находит пул, в `addresses` которого попадает адрес.
   Не попал ни в один пул → `EXTERNAL-IP <pending>`.
2. **Пул → BGP-анонс — по имени пула** в `BGPAdvertisement bgp-253.spec.ipAddressPools`.
   Пула нет в списке → IP выдан, но снаружи недоступен.
3. **Ingress → контроллер — по имени IngressClass:** `ingressClassName` в values-файле
   маршрутов = `ingressClassResource.name` в values контроллера. Каждый контроллер берёт
   только свой IngressClass по `controllerValue` (`k8s.io/ingress-nginx-20` / `-69`).
4. **Ingress → Service** — по имени, в namespace `default`.
5. **Service → backend:** для `proxies:` — EndpointSlice с меткой
   `kubernetes.io/service-name: <имя>-external` и IP внешнего сервера; для `internal:` —
   Service `ExternalName` на `<serviceName>.<serviceNamespace>.svc.cluster.local`.

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
├── values-proxy-20.yaml     # прод: ingressClassName nginx-20 + все прокси + internal
├── values-proxy-69-69.yaml  # тест: ingressClassName nginx-69
├── values-proxy-150.yaml    # legacy, не подключён ни к одному Application
└── values-proxy-69.yaml     # legacy, не подключён ни к одному Application
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

## Публикация приложений из кластера и TLS

Ingress может брать TLS-секрет **только из своего namespace**. Секрет `sadkomed-tls`
(wildcard) один и лежит в `default` — там же, где все Ingress-ы чарта `sadko_first`.

**Принятый вариант:** приложение живёт в своём namespace, а его Ingress создаётся в
`default` через список `internal:` values-файла нужного контроллера (Service `ExternalName`
→ сервис приложения). Пример для тестового контура:

```yaml
internal:
  - name: planka-test
    host: planka-test.sadkomed.ru
    serviceName: planka
    serviceNamespace: planka-test
    backendPort: 1337
```

Минус: новое приложение = манифесты приложения + строка в `internal:` (две правки);
при удалении приложения строку надо убрать отдельно.

**Отложено — `default-ssl-certificate` на контроллерах.** Когда приложений станет много
(или появится ApplicationSet «каталог = приложение»), удобнее, чтобы Ingress лежал в
каталоге самого приложения. Тогда в values контроллера добавляется:

```yaml
controller:
  extraArgs:
    default-ssl-certificate: default/sadkomed-tls
```

и Ingress приложения указывает `tls.hosts` **без** `secretName` — контроллер подставит
wildcard из `default`. Секрет по namespace-ам не копировать: при замене сертификата
придётся обновлять каждую копию. Порядок: сначала `ingress-nginx-69`, проверка
(`openssl s_client -connect 192.168.69.69:443 -servername nosuchhost.sadkomed.ru` →
`CN = *.sadkomed.ru`), потом `ingress-nginx-20`. Переход с `internal:` на свой Ingress —
перенести Ingress в каталог приложения и убрать строку из `internal:`.

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
| `configs.params."server.insecure": true` | ArgoCD-server отдаёт HTTP без TLS — терминация TLS на ingress (хост argocd.sadkomed.ru идёт через proxy-20, `internal:`) |
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
в namespace `default`. На чистом кластере создать до синка proxy-20 / proxy-69-69:

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
kubectl apply -f infra/metallb.yaml            # 1. MetalLB: пулы, BGP-пиринг
kubectl apply -f infra/ingress-nginx-20.yaml   # 2. прод-контроллер -> 192.168.69.20
kubectl apply -f infra/ingress-nginx-69.yaml   # 3. тестовый -> 192.168.69.69
kubectl apply -f infra/proxy-20.yaml           # 4. маршруты прод-контроллера
kubectl apply -f infra/proxy-69-69.yaml        # 5. маршруты тестового контроллера
kubectl apply -f infra/truenas-iscsi.yaml      # 6. democratic-csi (нужен секрет из 3a)
```

Namespace-ы контроллеров создаются сами (`CreateNamespace=true`). Каталог `legacy/`
не применять.

Проверка после каждого шага:

```bash
kubectl get application -n argocd                       # все Synced/Healthy
kubectl get svc -n ingress-nginx-20                     # EXTERNAL-IP 192.168.69.20
kubectl get svc -n ingress-nginx-69                     # EXTERNAL-IP 192.168.69.69
kubectl get ingress -n default | wc -l                  # ~60 ингрессов
curl -kI https://192.168.69.20 -H "Host: grafana.sadkomed.ru"
kubectl -n democratic-csi get pods -o wide              # 1 controller + node на каждом воркере
kubectl get csidriver,sc | grep truenas
```

---

# Эксплуатация

- **Сверка git и кластера** — `kubectl diff -f infra/`: пустой вывод = Application-ы в
  кластере совпадают с файлами (никто не правил через UI мимо git).
- **Добавить/поменять прокси** — правка `values-proxy-20.yaml` (или файла другого
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
  ingress-nginx-69 (все контроллеры смотрят в одну репу — обновлять через отдельную ветку и
  `targetRevision`, либо последовательно).
- **Обновить ArgoCD** — `helm upgrade ... --version <новая>` с этим же values-файлом.
- **Тонкость EndpointSlice:** kubernetes дописывает эндпойнтам `conditions: {}`,
  поэтому в Application прокси стоит `ignoreDifferences` на
  `.endpoints[].conditions` — без него все слайсы вечно OutOfSync.
- **Новые IP-пулы MetalLB** — в репе metallb; BGPAdvertisement `bgp-253`
  перечисляет пулы явно (если убрать поле `ipAddressPools` — анонсируются все).
- **Запуск чарта без ArgoCD** (новый кластер, DR — НЕ боевой кластер под Арго):
  `helm install proxy-20 ./sadko_first -n default -f values-proxy-20.yaml`;
  предпросмотр рендера: `helm template ... -f <values-файл>`.

## democratic-csi (TrueNAS iSCSI)

Драйвер `freenas-api-iscsi` (только API, без SSH), TrueNAS `f99-fs02` (10.69.0.251),
пул `ssd` (зеркало 2× Intel D3-S4620), тома в `ssd/k8s/iscsi/v`.
Ресурсы: `deploy/truenas-iscsi-democratic-csi-controller` (1 шт),
`ds/truenas-iscsi-democratic-csi-node` (только воркеры — у мастеров taint, iSCSI-клиента там нет).

**Что и зачем в values (`truenas-iscsi.yaml`):**

| Ключ | Зачем |
|------|-------|
| `driver.existingConfigSecret` | конфиг драйвера с API-ключом не в git (см. 3a) |
| `controller/node.driver.image.tag: v1.9.5` | не `latest` — обновление только осознанно |
| `controller.externalSnapshotter.enabled: false` | VolumeSnapshot CRD/контроллера в кластере нет — sidecar сыпал бы ошибками |
| `volumeSnapshotClasses: []` | то же |

**Тонкости:**

- `zvolBlocksize` (8K, под страницы Postgres) задаётся **на весь драйвер** в секрете,
  не на StorageClass. Меняется только для новых томов; существующие — миграцией.
- Образы: драйвер с `ghcr.io`, sidecar'ы с `registry.k8s.io`. Первый деплой/обновление
  образа тянется на все 21 ноду из интернета — ~15–20 мин в `ContainerCreating`, это норма.
- Расширение PVC: zvol растёт сразу, ФС в поде — через 1–3 мин (kubelet, `resize2fs`).
  Пока не прошло — PVC показывает старый размер. Под не перезапускается.
- Удаление PVC (`reclaimPolicy: Delete`) удаляет zvol, iSCSI target и extent на TrueNAS.
- **TrueNAS 25.04 отзывает API-ключ при обращении по http** — только `https` + `allowInsecure`.
- **REST API TrueNAS удаляется в 26.04.** Перед апгрейдом TrueNAS до 26.x — сначала новая
  версия democratic-csi (или переход на truenas-csi), иначе драйвер перестанет работать.
  https://github.com/democratic-csi/democratic-csi/issues/509
- Обновить чарт: на ansible13 `helm pull democratic-csi/democratic-csi --version <новая> --untar`
  → закоммитить поверх в `k8s/democratic-csi.git`.

**Эталонные замеры** (2026-10-07, fio из пода, 8k randwrite QD1 `--sync=dsync`):

| | p50 | p99 | IOPS |
|---|---|---|---|
| PVC `volumeMode: Block` | 0,55 мс | 1,71 мс | 1693 |
| PVC ext4, файл предварительно записан | 0,59 мс | 1,86 мс | 1565 |
| seq write 1M QD32 | — | — | ~420 МБ/с (предел записи SSD-зеркала) |

Грабли при тестах: fio по умолчанию создаёт файл через `fallocate` → блоки ext4
`unwritten`, первая запись в каждый блок коммитит журнал → два режима (0,6 / 1,7 мс),
p99 хуже. Перед тестом файл заполнять записью (`--fallocate=none`, seq write). `--sync=1`
(O_SYNC) на ext4 ещё в ~3 раза хуже — Postgres так не пишет, мерить `--sync=dsync`.

**TrueNAS — сетевая карта:** `ens4f0np0` (Intel i40e). При дефолтном RX ring 512 были
потери (`rx_missed_errors`). Поднято до 4096: `ethtool -G ens4f0np0 rx 4096 tx 4096`,
продублировано в System → Advanced → Init/Shutdown Scripts (Post Init). Контроль:
`ethtool -S ens4f0np0 | grep -E 'rx_missed_errors|port.rx_discards'` — под нагрузкой не
должны расти (база на 2026-10-07: 72291 / 2880). Растут — поднять до 8160 (максимум).

**Бэкап томов:** `ssd/k8s` снапшотится (Periodic Snapshot Task) и реплицируется
на другой NAS.

**Риски (не решены):**

- TrueNAS — единая точка отказа: его перезагрузка подвешивает все поды с томами.
- Только RWO. Для RWX нужен отдельный NFS-драйвер.
