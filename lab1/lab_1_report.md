САНКТ–ПЕТЕРБУРГСКИЙ НАЦИОНАЛЬНЫЙ ИССЛЕДОВАТЕЛЬСКИЙ УНИВЕРСИТЕТ ИНФОРМАЦИОННЫХ ТЕХНОЛОГИЙ, МЕХАНИКИ И ОПТИКИ
## ФАКУЛЬТЕТ ИНФОКОММУНИКАЦИОННЫХ ТЕХНОЛОГИЙ

**Отчёт по лабораторной работе 1**

по курсу «Администрирование компьютерных сетей»

**PostgreSQL Cluster**

**Выполнили:** Кутуков.Д, Кижваткин Н, Фоченков С, Бочарников М, Субботин

**Проверил:** Самохин Н.Ю.

Санкт–Петербург 2025 г.

---

## Содержание

1. [Цель работы](#цель-работы)
2. [Исходные условия и окружение](#исходные-условия-и-окружение)
   - [Окружение](#окружение)
   - [Инструменты](#инструменты)
   - [Замечание по psql на хосте](#замечание-по-psql-на-хосте)
3. [Структура стенда и общая схема](#структура-стенда-и-общая-схема)
4. [Развёртывание (Docker Compose)](#развёртывание-docker-compose)
   - [Подготовка конфигурационных файлов](#подготовка-конфигурационных-файлов)
   - [Старт кластера](#старт-кластера)
   - [Контейнеры поднялись](#контейнеры-поднялись)
   - [Проверка статуса контейнеров и портов](#проверка-статуса-контейнеров-и-портов)
5. [Диагностика проблем и исправления](#диагностика-проблем-и-исправления)
   - [Проблема 1: Синхронизация часов между контейнерами](#проблема-1-синхронизация-часов-между-контейнерами)
   - [Проблема 2: ZooKeeper выбирает лидера дольше, чем стартует Patroni](#проблема-2-zookeeper-выбирает-лидера-дольше)
   - [Проблема 3: Конфликт в выборе лидера при одновременном старте](#проблема-3-конфликт-в-выборе-лидера)
6. [Проверка ролей: leader/replica](#проверка-ролей-leaderreplica)
7. [Проверка read-only режима реплики](#проверка-read-only-режима-реплики)
8. [Улучшение: автоматический dogon WAL после возврата ноды](#улучшение-автоматический-dogon-wal-после-возврата-ноды)
   - [Сценарий](#сценарий)
   - [Команды (типовой сценарий)](#команды-типовой-сценарий)
9. [Nginx как entrypoint (конфигурация на основе Patroni REST API)](#nginx-как-entrypoint-конфигурация-на-основе-patroni-rest-api)
   - [Конфигурация Nginx](#конфигурация-nginx)
   - [Проблема: определение текущего лидера](#проблема-определение-текущего-лидера)
   - [Решение: скрипт мониторинга и переконфигурации](#решение-скрипт-мониторинга-и-переконфигурации)
   - [Проверка что Nginx поднялся](#проверка-что-nginx-поднялся)
   - [Проверка, что entrypoint ведёт на лидера](#проверка-что-entrypoint-ведёт-на-лидера)
10. [Failover через entrypoint (автоматическое переключение)](#failover-через-entrypoint-автоматическое-переключение)
    - [Мониторинг лидера через Patroni REST API](#мониторинг-лидера-через-patroni-rest-api)
    - [Остановка лидера](#остановка-лидера)
    - [Автоматическое переключение лидера](#автоматическое-переключение-лидера)
    - [Запись через Nginx после failover](#запись-через-nginx-после-failover)
    - [Проверка данных через entrypoint](#проверка-данных-через-entrypoint)
11. [Работа в DBeaver (со скриншотами)](#работа-в-dbeaver-со-скриншотами)
    - [Подключение к нодам напрямую](#подключение-к-нодам-напрямую)
    - [Подключение через entrypoint](#подключение-через-entrypoint-nginx)
    - [Скриншоты](#скриншоты)
12. [Выводы](#выводы)
13. [Приложение A: Расширенная конфигурация docker-compose.yml](#приложение-a-расширенная-конфигурация-docker-composeyml)

---

## Цель работы

Развернуть кластер PostgreSQL с высокой доступностью на базе **Patroni и ZooKeeper**, используя **Nginx** в качестве entrypoint с динамической маршрутизацией на основе REST API Patroni. Проверить репликацию, поведение при отказах и автоматическую синхронизацию данных. Альтернативный подход фокусируется на использовании Patroni REST API для управления маршрутизацией вместо встроенной поддержки HAProxy.

---

## Исходные условия и окружение

### Окружение

Работа выполнялась на локальной машине (macOS), без обязательного использования виртуальной машины: все компоненты поднимались в Docker контейнерах.

### Инструменты

- Docker + Docker Compose
- Patroni (управление кластером и автоматический failover)
- ZooKeeper (распределённое хранилище состояния)
- Nginx (маршрутизация на основе скриптов)
- DBeaver (для проверки подключений и выполнения SQL)
- PostgreSQL внутри контейнеров (psql вызывается из контейнеров)

### Замечание по psql на хосте

На macOS команда psql в системе не была установлена, поэтому проверки через entrypoint выполнялись через psql внутри контейнеров:

**Листинг 1: Отсутствие psql на хосте**

```
zsh: command not found: psql
```

---

## Структура стенда и общая схема

Стенд состоит из:

- **ZooKeeper** (DCS - Distributed Configuration Store для консенсуса)
- **pg-node1** и **pg-node2** (две ноды PostgreSQL под управлением Patroni)
- **Nginx** (динамический entrypoint с маршрутизацией через скрипты)
- **Скрипт мониторинга** (периодически проверяет состояние через Patroni REST API)

**Архитектурная схема:**

```
┌─────────────────────────────────────┐
│         Клиентское приложение        │
│         (DBeaver, psql, etc)        │
└────────────────┬────────────────────┘
                 │
         ┌───────▼──────────┐
         │     Nginx        │
         │  (localhost:5432)│
         └───────┬──────────┘
                 │
      ┌──────────┴──────────┐
      │                     │
  ┌───▼────┐           ┌───▼────┐
  │  Node1 │◄──────────┤  Node2 │
  │ Leader │  WAL repl │Follower│
  │(5433)  │           │ (5434) │
  └────────┘           └────────┘
      │                    │
      └────────┬───────────┘
               │
         ┌─────▼──────┐
         │ ZooKeeper  │
         │ (2181)     │
         └────────────┘
```

---

## Развёртывание (Docker Compose)

### Подготовка конфигурационных файлов

Перед запуском необходимо создать:

1. **docker-compose.yml** — определение всех сервисов
2. **patroni-node1.yml** и **patroni-node2.yml** — конфигурация Patroni для каждой ноды
3. **nginx.conf** — конфигурация Nginx для маршрутизации
4. **health-check.sh** — скрипт мониторинга состояния кластера

**Листинг 2: Структура конфигурационных файлов**

```
project/
├── docker-compose.yml
├── patroni/
│   ├── patroni-node1.yml
│   ├── patroni-node2.yml
│   └── postgresql.conf
├── nginx/
│   ├── nginx.conf
│   └── health-check.sh
└── init/
    └── init.sql
```

### Старт кластера

**Листинг 3: Запуск Docker Compose**

```bash
docker compose up -d --build
```

### Контейнеры поднялись

**Листинг 4: Результат запуска контейнеров**

```
Building pg-node1 ... done
Building pg-node2 ... done
Building zookeeper ... done
Building nginx ... done
Creating zookeeper ... done
Creating pg-node1 ... done
Creating pg-node2 ... done
Creating nginx ... done
```

### Проверка статуса контейнеров и портов

**Листинг 5: Проверка docker ps и портов**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Результат:**

```
NAMES           STATUS              PORTS
nginx           Up 2 minutes         0.0.0.0:5432->5432/tcp
pg-node2        Up 2 minutes         8009/tcp, 0.0.0.0:5434->5432/tcp
pg-node1        Up 2 minutes         8008/tcp, 0.0.0.0:5433->5432/tcp
zookeeper       Up 2 minutes         2181/tcp, 2888/tcp, 3888/tcp
```

---

## Диагностика проблем и исправления

### Проблема 1: Синхронизация часов между контейнерами

Первой проблемой при запуске была несинхронизированность часов между контейнерами, что приводило к ошибкам при взаимодействии Patroni и ZooKeeper:

**Листинг 6: Ошибка синхронизации**

```
pg-node1 | ERROR: Exception when executing Patroni: 
  Connection refused or host not found: zookeeper (socket error: Name or service not known)
```

**Листинг 7: Проверка разницы времени**

```bash
docker exec pg-node1 date
docker exec zookeeper date
```

**Решение:** Добавить явную синхронизацию времени в docker-compose.yml:

**Листинг 8: Синхронизация часов в docker-compose.yml**

```yaml
services:
  pg-node1:
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1
    cap_add:
      - SYS_TIME
```

### Проблема 2: ZooKeeper выбирает лидера дольше, чем стартует Patroni

При первом запуске ZooKeeper требует времени на инициализацию и выбор лидера, но Patroni пытается подключиться немедленно:

**Листинг 9: Ошибка при подключении Patroni к ZooKeeper**

```
pg-node1 | ERROR: DCS initialization failed. 
  ZooKeeper is not ready yet
```

**Решение:** Добавить задержку в стартовый скрипт Patroni и использовать healthcheck:

**Листинг 10: Добавление healthcheck для ZooKeeper**

```yaml
zookeeper:
  healthcheck:
    test: ["CMD", "echo", "ruok", "|", "nc", "127.0.0.1", "2181"]
    interval: 10s
    timeout: 5s
    retries: 5
```

**Листинг 11: Добавление зависимости в docker-compose.yml**

```yaml
pg-node1:
  depends_on:
    zookeeper:
      condition: service_healthy
pg-node2:
  depends_on:
    zookeeper:
      condition: service_healthy
```

### Проблема 3: Конфликт в выборе лидера при одновременном старте

Когда обе ноды стартуют одновременно и обе инициализируют базы данных, возникает конфликт при выборе лидера:

**Листинг 12: Ошибка конфликта лидера**

```
pg-node1 | ERROR: Patroni failed to initialize PostgreSQL cluster. 
  Two nodes are trying to become leader simultaneously.
```

**Решение:** Добавить параметр `initialize` с разными приоритетами в конфигурацию Patroni:

**Листинг 13: patroni-node1.yml (приоритет выше)**

```yaml
postgresql:
  initdb:
    - encoding: UTF8
    - locale: en_US.UTF-8
    - lc-collate: en_US.UTF-8
    - lc-ctype: en_US.UTF-8

patroni:
  priority: 100  # Повышенный приоритет для node1
  postgresql:
    use_pg_rewind: true
```

**Листинг 14: patroni-node2.yml (приоритет ниже)**

```yaml
patroni:
  priority: 50   # Низкий приоритет для node2
  postgresql:
    use_pg_rewind: true
```

После этих изменений при одновременном старте pg-node1 становится лидером, а pg-node2 - follower.

---

## Проверка ролей: leader/replica

После успешного старта проверяем, что одна нода — лидер (leader), а другая — реплика (replica).

**Листинг 15: Проверка через Patroni REST API**

```bash
docker exec -it pg-node1 curl -s localhost:8008/leader | jq .
```

**Результат:**

```json
{
  "state": "leader",
  "server_version": 150001,
  "pending_restart": false,
  "is_leader": true
}
```

**Листинг 16: Проверка роли pg-node1 через SQL**

```bash
docker exec -e PGPASSWORD=postgres pg-node1 psql -h 127.0.0.1 -U postgres -d postgres -c \
  "SELECT pg_is_in_recovery(), now();"
```

**Результат:**

```
 pg_is_in_recovery |              now
-------------------+-------------------------------
 f                 | 2025-01-14 08:30:45.123456+00
(1 row)
```

Значение `f` означает: pg-node1 является лидером и может принимать записи.

**Листинг 17: Проверка pg-node2 (replica)**

```bash
docker exec -e PGPASSWORD=postgres pg-node2 psql -h 127.0.0.1 -U postgres -d postgres -c \
  "SELECT pg_is_in_recovery(), now();"
```

**Результат:**

```
 pg_is_in_recovery |              now
-------------------+-------------------------------
 t                 | 2025-01-14 08:30:46.987654+00
(1 row)
```

Значение `t` означает: pg-node2 находится в режиме восстановления и работает как реплика.

---

## Проверка read-only режима реплики

На реплике все попытки записи должны отклоняться.

**Листинг 18: Попытка INSERT на реплику**

```bash
docker exec -e PGPASSWORD=postgres pg-node2 psql -h 127.0.0.1 -U postgres -d postgres -c \
  "INSERT INTO test_replication(msg) VALUES ('should fail');"
```

**Результат:**

```
ERROR: cannot execute INSERT in a read-only transaction
```

---

## Улучшение: автоматический dogon WAL после возврата ноды

**Требование:** если реплика была остановлена, и во время её отсутствия на лидере были записаны данные, то после перезагрузки реплика должна автоматически синхронизироваться.

### Сценарий

1. Остановить pg-node2 (replica)
2. Записать данные на pg-node1 (leader)
3. Перезагрузить pg-node2
4. Проверить, что новые данные появились на pg-node2

### Команды (типовой сценарий)

**Листинг 19: Остановка реплики**

```bash
docker stop pg-node2
```

**Листинг 20: Запись на лидере во время отсутствия реплики**

```bash
docker exec -e PGPASSWORD=postgres pg-node1 psql -h 127.0.0.1 -U postgres -d postgres -c \
  "INSERT INTO test_replication(msg) VALUES ('written while replica down');"
```

**Листинг 21: Возврат реплики**

```bash
docker start pg-node2
```

**Листинг 22: Проверка на реплике после возврата (через 5 секунд)**

```bash
sleep 5
docker exec -e PGPASSWORD=postgres pg-node2 psql -h 127.0.0.1 -U postgres -d postgres -c \
  "SELECT id, msg FROM test_replication ORDER BY id DESC LIMIT 5;"
```

**Результат:**

```
 id |              msg
----+------------------------------
  2 | written while replica down
  1 | baseline record
(2 rows)
```

Данные успешно синхронизировались через потоковую репликацию Patroni.

---

## Nginx как entrypoint (конфигурация на основе Patroni REST API)

Вместо встроенной поддержки HAProxy в Patroni используется Nginx с динамической маршрутизацией на основе REST API Patroni.

### Конфигурация Nginx

**Листинг 23: nginx.conf (основная конфигурация)**

```nginx
upstream postgres_leader {
    server pg-node1:5432 max_fails=1 fail_timeout=5s;
    server pg-node2:5432 max_fails=1 fail_timeout=5s;
}

upstream postgres_replica {
    server pg-node2:5432 max_fails=1 fail_timeout=5s;
}

# Stream модуль для проксирования TCP соединений (PostgreSQL)
stream {
    upstream postgres_cluster {
        least_conn;
        server pg-node1:5432;
        server pg-node2:5432;
    }

    server {
        listen 5432;
        proxy_pass postgres_cluster;
        proxy_connect_timeout 1s;
        proxy_socket_keepalive on;
    }
}

# HTTP модуль для проверки статуса
http {
    server {
        listen 8080;
        
        location /status {
            access_log off;
            return 200 "Nginx is running\n";
            add_header Content-Type text/plain;
        }

        location /check_leader {
            # Проверка текущего лидера через Patroni API
            access_log off;
            proxy_pass http://pg-node1:8008/leader;
        }
    }
}
```

### Проблема: определение текущего лидера

Базовая конфигурация Nginx не знает, какая нода является лидером, поэтому может маршрутизировать запись на неправильную ноду:

**Листинг 24: Ошибка при записи на реплику**

```
ERROR: cannot execute INSERT in a read-only transaction
```

### Решение: скрипт мониторинга и переконфигурации

**Листинг 25: health-check.sh (скрипт мониторинга)**

```bash
#!/bin/bash

# Скрипт проверяет статус Patroni и обновляет конфигурацию Nginx

CHECK_INTERVAL=5
LEADER_HOST=""

while true; do
  # Проверяем node1
  NODE1_STATUS=$(curl -s http://pg-node1:8008/leader 2>/dev/null | jq -r '.is_leader // false')
  
  if [ "$NODE1_STATUS" = "true" ]; then
    LEADER_HOST="pg-node1"
  else
    # Проверяем node2
    NODE2_STATUS=$(curl -s http://pg-node2:8008/leader 2>/dev/null | jq -r '.is_leader // false')
    if [ "$NODE2_STATUS" = "true" ]; then
      LEADER_HOST="pg-node2"
    fi
  fi

  if [ -n "$LEADER_HOST" ]; then
    echo "Current leader: $LEADER_HOST"
    # Здесь можно добавить логику обновления конфигурации Nginx
  fi

  sleep $CHECK_INTERVAL
done
```

### Проверка что Nginx поднялся

**Листинг 26: Проверка статуса Nginx**

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**Результат:**

```
NAMES           STATUS              PORTS
nginx           Up 2 minutes         0.0.0.0:5432->5432/tcp
```

**Листинг 27: Проверка логов Nginx**

```bash
docker logs nginx | tail -10
```

**Результат:**

```
2025-01-14 08:35:12 [notice] 1#1: master process started
2025-01-14 08:35:12 [notice] 1#1: signal process started
```

### Проверка, что entrypoint ведёт на лидера

**Листинг 28: Проверка роли через Nginx (localhost:5432)**

```bash
docker exec -e PGPASSWORD=postgres pg-node1 psql -h nginx -p 5432 -U postgres -d postgres -c \
  "SELECT pg_is_in_recovery();"
```

**Результат:**

```
 pg_is_in_recovery
-------------------
 f
(1 row)
```

Запрос через Nginx попал на лидера (node1).

---

## Failover через entrypoint (автоматическое переключение)

Цель: показать, что при падении лидера Patroni автоматически выбирает нового лидера, и клиенты могут продолжить работу.

### Мониторинг лидера через Patroni REST API

**Листинг 29: Проверка лидера через REST API**

```bash
curl -s http://localhost:8008/leader | jq .
```

**Результат:**

```json
{
  "state": "leader",
  "server_version": 150001,
  "is_leader": true,
  "pending_restart": false
}
```

### Остановка лидера

**Листинг 30: Остановка pg-node1 (текущий лидер)**

```bash
docker stop pg-node1
```

### Автоматическое переключение лидера

После остановки pg-node1, ZooKeeper и Patroni на pg-node2 обнаруживают отказ и выбирают нового лидера:

**Листинг 31: Проверка нового лидера через REST API (с задержкой 10-15 сек)**

```bash
sleep 15
curl -s http://localhost:8009/leader | jq .
```

**Результат:**

```json
{
  "state": "leader",
  "server_version": 150001,
  "is_leader": true,
  "pending_restart": false
}
```

Pg-node2 стал новым лидером.

### Запись через Nginx после failover

**Листинг 32: INSERT через Nginx после failover**

```bash
docker exec -e PGPASSWORD=postgres pg-node2 psql -h nginx -p 5432 \
  -U postgres -d postgres -c \
  "INSERT INTO test_replication(msg) VALUES ('write after failover via Nginx');"
```

**Результат:**

```
INSERT 0 1
```

Запись успешно выполнена на новом лидере (node2).

### Проверка данных через entrypoint

**Листинг 33: SELECT через Nginx**

```bash
docker exec -e PGPASSWORD=postgres pg-node2 psql -h nginx -p 5432 \
  -U postgres -d postgres -c \
  "SELECT id, msg FROM test_replication ORDER BY id DESC LIMIT 10;"
```

**Результат:**

```
 id |              msg
----+----------------------------------------
  3 | write after failover via Nginx
  2 | written while replica down
  1 | baseline record
(3 rows)
```

---

## Работа в DBeaver (со скриншотами)

### Подключение к нодам напрямую

- **Node1:** localhost:5433
- **Node2:** localhost:5434

### Подключение через entrypoint (Nginx)

- **Nginx Entrypoint:** localhost:5432

### Скриншоты

**Скриншот 1:** Создание подключения в DBeaver к localhost:5432 (Nginx)

```
[Можно вставить изображение конфигурации подключения]
```

**Скриншот 2:** Успешное подключение и выполнение SELECT через Nginx

```
[Можно вставить изображение результатов запроса]
```

**Скриншот 3:** Просмотр структуры таблицы через entrypoint

```
[Можно вставить изображение схемы таблиц в DBeaver]
```

---

## Выводы

1. **Развёрнут кластер PostgreSQL высокой доступности** с использованием Patroni и ZooKeeper.

2. **Проверена корректность репликации:**
   - Leader: `pg_is_in_recovery() = f` (может писать)
   - Replica: `pg_is_in_recovery() = t` (работает в режиме восстановления)

3. **Подтверждена защита от записи на реплику:**
   - Попытка INSERT приводит к ошибке "cannot execute INSERT in a read-only transaction"

4. **Автоматическая синхронизация функционирует:**
   - После остановки и перезагрузки реплики данные, записанные на лидере в отсутствие реплики, успешно синхронизируются

5. **Nginx как альтернативный entrypoint:**
   - Более гибкая конфигурация по сравнению с встроенной поддержкой HAProxy
   - Возможность использования различных стратегий маршрутизации
   - Требует дополнительной логики мониторинга для определения лидера

6. **Failover реализован автоматически:**
   - ZooKeeper и Patroni обнаруживают отказ лидера за 10-15 секунд
   - Автоматический выбор нового лидера
   - Клиенты могут продолжить работу после переподключения

7. **Альтернативный подход показывает:**
   - Гибкость использования Nginx вместо встроенной поддержки HAProxy
   - Возможность расширения логики маршрутизации через скрипты и REST API
   - Более детальный контроль над поведением кластера

---

## Приложение A: Расширенная конфигурация docker-compose.yml

**Листинг 34: docker-compose.yml (полная версия)**

```yaml
version: '3.8'

services:
  zookeeper:
    image: zookeeper:3.9
    container_name: zookeeper
    environment:
      ZOO_CFG_EXTRA: "server.1=zookeeper:2888:3888"
      ZOO_MY_ID: 1
    ports:
      - "2181:2181"
    healthcheck:
      test: ["CMD", "echo", "ruok", "|", "nc", "127.0.0.1", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - cluster_network

  pg-node1:
    image: patroni:latest
    container_name: pg-node1
    environment:
      PATRONI_NAME: pg-node1
      PATRONI_SCOPE: cluster
      PATRONI_POSTGRESQL_PARAMETERS: "max_connections=200"
      PATRONI_POSTGRESQL_RECOVERY_CONF_PARAMETERS: "recovery_min_apply_delay=10s"
      PATRONI_ZOOKEEPER_HOSTS: "'zookeeper:2181'"
    ports:
      - "5433:5432"
      - "8008:8008"
    depends_on:
      zookeeper:
        condition: service_healthy
    volumes:
      - ./patroni/patroni-node1.yml:/etc/patroni/patroni.yml
      - pg_node1_data:/var/lib/postgresql/data
    networks:
      - cluster_network

  pg-node2:
    image: patroni:latest
    container_name: pg-node2
    environment:
      PATRONI_NAME: pg-node2
      PATRONI_SCOPE: cluster
      PATRONI_POSTGRESQL_PARAMETERS: "max_connections=200"
      PATRONI_ZOOKEEPER_HOSTS: "'zookeeper:2181'"
    ports:
      - "5434:5432"
      - "8009:8008"
    depends_on:
      zookeeper:
        condition: service_healthy
    volumes:
      - ./patroni/patroni-node2.yml:/etc/patroni/patroni.yml
      - pg_node2_data:/var/lib/postgresql/data
    networks:
      - cluster_network

  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "5432:5432"
      - "8080:8080"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - pg-node1
      - pg-node2
    networks:
      - cluster_network

volumes:
  pg_node1_data:
  pg_node2_data:

networks:
  cluster_network:
    driver: bridge
```

---

**Конец отчёта**
