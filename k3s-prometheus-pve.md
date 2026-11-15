# Мониторинг Proxmox VE и Kubernetes (k3s) через Prometheus, Grafana и Alertmanager

Данный репозиторий содержит инструкцию и манифесты для развертывания стека мониторинга в кластере k3s. Стек собирает метрики с независимых узлов Proxmox VE (через API), мониторит состояние подов и отправляет алерты в Telegram.

## 📋 Компоненты
*   **k3s** — легковесный Kubernetes.
*   **kube-prometheus-stack (Helm)** — Prometheus, Alertmanager, Grafana, Prometheus Operator, kube-state-metrics, node-exporter.
*   **prometheus-pve-exporter** — экспортёр метрик Proxmox VE (развернут отдельно для каждого узла PVE).
*   **Telegram Bot** — канал доставки алертов.

## 🚀 Пошаговая установка

### Шаг 1. Подготовка Proxmox VE
Для каждого узла PVE создаем read-only пользователя и API-токен.
> **Важно:** Для standalone-узлов (не в кластере) токены не синхронизируются. Создайте пользователя **на каждом** узле PVE отдельно.

Выполните на каждом узле PVE (через SSH):
```bash
pveum user add pve-exporter@pve
pveum acl modify / -user pve-exporter@pve -role PVEAuditor
# КРИТИЧЕСКИ ВАЖНО: флаг --privsep 0, иначе токен не унаследует права пользователя!
pveum user token add pve-exporter@pve exporter-token --privsep 0
```
**Сохраните значение `value`** — оно показывается только один раз.

### Шаг 2. Установка k3s и Helm
Установите k3s (с учетом сетевых особенностей вашего региона используйте зеркала):
```bash
curl -sfL https://get.k3s.io | sh -
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```
Установите Helm (скачайте бинарник с зеркала, если GitHub недоступен):
```bash
curl -kLO https://debian.sbor.net/mirrors/github/helm/helm/v4.3.1/helm-linux-amd64.tar.gz
tar -zxvf helm-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### Шаг 3. Установка стека мониторинга (kube-prometheus-stack)
Создайте файл `values.yaml`. В нем мы сразу задаем лимиты ресурсов (чтобы избежать OOMKilled Grafana), настраиваем NodePort и **обязательно добавляем receiver `null`** для оператора.

```yaml
grafana:
  # Для лабораторной работы пароль можно указать здесь.
  # В продакшене используйте существующий Secret (grafana.admin.existingSecret)
  adminPassword: "ВашНадёжныйПароль"
  service:
    type: NodePort
  persistence:
    enabled: true
    size: 1Gi  # или 5Gi, если планируете много дашбордов
  resources:
    requests: { memory: 256Mi, cpu: 100m }
    limits: { memory: 512Mi }

# Отключаем мониторинг компонентов, которых нет в k3s (иначе будут ложные алерты)
kubeControllerManager: { enabled: false }
kubeScheduler: { enabled: false }
kubeProxy: { enabled: false }
kubeEtcd: { enabled: false }

alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'telegram-notifications'
      group_by: ['alertname']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
    receivers:
    # ФИКС ОШИБКИ: Operator требует наличия receiver "null" по умолчанию
    - name: 'null'                     
    - name: 'telegram-notifications'
      telegram_configs:
      # Токен указывается БЕЗ кавычек и НЕ в base64
      - bot_token: 123456789:AAExampleTokenHere
        chat_id: 123456789             # Для личных чатов положительное число
        send_resolved: true
        parse_mode: 'HTML'
```
Установка:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kps prometheus-community/kube-prometheus-stack -n monitoring --create-namespace -f ~/k3s-lab/values.yaml
```

### Шаг 4. Деплой pve-exporter (для каждого узла PVE)
Для каждого узла создаем свой манифест. Пример для узла `192.168.231.20` (файл `pve-exporter.yaml`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pve-exporter-auth
  namespace: monitoring
type: Opaque
stringData:
  PVE_USER: "pve-exporter@pve"
  PVE_TOKEN_NAME: "exporter-token"
  PVE_TOKEN_VALUE: "ВСТАВЬТЕ_ТОКЕН_ИЗ_ШАГА_1"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pve-exporter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels: { app: pve-exporter }
  template:
    metadata:
      labels: { app: pve-exporter }
    spec:
      containers:
      - name: pve-exporter
        image: prompve/prometheus-pve-exporter:latest
        ports:
        - containerPort: 9221
          name: http
        env:
        - name: PVE_USER
          valueFrom: { secretKeyRef: { name: pve-exporter-auth, key: PVE_USER } }
        - name: PVE_TOKEN_NAME
          valueFrom: { secretKeyRef: { name: pve-exporter-auth, key: PVE_TOKEN_NAME } }
        - name: PVE_TOKEN_VALUE
          valueFrom: { secretKeyRef: { name: pve-exporter-auth, key: PVE_TOKEN_VALUE } }
        - name: PVE_VERIFY_SSL
          value: "false"
---
apiVersion: v1
kind: Service
metadata:
  name: pve-exporter
  namespace: monitoring
  labels: { app: pve-exporter }
spec:
  selector: { app: pve-exporter }
  ports:
  - name: http
    port: 9221
    targetPort: 9221
```
> Для остальных узлов PVE копируем этот файл, меняя имена ресурсов (`pve-exporter-pve2` и т.д.), IP в ServiceMonitor и значения в Secret.

### Шаг 5. Настройка сбора метрик (ServiceMonitor)
Создаем ServiceMonitor для каждого PVE. **Критически важно** явно передать `target` (IP PVE), иначе экспортёр будет стучаться в `localhost`.

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: pve-exporter
  namespace: monitoring
  labels:
    release: kps
spec:
  selector:
    matchLabels: { app: pve-exporter }
  endpoints:
  - port: http
    path: /pve
    params:
      module: [default]
      cluster: ["1"]
      node: ["1"]
      target: ["192.168.231.20"]   # <--- IP конкретного PVE
    interval: 60s
    scrapeTimeout: 45s
    relabelings:
    - targetLabel: pve_name
      replacement: "PVE-Avito-Lab" # Человекочитаемое имя для дашбордов
      action: replace
```

### Шаг 6. Настройка алертов (PrometheusRule)
Создаем правила для Proxmox и Kubernetes (файл `alerts.yaml`):
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  namespace: monitoring
  labels:
    release: kps
spec:
  groups:
  - name: pve.rules
    rules:
    - alert: PveGuestDown
      expr: pve_up{id=~"qemu/.*"} == 0
      for: 2m
      labels: { severity: warning }
      annotations:
        summary: "VM {{ $labels.id }} недоступна в Proxmox"
    - alert: PveStorageLow
      expr: (pve_disk_usage_bytes{id=~"storage/.*"} / pve_disk_size_bytes{id=~"storage/.*"}) > 0.9
      for: 5m
      labels: { severity: warning }
      annotations:
        summary: "Хранилище {{ $labels.id }} заполнено > 90%"
  - name: k8s.rules
    rules:
    - alert: PodRestartingTooMuch
      expr: increase(kube_pod_container_status_restarts_total{namespace="default"}[10m]) > 2
      for: 1m
      labels: { severity: warning }
      annotations:
        summary: "Pod {{ $labels.pod }} перезапускается слишком часто"
```

### 📊 Визуализация в Grafana (Дашборды)

После успешного сбора метрик перед вами встанет вопрос их отображения. В экосистеме Grafana есть готовые решения, но стоит учитывать специфику ваших метрик (кастомная метка `pve_name`).

#### Вариант 1: Готовый дашборд от сообщества (с ограничениями)
Вы можете импортировать популярный дашборд для Proxmox через `kube-prometheus-stack`. 
*   **ID дашборда:** `10347` (Proxmox via Prometheus) или `24550` (Proxmox VE - pve-exporter).
*   **Как импортировать:** В Grafana перейдите в **Dashboards → New → Import**, введите ID и выберите ваш Prometheus.
*   **⚠️ Важное замечание:** Готовые дашборды часто используют метку `instance` (которая у нас содержит IP пода-экспортёра) в качестве фильтра. Из-за этого в выпадающем списке вы будете видеть `10.42.0.x:9221` вместо имен узлов. Вы можете либо смириться с этим, либо переписать запросы в панелях, заменив `instance=~"$instance"` на `pve_name=~"$instance"`.

#### Вариант 2: Создание собственного дашборда (Рекомендуемый)
Так как вы уже добавили метку `pve_name` в `ServiceMonitor`, проще всего собрать свой лаконичный дашборд, который будет работать идеально.

**1. Создайте переменную для фильтрации по узлам:**
*   Перейдите в **Dashboard settings → Variables → New variable**.
*   **Name:** `pve_name`
*   **Type:** `Query`
*   **Query type:** `Label values`
*   **Label:** `pve_name` (очистите поле Metric, если это возможно).
*   *Теперь в выпадающем списке будут красивые имена (PVE-Avito-Lab и т.д.).*

**2. Добавьте базовые панели (Time series):**

*   **Загрузка CPU (в %):**
    ```promql
    pve_cpu_usage_ratio{pve_name=~"$pve_name", id=~"node/.*"} * 100
    ```
    *(Установите Unit: Percent (0-100), Legend: `{{pve_name}}`)*

*   **Использование памяти (в ГБ):**
    ```promql
    pve_memory_usage_bytes{pve_name=~"$pve_name", id=~"node/.*"} / 1024 / 1024 / 1024
    ```
    *(Установите Unit: Data (IEC) -> Gibibytes, Legend: `{{pve_name}}`)*

*   **Заполнение хранилищ (в %):**
    ```promql
    (pve_disk_usage_bytes{pve_name=~"$pve_name", id=~"storage/.*"} / pve_disk_size_bytes{pve_name=~"$pve_name", id=~"storage/.*"}) * 100
    ```
    *(Установите Unit: Percent (0-100), Legend: `{{id}}`)*

*   **Статус ВМ и узлов (Table):**
    ```promql
    pve_up{pve_name=~"$pve_name"}
    ```
    *(Визуализация: Table, Legend: `{{pve_name}} - {{id}}`)*

> **Совет:** Не забудьте включить `grafana.persistence.enabled: true` в `values.yaml`, иначе все ваши созданные дашборды исчезнут после `helm upgrade` или перезапуска пода Grafana.

## 🛠 Решенные проблемы (Troubleshooting)

В процессе настройки были выявлены и устранены следующие ошибки:

1.  **Ошибка `pve-exporter` `Connection refused` (localhost:8006)**
    *   *Причина:* Экспортёр по умолчанию стучался в localhost, игнорируя переменную `PVE_HOST`.
    *   *Решение:* В `ServiceMonitor` в блок `params` жестко добавлен параметр `target: ["<IP_PVE>"]`.
2.  **Ошибка `undefined receiver "null" used in route` в Prometheus Operator**
    *   *Причина:* Чарт `kube-prometheus-stack` использует receiver `null` для сброса алертов, но при переопределении блока `config` он теряется.
    *   *Решение:* В `values.yaml` в список `receivers` добавлена пустая заглушка `- name: 'null'`.
3.  **Ложные алерты `KubeControllerManagerDown` в k3s**
    *   *Причина:* В k3s нет отдельных подов для controller-manager и scheduler, Prometheus их не видит.
    *   *Решение:* В `values.yaml` отключены соответствующие компоненты (`kubeControllerManager.enabled: false` и т.д.).
4.  **`OOMKilled` пода Grafana**
    *   *Причина:* Дефолтных лимитов памяти (256Mi) не хватает для работы sidecar-контейнеров.
    *   *Решение:* Лимиты памяти Grafana и её sidecar увеличены через `values.yaml`.
```

