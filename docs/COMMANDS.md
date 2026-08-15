# 📋 1C Infrastructure — Шпаргалка по командам

<a name="top"></a>

Быстрый справочник по управлению инфраструктурой `1c-home-infrastructure`.

---

## 🔌 Управление сервисами

### Все сервисы
```powershell
# Запустить все
docker-compose up -d

# Остановить все
docker-compose down

# Перезапустить все
docker-compose restart

# Просмотр логов всех сервисов
docker-compose logs --tail 100

# Статус всех сервисов
docker-compose ps
```

### Отдельные сервисы
```powershell
# PostgreSQL (СУБД для 1С)
docker-compose stop postgres-1c
docker-compose start postgres-1c
docker-compose restart postgres-1c
docker-compose logs postgres-1c --tail 50

# pgAdmin (веб-интерфейс БД)
docker-compose restart pgadmin
docker-compose logs pgadmin --tail 50

# Portainer (оркестрация)
docker-compose restart portainer
docker-compose logs portainer --tail 50

# Prometheus (сбор метрик)
docker-compose restart prometheus
docker-compose logs prometheus --tail 50

# Grafana (дашборды + алерты)
docker-compose restart grafana
docker-compose logs grafana --tail 50

# Blackbox Exporter (HTTP-проверки)
docker-compose restart blackbox-exporter
docker-compose logs blackbox-exporter --tail 50

# postgres-exporter (метрики СУБД)
docker-compose restart postgres-exporter
docker-compose logs postgres-exporter --tail 50

# cAdvisor (метрики контейнеров)
docker-compose restart cadvisor
docker-compose logs cadvisor --tail 50

# VoceChat (уведомления)
docker-compose restart vocechat-notifications
docker-compose logs vocechat-notifications --tail 50
```

[🔝 Наверх](#top)

---

## 🔍 Prometheus Queries

### Проверка доступности сервисов
```promql
# Все сервисы
up

# PostgreSQL
pg_up

# Конкретный job
up{job="prometheus"}
up{job="cadvisor"}
up{job="postgres-exporter"}

# HTTP доступность (Blackbox)
probe_success{job="blackbox-http"}

# Конкретные HTTP endpoints (внутренние порты Docker-сети)
probe_success{instance="http://grafana:3000/api/health"}
probe_success{instance="http://pgadmin4:80"}
probe_success{instance="http://portainer:9000"}
probe_success{instance="http://vocechat-notifications:3000"}
```

### Метрики контейнеров (cAdvisor)
```promql
# CPU использование (в процентах, за 5 минут)
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Использование памяти (в байтах)
container_memory_usage_bytes{name!=""}

# Использование памяти (в МБ)
container_memory_usage_bytes{name!=""} / 1024 / 1024

# Активность контейнера
container_last_seen{name="grafana"}
```

### PostgreSQL метрики
```promql
# Версия PostgreSQL
pg_static

# Количество транзакций (commit)
pg_stat_database_xact_commit

# Размер БД (в байтах)
pg_database_size_bytes

# Статус подключения
pg_up

# Количество активных соединений
pg_stat_activity_count
```

[🔝 Наверх](#top)

---

## 🚨 Алерты (9 правил)

### 🏗️ Инфраструктурные алерты

| Алерт | Query | Порог | Описание |
|-------|-------|-------|----------|
| `PostgreSQLDown` | `pg_up` | `== 0` | postgres-exporter не может подключиться к БД |
| `PrometheusDown` | `up{job="prometheus"}` | `== 0` | Сам Prometheus недоступен |
| `cAdvisorDown` | `up{job="cadvisor"}` | `== 0` | Экспортер метрик контейнеров недоступен |
| `GrafanaDown` | `probe_success{instance="http://grafana:3000/api/health", job="blackbox-http"}` | `== 0` | Grafana health check не пройден |
| `PgAdminDown` | `probe_success{instance="http://pgadmin4:80", job="blackbox-http"}` | `== 0` | pgAdmin не отвечает |
| `PortainerDown` | `probe_success{instance="http://portainer:9000", job="blackbox-http"}` | `== 0` | Portainer не отвечает |
| `VoceChatDown` | `probe_success{instance="http://vocechat-notifications:3000", job="blackbox-http"}` | `== 0` | VoceChat не отвечает |
| `ContainerHighCPU` | `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100` | `> 80` | CPU контейнера > 80% в течение 5 мин |
| `ContainerHighMemory` | `container_memory_usage_bytes{name!=""}` | `> 5368709120` | RAM контейнера > 5 ГБ в течение 5 мин |

### Тестирование алертов
```powershell
# Остановить сервис → подождать 2-3 минуты → проверить VoceChat
docker-compose stop pgadmin

# Запустить обратно
docker-compose start pgadmin

# Проверить алерты в Grafana
# → http://localhost:3002/alerting/list
```

> 💡 Уведомления о срабатывании алертов настраиваются через **Grafana UI → Alerting → Contact points**.

[🔝 Наверх](#top)

---

## 🔧 Диагностика

### Проверка логов
```powershell
# Логи конкретного сервиса
docker-compose logs <service_name> --tail 100 -f

# Логи за последние 10 минут
docker-compose logs <service_name> --since 10m
```

### Проверка сети
```powershell
# Список сетей
docker network ls

# Инспекция сети (имя зависит от папки проекта)
docker network inspect 1c-home-infrastructure_1c-infrastructure
docker network inspect 1c-home-infrastructure_monitoring

# Проверка DNS resolution из контейнера
docker exec prometheus nslookup postgres
docker exec prometheus nslookup grafana
```

### Проверка volumes
```powershell
# Список volumes
docker volume ls

# Инспекция volume (имя зависит от папки проекта)
docker volume inspect 1c-home-infrastructure_postgres-data
```

### Очистка (осторожно!)
```powershell
# Очистка остановленных контейнеров
docker container prune

# Очистка неиспользуемых volumes
docker volume prune

# Очистка неиспользуемых сетей
docker network prune

# Просмотр использования диска
docker system df

# Полная очистка (⚠️ удалит всё неиспользуемое!)
docker system prune -a --volumes
```

[🔝 Наверх](#top)

---

## 🔐 Переменные окружения

Основные пароли хранятся в `.env` (файл в `.gitignore`):

```powershell
# Скопировать пример
Copy-Item .env.example .env

# Редактировать (VS Code)
code .env

# Проверить, что .env в .gitignore
git check-ignore .env
```

> ⚠️ **Никогда не коммитьте `.env`!**

[🔝 Наверх](#top)

---

## 📁 Основные файлы

| Файл | Путь (от корня репозитория) |
|------|------|
| `docker-compose.yml` | `./docker-compose.yml` |
| `prometheus.yml` | `./monitoring/prometheus.yml` |
| `blackbox.yml` | `./monitoring/blackbox.yml` |
| `alerts.yml` | `./monitoring/prometheus/alerts.yml` |
| `recording-rules.yml` | `./monitoring/prometheus/recording-rules.yml` |
| `datasources.yml` | `./monitoring/grafana/provisioning/datasources.yml` |
| `.env` | `./.env` (создаётся из `.env.example`) |
| `.gitignore` | `./.gitignore` |

### Перезагрузка конфигурации
```powershell
# Prometheus (после изменения prometheus.yml или alerts.yml)
docker-compose restart prometheus

# Grafana (после изменения datasources.yml)
docker-compose restart grafana

# Blackbox (после изменения blackbox.yml)
docker-compose restart blackbox-exporter
```

[🔝 Наверх](#top)

---

## 🔗 Полезные URL

| Сервис | URL | Логин / Пароль |
|--------|-----|----------------|
| **PostgreSQL** | `localhost:5432` | `postgres` / из `.env` |
| **Portainer** | [http://localhost:9000](http://localhost:9000) | Создаётся при первом входе |
| **pgAdmin** | [http://localhost:5050](http://localhost:5050) | из `.env` |
| **Grafana** | [http://localhost:3002](http://localhost:3002) | из `.env` |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | без авторизации |
| **VoceChat** | [http://localhost:3001](http://localhost:3001) | из `.env` |
| **Blackbox Exporter** | [http://localhost:9115](http://localhost:9115) | без авторизации |
| **cAdvisor** | [http://localhost:8080](http://localhost:8080) | без авторизации |

> 🔒 **Удалённый доступ:** через **Tailscale VPN** по адресу `100.x.x.x:port` — безопасно, без проброса портов.

[🔝 Наверх](#top)

---

## 📚 Документация проекта

| Файл | Описание |
|------|----------|
| [🏠 README.md](../README.md) | Главный README репозитория |
| [📘 infrastructure-guide.md](infrastructure-guide.md) | Полное руководство по развёртыванию |
| [🎥 backup-test-demo.md](backup-test-demo.md) | Видео: тестовый режим бэкапа |
| [🎥 backup-full-demo.md](backup-full-demo.md) | Видео: полный бэкап |

> 💡 **История разработки и инсайты:**  
> Опубликованы в статье на InfoStart:  
> 🔗 [DevOps для 1С на практике](https://infostart.ru/1c/articles/2658161/)

[🔝 Наверх](#top)

---

## 📞 Поддержка

При проблемах:

1. **Проверить логи:** `docker-compose logs <service> --tail 100`
2. **Проверить статус:** `docker-compose ps`
3. **Перезапустить сервис:** `docker-compose restart <service>`
4. **Проверить алерты в Grafana:** [http://localhost:3002/alerting/list](http://localhost:3002/alerting/list)
5. **Проверить Prometheus targets:** [http://localhost:9090/targets](http://localhost:9090/targets)

[🔝 Наверх](#top)