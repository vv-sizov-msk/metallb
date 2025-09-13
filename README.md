# MetalLB (L2) — Kustomize overlay для Argo CD (multi-source)

Этот репозиторий содержит **минимальные манифесты конфигурации MetalLB** (L2 mode):  
`IPAddressPool` + `L2Advertisement`, оформленные как **Kustomize overlay**.  
Используется совместно с **Argo CD Application (multi-source)**, где:
- Source #1: Helm-чарт `metallb` из официального Helm-репозитория;
- Source #2: этот Git-репозиторий с Kustomize-оверлеем.

> Почему так? Чарт ставит контроллер/спикер и CRD, а конфиг (пулы/объявления) живёт отдельно в Git — прозрачно, просто и GitOps-дружелюбно.

---

## Структура
```plaintext
manifests/metallb-resources/
├─ kustomization.yaml
├─ ipaddresspool.yaml
└─ l2advertisement.yaml
```

- `ipaddresspool.yaml` — пул адресов для выдачи `LoadBalancer`-сервисам.  
- `l2advertisement.yaml` — публикация адресов в L2 (ARP/ND).  
- `kustomization.yaml` — склеивает ресурсы и задаёт аннотации для корректного порядка применения.

> Принята договорённость: **в первой строке каждого манифеста — комментарий с названием файла**.

---

## Предпосылки

1. MetalLB разворачивается **отдельно** Helm-чартом через тот же Argo CD Application (multi-source).  
2. Namespace по умолчанию — `kube-system`.  
3. В `Application.spec.syncPolicy.syncOptions` рекомендовано:
   ```yaml
   syncOptions:
     - CreateNamespace=true
     - ServerSideApply=true
4. Для гарантии порядка: конфиги (Pool/Advertisement) помечены argocd.argoproj.io/sync-wave: "1" — применяются после установки CRD/контроллера чарта (wave 0).

## Быстрый старт

Отредактируйте диапазон IP под вашу сеть в ipaddresspool.yaml:

spec:
  addresses:
    - 192.168.100.220-192.168.100.230  # ⚠️ диапазон вне DHCP


Закоммитьте изменения и пушните в публичный GitHub.

В Argo CD Application укажите этот репозиторий как второй source:
```yaml
sources:
  - repoURL: https://metallb.github.io/metallb
    chart: metallb
    targetRevision: 0.15.x
  - repoURL: https://github.com/<USER>/<REPO>.git
    targetRevision: main
    path: manifests/metallb-resources
```

Нажмите SYNC в Argo CD.

Проверьте готовность:
```bash
kubectl -n metallb-system get pods
kubectl get ipaddresspools.metallb.io -A
kubectl get l2advertisements.metallb.io -A
```

Пример тестового сервиса
```yaml
# svc-nginx-lb.yaml — пример выдачи IP из пула
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
  namespace: default
  annotations:
    # укажи пул явно, если используется несколько
    metallb.universe.tf/address-pool: lan-pool
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

После kubectl apply -f svc-nginx-lb.yaml сервис получит IP из пула.

### Лучшие практики
### Сеть и адреса

- Выбирайте `диапазон вне DHCP` и зарезервируйте его в вашем маршрутизаторе/ДНС.

- Держите диапазон `компактным` (10–20 адресов для домашнего кластера достаточно).

- Для разных подсетей/ВЛАН делайте `отдельные пулы` и объявления (L2Advertisement).

- В L2Advertisement можно `ограничить ноды` по меткам:
```yaml
spec:
  ipAddressPools: [ lan-pool ]
  nodeSelectors:
    - matchLabels:
        node-pool: apps
```

- Если L2 доменов несколько — никогда не объявляйте один и тот же IP в двух разных L2.

### Порядок и синхронизация

- Используйте sync waves для конфигов (wave 1), чтобы применялись после установки CRD из чарта.

- Включите ServerSideApply — меньше конфликтов при апдейтах.

- Если конфиг подхватывается до установки CRD — просто повторный SYNC решает; waves обычно достаточно.

### Наблюдаемость

- В values чарта включите метрики:
```yaml
prometheus:
  serviceMonitor:
    enabled: true
```

Это позволит Prometheus забрать метрики MetalLB.

- Полезные команды:
```bash
kubectl -n metallb-system logs deploy/metallb-controller
kubectl -n metallb-system logs ds/metallb-speaker -c speaker
kubectl describe svc -n <ns> <name>
```
### Безопасность

- Держите пулы `минимально необходимыми`.

- Разделяйте пулы между окружениями (dev/stage/prod) и пространствами имён через аннотации на Service.

- В частных сетях с pfSense/iptables следите за ARP-таймутами и не используйте один и тот же IP за NAT/Port-Forward одновременно.

### Трюки и тонкости

- Явный выбор пула — аннотация metallb.universe.tf/address-pool: <name> на Service.

- Выдача IP только по запросу — держите `spec.allocateLoadBalancerNodePorts`/`externalTrafficPolicy` согласно политике трафика.

- `Ограничение интерфейсов` для L2 (при необходимости):
```yaml
spec:
  ipAddressPools: [ lan-pool ]
  interfaces: [ "eth0" ]
```
Локальная проверка (без применения)
```bash
kubectl kustomize manifests/metallb-resources | kubectl apply --dry-run=client -f -
```
### Частые проблемы

- `no matches for kind "IPAddressPool"` — CRD ещё не применены. Решение: дождаться установки чарта / повторить SYNC (или используйте sync-waves).

- IP не выдаётся — проверьте:

    - правильный type: LoadBalancer;

    - аннотацию пула (если пулов несколько);

    - что адрес свободен и не в DHCP-диапазоне;

    - логи metallb-controller/metallb-speaker.

- Адрес не пингуется — проверяйте L2-домены, ARP-записи на шлюзе, firewall.

### Полезные команды
```bash
# События в пространстве MetalLB
kubectl -n metallb-system get events --sort-by=.lastTimestamp

# Состояние ресурсов MetalLB
kubectl get ipaddresspools.metallb.io -A -o wide
kubectl get l2advertisements.metallb.io -A -o yaml

# Проверка назначения IP сервису
kubectl -n <ns> get svc <name> -o wide
```
