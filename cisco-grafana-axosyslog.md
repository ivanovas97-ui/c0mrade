
# 📊 Runbook: Мониторинг логов Cisco (AxoSyslog + Loki + Grafana)

Данный документ описывает процесс развертывания, настройки и эксплуатации системы централизованного сбора логов (Syslog) с устройств Cisco.

**Стек технологий:**
*   **ОС:** Ubuntu 24.04
*   **Контейнеризация:** Docker, Docker Compose
*   **Хранилище логов:** Grafana Loki
*   **Визуализация:** Grafana
*   **Коллектор:** AxoSyslog (syslog-ng)
*   **Уведомления:** Telegram (через Grafana Alerting)

---

## 📑 Содержание
1. [Подготовка среды и установка Docker](#1-подготовка-среды-и-установка-docker)
2. [Развертывание Loki и Grafana](#2-развертывание-loki-и-grafana)
3. [Установка и настройка AxoSyslog](#3-установка-и-настройка-axosyslog)
4. [Настройка устройств Cisco](#4-настройка-устройств-cisco)
5. [Создание дашбордов в Grafana](#5-создание-дашбордов-в-grafana)
6. [Настройка системы алертов](#6-настройка-системы-алертов)
7. [Диагностика и устранение неисправностей](#7-диагностика-и-устранение-неисправностей)

---

## 1. Подготовка среды и установка Docker

Выполняется на чистой виртуальной машине (ВМ) с Ubuntu 24.04.

### 1.1. Установка Docker и Compose
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
```
*Проверка:* `docker --version` (должна быть версия 29.1.x) и `docker compose version` (должна быть версия 2.40.x).

### 1.2. Настройка DNS для Docker (Критично!)

**Внимание:** По умолчанию Docker в Ubuntu наследует локальный DNS (127.0.0.53), который недоступен внутри контейнеров. Это приведет к ошибке разрешения имен между Grafana и Loki (`Could not resolve host`).

Создайте или отредактируйте файл:
```bash
sudo nano /etc/docker/daemon.json
```
Вставьте:
```json
{
  "dns": ["8.8.8.8", "1.1.1.1"]
}
```
Перезапустите Docker:
```bash
sudo systemctl restart docker
```

---

## 2. Развертывание Loki и Grafana

### 2.1. Создание `docker-compose.yml`
Создайте директорию и файл конфигурации:
```bash
mkdir -p /opt/loki-stack && cd /opt/loki-stack
nano docker-compose.yml
```

Вставьте следующее содержимое (обратите внимание на проброс порта **9095** для gRPC):
```yaml
services:
  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
      - "9095:9095"  # Порт для gRPC (приема логов от AxoSyslog)
    restart: unless-stopped
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on:
      - loki
    restart: unless-stopped
    networks:
      - monitoring

volumes:
  grafana-data:

networks:
  monitoring:
    driver: bridge
```

Запуск:
```bash
docker compose up -d
```

### 2.2. Проверка работоспособности
1. Проверьте, что оба контейнера в одной сети:
   ```bash
   docker network inspect loki-stack_monitoring
   ```
   (В разделе `Containers` должны быть и `loki`, и `grafana`).
2. Проверьте, что Loki отвечает изнутри Grafana:
   ```bash
   docker exec -it grafana /bin/sh -c "curl -s http://loki:3100/ready"
   ```
   *(Если видите `Ingester not ready`, подождите 15 секунд и повторите).*

---

## 3. Установка и настройка коллектора AxoSyslog

AxoSyslog принимает syslog-сообщения от Cisco по UDP (порт 514) и отправляет их в Loki по gRPC (порт 9095).

### 3.1. Установка AxoSyslog
```bash
sudo -i
wget -qO - https://pkg.axoflow.io/axoflow-code-signing-pub.asc | gpg --dearmor > /usr/share/keyrings/axoflow-code-signing-pub.gpg
echo "deb [signed-by=/usr/share/keyrings/axoflow-code-signing-pub.gpg] https://pkg.axoflow.io/apt stable ubuntu-noble" | tee --append /etc/apt/sources.list.d/axoflow.list
apt update
apt install -y axosyslog
```

### 3.2. Конфигурация `/etc/syslog-ng/syslog-ng.conf`
Откройте файл и приведите к следующему виду:
```text
@version: 4.29
@include "scl.conf"

source s_cisco {
    network(
        transport("udp")
        port(514)
        flags(no-parse) # Обязательно для cisco-parser
    );
};

parser p_cisco {
    cisco-parser();
};

destination d_loki {
    loki(
        url("172.18.0.2:9095") # IP контейнера Loki (см. пункт 3.3)
        timestamp("current")   # Критично для Loki: используем время сервера
        labels(
            "job" => "cisco",
            "host" => "${HOST}"
        )
        batch-lines(1)         # Отправлять сразу
        batch-timeout(100)
    );
};

log {
    source(s_cisco);
    parser(p_cisco);
    destination(d_loki);
};
```

### 3.3. Получение IP-адреса Loki
Так как AxoSyslog работает на хосте, а Loki в Docker, нужно узнать внутренний IP:
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' loki
```
Впишите полученный IP (например, `172.18.0.2`) в параметр `url()` конфигурации AxoSyslog.

Перезапуск и проверка:
```bash
systemctl restart syslog-ng
systemctl status syslog-ng
ss -ulnp | grep 514  # AxoSyslog должен слушать порт 514
```

---

## 4. Настройка устройств Cisco

В режиме глобальной конфигурации (`configure terminal`) на роутере выполните:
```cisco
! Включаем временные метки
service timestamps log datetime msec localtime show-timezone

! Отправляем логи на IP ВМ (где установлен AxoSyslog)
logging host <IP_ВМ>

! Уровень логирования (informational достаточно для мониторинга)
logging trap informational

! Включаем логирование неудачных входов (для безопасности)
login on-failure log

! (Опционально) Отправка успешных входов
login on-success log
```

---

## 5. Создание дашбордов в Grafana

**Обязательное условие:** в Grafana добавлен источник данных Loki (URL: `http://loki:3100`).

### 5.1. Переменная для фильтрации (`device`)
1. В настройках дашборда (шестеренка) -> **Variables** -> **New variable**.
2. **Type**: `Query`. **Data source**: `Loki`.
3. **Query type**: `Label values`. **Label**: `host`.
4. **Name**: `device`.
5. Нажмите **Apply**.

### 5.2. Панель «Сводка по безопасности» (Table)
*   **Visualization**: `Table`
*   **Query (LogQL)**:
    ```logql
    {job="cisco", host=~"$device"} |= "LOGIN_FAILED"
      | regexp `\[user: (?P<user>[^\]]+)\] \[Source: (?P<src_ip>\d+\.\d+\.\d+\.\d+)\] \[localport: (?P<port>\d+)\] \[Reason: (?P<reason>[^\]]+)\]`
    ```
*   **Panel options**: Sort by `Time`, Descending.

### 5.3. Панель «Топ IP по неудачным входам» (Bar Gauge)
*   **Visualization**: `Bar Gauge`
*   **Query (LogQL)**:
    ```logql
    sum by (src_ip) (
      count_over_time(
        {job="cisco", host=~"$device"} |= "LOGIN_FAILED"
          | regexp `\[Source: (?P<src_ip>\d+\.\d+\.\d+\.\d+)\]`
        [5m]
      )
    )
    ```

### 5.4. Панель «Изменения конфигурации» (Logs)
*   **Visualization**: `Logs`
*   **Query**: `{job="cisco", host=~"$device"} |= "CONFIG_I"`

---

## 6. Настройка системы алертов (Telegram)

### 6.1. Подготовка Telegram
1. В Telegram найдите `@BotFather`, создайте бота (`/newbot`), получите **API Token**.
2. Создайте группу в Telegram, добавьте туда бота.
3. Напишите в группу любое сообщение. Откройте веб-версию Telegram, в URL группы найдите ID (начинается с минуса, например, `-4266674385`). Это **Chat ID**.

### 6.2. Создание точки контакта (Contact Point)
1. В Grafana: **Alerting** -> **Notification configuration** -> **Contact points** -> **+ Add contact point**.
2. **Name**: `Telegram-Cisco`. **Integration**: `Telegram`.
3. Вставьте **API Token** и **Chat ID** (с минусом).
4. Нажмите **Test**. Сохраните.

### 6.3. Создание правил (Alert Rules)
Перейдите в **Alerting** -> **Alert rules** -> **+ New alert rule**.

**Пример: События интерфейсов (Link Flaps)**
1. **Name**: `Cisco: Interface Events`.
2. **Query**: 
   ```logql
   sum(count_over_time({job="cisco", host=~".*"} |~ "LINK-|LINEPROTO-" [5m]))
   ```
3. **Condition**: `IS ABOVE 0`.
4. **Folder**: `Cisco Alerts`. **Evaluation group**: `cisco-logs`, Every `1m`.
5. **Configure no data and error handling**: 
   * Alert state if no data or all values are null: **`Normal`**.
6. **Configure notifications**: Выберите `Telegram-Cisco`.

*Повторите для других правил:*
*   **Аномалии среды**: `sum(count_over_time({job="cisco"} |~ "(?i)temp|temperature|fan|psu|power supply" [5m]))`
*   **Системные события**: `sum(count_over_time({job="cisco"} |~ "(%SYS-|%ENVMON-|%PLATFORM-)" != "CONFIG_I" [5m]))`

---

## 7. Диагностика и устранение неисправностей

В процессе настройки мы использовали следующие команды для поиска проблем. Если логи не доходят до Grafana, выполните эти шаги по порядку:

| Проблема | Команда диагностики | Решение |
| :--- | :--- | :--- |
| **Доходят ли пакеты до ВМ?** | `tcpdump -i any -n port 514 -c 5` | Если пакетов нет — проверьте настройки `logging host` на Cisco и firewall. |
| **Слушает ли AxoSyslog порт?** | `ss -ulnp \| grep 514` | Если пусто — проверьте `systemctl status syslog-ng`. |
| **Ошибки в AxoSyslog?** | `journalctl -u syslog-ng -n 20 --no-pager` | Ошибка `hpack_parser` означала отправку на HTTP-порт 3100 вместо gRPC 9095. |
| **Отладка AxoSyslog** | `systemctl stop syslog-ng`<br>`syslog-ng -Fevd` | Ищите строку `Loki batch delivered` (успех) или `Incoming log entry` (получение). |
| **Проверка метрик Loki** | `curl -s http://localhost:3100/metrics \| grep loki_distributor_lines_received_total` | Если счетчик растет — данные доходят до Loki. Если нет — проблема в AxoSyslog. |
| **Проблема с DNS в Docker** | `docker exec -it grafana /bin/sh -c "curl -s http://loki:3100/ready"` | Ошибка `Could not resolve host` — добавьте DNS в `/etc/docker/daemon.json`. |
| **Алерты шлют `DatasourceNoData`** | (В интерфейсе Grafana) | В настройках правила: `Alert state if no data...` -> `Normal`. |

---
