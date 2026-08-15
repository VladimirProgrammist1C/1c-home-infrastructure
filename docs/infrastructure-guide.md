# 📘 1C Infrastructure — Подробное руководство

<a name="top"></a>

Полное руководство по развёртыванию и настройке инфраструктуры для 1С-разработки.

**Версия:** 3.0 (Clean for Public Repo)  
**Последнее обновление:** 16.08.2026  
**Автор:** Vladimir Bessonov

---

## 📋 Содержание

- [Введение](#введение)
- [Требования](#требования)
- [Установка](#установка)
- [Настройка](#настройка)
- [Мониторинг](#мониторинг)
- [Система бэкапов](#система-бэкапов)
- [Управление сервисами](#управление-сервисами)
- [Диагностика](#диагностика)
- [FAQ](#faq)
- [Полезные ресурсы](#полезные-ресурсы)
- [Changelog](#changelog)
- [Поддержка](#поддержка)

[🔝 Наверх](#top)

---

## Введение

Эта инфраструктура предназначена для:

✅ **Изолированной 1С-разработки** — отдельные контуры dev/test/prod  
✅ **Мониторинга в реальном времени** — Prometheus + Grafana + Blackbox  
✅ **Автоматических уведомлений** — VoceChat (on-premise мессенджер)  
✅ **Безопасного удалённого доступа** — Tailscale VPN (WireGuard)  
✅ **Автоматических бэкапов** — 4-этапный скрипт (конфиги, БД, volumes)

> 💡 **Цель:** Максимально приблизить домашнюю среду к продакшену без потери удобства разработки.

**Основное оборудование:** Geekom A9 Max (Ryzen AI 9 HX 370, Windows 11 Pro)  
**Время развёртывания:** ~30 минут (при наличии дистрибутива PostgreSQL 1С)

[🔝 Наверх](#top)

---

## Требования

### Аппаратные

| Компонент | Минимум | Рекомендуется |
|-----------|---------|---------------|
| **CPU** | 8 ядер | 12+ ядер (Ryzen AI 9 HX 370) |
| **RAM** | 16 ГБ | 32–64 ГБ |
| **Диск** | 500 ГБ SSD | 1 ТБ NVMe (отдельный том для данных) |
| **Сеть** | 1 Гбит/с | 2.5 Гбит/с + Wi-Fi 6 |
| **ОС** | Windows 10/11 Pro | Windows 11 Pro x64 |

### Программные

| Компонент | Версия | Примечание |
|-----------|--------|------------|
| **Docker Desktop** | 4.25+ | С включённым WSL2 backend |
| **Git** | 2.40+ | Для работы с репозиторием |
| **PowerShell** | 5.1+ | Для запуска скриптов |
| **Tailscale** | 1.95+ | Для защищённого доступа (опционально) |
| **1С:Предприятие** | 8.3.20 (или новее) | Клиент-серверный режим |

[🔝 Наверх](#top)

---

## Установка

### Шаг 1: Установка Docker Desktop

```powershell
# 1. Скачайте с официального сайта
#    https://www.docker.com/products/docker-desktop

# 2. Установите с настройками по умолчанию
# 3. Перезагрузите компьютер
# 4. Запустите Docker Desktop

# 5. Проверьте установку
docker --version
docker-compose --version
```

### Шаг 2: Клонирование репозитория

```bash
git clone <repository-url> 1c-home-infrastructure
cd 1c-home-infrastructure
```

### Шаг 3: Настройка переменных окружения

```powershell
# Скопируйте пример
Copy-Item .env.example .env

# Отредактируйте .env (VS Code, Notepad++ или любой редактор)
code .env
```

**Обязательно измените пароли в `.env`:**

```env
# Пароли для Docker-сервисов (НЕ КОММИТИТЬ!)
DB_PASSWORD=your_password_here
PGADMIN_EMAIL=admin@example.ru
PGADMIN_PASSWORD=admin

# === Мониторинг ===
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=your_grafana_password_here
VOCECHAT_NOTIFY_PASSWORD=your_vocechat_notify_password_here
VOCECHAT_NOTIFY_WEBHOOK_TOKEN=your-webhook-token-here
```

> ⚠️ **Важно:** Файл `.env` добавлен в `.gitignore`. **Никогда не коммитьте его в репозиторий!**

### Шаг 4: Подготовка PostgreSQL 1С

1. Скачайте архив `postgresql_18.1_2_ubuntu_22.04_x86_64_package.tar.bz2` с [releases.1c.ru](https://releases.1c.ru).
2. Создайте папку `docker/postgres-1c/` и положите архив туда.
3. Добавьте `Dockerfile` и `entrypoint.sh` (с русской локалью `ru_RU.UTF-8`).
4. Соберите образ:

```powershell
docker build -t postgres:18.1-2.1C-ubuntu2204 ./docker/postgres-1c
```

> ⚠️ **Критично:** Кластер PostgreSQL должен быть инициализирован с русской локалью (`ru_RU.UTF-8`). Без этого 1С не сможет создавать базы!

### Шаг 5: Запуск инфраструктуры

```powershell
# Запустите все сервисы
docker-compose up -d

# Проверьте статус
docker-compose ps
# Все сервисы должны быть "Up" или "Up (healthy)"
```

### Шаг 6: Проверка доступа

```
🔗 Локально:
├─ PostgreSQL:   localhost:5432     (postgres / из .env)
├─ Portainer:    http://localhost:9000
├─ pgAdmin:      http://localhost:5050   (из .env)
├─ Grafana:      http://localhost:3002   (из .env)
├─ Prometheus:   http://localhost:9090
├─ VoceChat:     http://localhost:3001   (из .env)
├─ Blackbox:     http://localhost:9115
└─ cAdvisor:     http://localhost:8080
```

[🔝 Наверх](#top)

---

## Настройка

### 🔷 PostgreSQL

**Подключение:**

| Параметр | Значение |
|----------|----------|
| Host | `localhost` (извне) или `postgres` (в Docker-сети) |
| Port | `5432` |
| Database | `template1c` (или создайте свою) |
| Username | `postgres` |
| Password | из `.env` |

**Создание базы для 1С:**

```powershell
docker exec -it postgres-1c psql -U postgres

# Внутри psql:
CREATE DATABASE "DemoHRMCorpDemo_bot";
\q
```

### 🔷 pgAdmin

1. Откройте [http://localhost:5050](http://localhost:5050)
2. Войдите с логином и паролем из `.env`
3. Добавьте сервер PostgreSQL:
   - **Host:** `postgres`
   - **Port:** `5432`
   - **Username:** `postgres`
   - **Password:** из `.env`

### 🔷 Grafana

1. Откройте [http://localhost:3002](http://localhost:3002)
2. Войдите с логином и паролем из `.env`
3. Смените пароль при первом входе
4. **Настройка дашборда по умолчанию:**
   - Кликните по иконке пользователя → **Profile**
   - В разделе **Preferences** → **Home Dashboard** выберите нужный дашборд
   - Нажмите **Save preferences**

**Алерты уже настроены!** Проверьте:
- Список алертов: [http://localhost:3002/alerting/list](http://localhost:3002/alerting/list)
- Contact Points: [http://localhost:3002/alerting/notifications](http://localhost:3002/alerting/notifications)

> 💡 **Уведомления** настраиваются вручную через **Alerting → Contact points** (например, VoceChat webhook, Telegram, Slack, Email).

### 🔷 Email-уведомления (резервный канал)

**Email используется** только как резервный канал — когда основной канал (VoceChat) недоступен.

#### 📋 Настройка

**1. Включите SMTP в `.env`:**

```env
GF_SMTP_ENABLED=true
GF_SMTP_HOST=smtp.yandex.ru:465
GF_SMTP_USER=your-email@yandex.ru
GF_SMTP_PASSWORD=your-app-password    # Пароль приложения, не от ящика!
GF_SMTP_FROM_ADDRESS=grafana-alerts@yourdomain.ru
```

**2. Перезапустите Grafana:**

```powershell
docker-compose up -d --force-recreate grafana
```

**3. Создайте Contact Point в Grafana UI:**

1. Откройте: **Alerting** → **Contact points** → **+ Add contact point**
2. **Name:** `Email Critical`
3. **Integration:** `Email`
4. **Addresses:** ваш email для критичных уведомлений
5. **Save & Test**

**4. Настройте маршрутизацию:**

1. Откройте: **Alerting** → **Notification policies**
2. Добавьте политику для `alertname = VoceChatDown`
3. Добавьте `Email Critical` как Contact Point

> ✅ **Логика:** Все алерты идут в VoceChat. **Только** при падении VoceChat уведомление дублируется на email.

#### 📮 SMTP-настройки популярных провайдеров

| Провайдер | SMTP хост | Порт |
|-----------|-----------|------|
| **Yandex 360** | `smtp.yandex.ru` | 465 (SSL) / 587 (STARTTLS) |
| **Mail.ru** | `smtp.mail.ru` | 465 (SSL) / 587 (STARTTLS) |
| **Gmail** | `smtp.gmail.com` | 465 (SSL) / 587 (STARTTLS) |

> 💡 Уточните SMTP-настройки у вашего почтового провайдера (раздел «Почта» → «Настройки» → «SMTP»).


### 🔷 Portainer

1. Откройте [http://localhost:9000](http://localhost:9000)
2. Создайте аккаунт при первом входе
3. Подключение к Docker настроено автоматически

### 🔷 Tailscale VPN (опционально)

1. Установите Tailscale на сервер и клиентские устройства
2. Войдите под одним аккаунтом
3. Получите IP-адрес устройства (например, `100.74.x.x`)
4. Подключайтесь к сервисам по этому IP

> 🔐 **Tailscale** обеспечивает безопасный доступ через зашифрованный туннель (WireGuard).

⚠️ **Важно:** Порты `0.0.0.0` в `docker-compose.yml` безопасны **только при использовании VPN**!

[🔝 Наверх](#top)

---

## Мониторинг

### 🏗️ Архитектура

```
┌─────────────────┐
│  Сервисы (9)    │
└─────────────────┘
         │
    ┌────▼─────┐
    │Prometheus│ ← Сбор метрик каждые 30 сек
    └────┬─────┘
         │
    ┌────▼─────┐
    │  Grafana  │ ← Алерты, дашборды
    └────┬─────┘
         │
    ┌────▼──────┐
    │ VoceChat  │ ← Уведомления (через webhook)
    └───────────┘
```

### 📦 Компоненты мониторинга

| Компонент | Назначение | Порт |
|-----------|------------|------|
| Prometheus | Сбор и хранение метрик (time-series DB) | 9090 |
| cAdvisor | Метрики контейнеров (CPU, RAM) | 8080 |
| postgres-exporter | Метрики PostgreSQL | 9187 |
| Blackbox Exporter | HTTP-проверки доступности | 9115 |
| Grafana | Визуализация, алерты | 3002 |

### 🔔 Алерты (9 правил)

Алерты настроены в `monitoring/prometheus/alerts.yml` и автоматически подхватываются Prometheus.

#### 🏗️ Инфраструктурные алерты

| Алерт | Критичность | Назначение |
|-------|-------------|---------|
| `PostgreSQLDown` | 🔴 Critical | Контроль доступности СУБД |
| `PrometheusDown` | 🔴 Critical | Контроль доступности сборщика метрик |
| `cAdvisorDown` | 🔴 Critical | Контроль доступности метрик контейнеров |
| `GrafanaDown` | 🔴 Critical | Контроль доступности панели мониторинга |
| `PgAdminDown` | 🔴 Critical | Контроль доступности админки БД |
| `PortainerDown` | 🔴 Critical | Контроль доступности оркестратора |
| `VoceChatDown` | 🔴 Critical | Контроль доступности мессенджера |
| `ContainerHighCPU` | 🟡 Warning | CPU контейнера > 80% в течение 5 мин |
| `ContainerHighMemory` | 🟡 Warning | RAM контейнера > 5 ГБ в течение 5 мин |

### 📨 Уведомления

Уведомления настраиваются через **Grafana UI → Alerting → Contact points**. Возможные каналы:

- 💬 **VoceChat** — on-premise мессенджер (через webhook)
- 📧 **Email** — любой SMTP-сервер
- 📱 **Telegram** — через Bot API
- 📨 **Slack** — через Incoming Webhook

[🔝 Наверх](#top)

---

## Система бэкапов

В репозитории есть **интерактивный 4-этапный скрипт** `scripts/backup-scripts/full-backup-rus.ps1`.

### 🎯 Что делает скрипт

1. **Этап 1: Конфиги проекта** — robocopy всех файлов проекта (docker-compose.yml, .env, monitoring/, scripts/)
2. **Этап 2: Не-1С базы PostgreSQL** — pg_dump + gzip (по whitelist из `non-1c-databases.txt`)
3. **Этап 3: Конфиги PostgreSQL** — postgresql.conf, pg_hba.conf, pg_ident.conf
4. **Этап 4: Docker Volumes** — tar.gz архивы для каждого volume

> 💡 **1С-ные базы** (определяемые по отсутствию в whitelist) должны бэкапиться **отдельным инструментом** на ваш выбор.

### 🔧 Альтернативные инструменты для бэкапа 1С-ных баз

- **Обновлятор 1С** (В. Милькин) — GUI-инструмент с встроенным планировщиком
- **pg_dump** — стандартная утилита PostgreSQL
- **v8tools / rac** — консольные утилиты кластера 1С
- **Собственные PowerShell-скрипты** — под конкретную инфраструктуру
- **pgBackRest** — профессиональный инструмент для production (инкрементальные бэкапы, PITR)

> ⚠️ **Рекомендация:** 1С-ные базы (особенно в кластере) рекомендуется бэкапить специализированными инструментами, которые корректно работают с транзакционной моделью 1С и хранилищами конфигураций.

### 🚀 Запуск

```powershell
cd scripts/backup-scripts
.\full-backup-rus.ps1
```

Подробнее: [scripts/README.md](../scripts/README.md)

### 🎥 Видеодемонстрации

- [Тестовый режим бэкапа](backup-test-demo.md) (~3 мин)
- [Полный бэкап](backup-full-demo.md) (~10 мин)

[🔝 Наверх](#top)

---

## Управление сервисами

### 🎛️ Основные команды

```powershell
# Запустить все
docker-compose up -d

# Остановить все
docker-compose down

# Перезапустить сервис
docker-compose restart <service_name>

# Просмотр логов
docker-compose logs <service_name> --tail 100

# Статус всех сервисов
docker-compose ps

# Пересоздание контейнера (с сохранением volumes)
docker-compose up -d --force-recreate <service_name>
```

> 📚 Полная шпаргалка: [COMMANDS.md](COMMANDS.md)

### 🧪 Тестирование алертов

```powershell
# 1. Остановите сервис (например, pgAdmin)
docker-compose stop pgadmin

# 2. Подождите 2-3 минуты
# 3. Проверьте канал уведомлений — должно прийти сообщение

# 4. Запустите сервис обратно
docker-compose start pgadmin
# 5. Должно прийти RESOLVED-уведомление
```

### 📱 Управление через Portainer

Откройте [http://localhost:9000](http://localhost:9000) → **Containers**. Доступны все основные действия: Start, Stop, Restart, Logs, Console, Stats.

### 🔄 Обновление образов Docker

```powershell
docker-compose pull
docker-compose up -d --force-recreate
```

[🔝 Наверх](#top)

---

## Диагностика

### 🔴 Сервис не запускается

```powershell
# 1. Проверьте логи
docker-compose logs <service_name> --tail 100

# 2. Проверьте статус
docker-compose ps

# 3. Пересоздайте контейнер
docker-compose up -d --force-recreate <service_name>

# 4. Проверьте переменные окружения
Get-Content .env
```

### 🔴 Алерты не приходят

1. Проверьте статус алертов: [http://localhost:3002/alerting/list](http://localhost:3002/alerting/list)
2. Проверьте Contact Point: [http://localhost:3002/alerting/notifications](http://localhost:3002/alerting/notifications)
3. Проверьте Notification policies: [http://localhost:3002/alerting/routes](http://localhost:3002/alerting/routes)
4. Проверьте логи Grafana: `docker-compose logs grafana --tail 100`

### 🔴 Prometheus не видит метрики

1. Проверьте targets: [http://localhost:9090/targets](http://localhost:9090/targets)
2. Проверьте конфигурацию: `Get-Content monitoring/prometheus.yml`
3. Перезапустите: `docker-compose restart prometheus`

### 🔴 Blackbox не проверяет HTTP

1. Проверьте логи: `docker-compose logs blackbox-exporter --tail 50`
2. Проверьте конфигурацию: `Get-Content monitoring/blackbox.yml`
3. Найдите job `blackbox-http` в [http://localhost:9090/targets](http://localhost:9090/targets)

### 🔴 PostgreSQL не подключается

```powershell
# Статус
docker-compose ps postgres

# Проверка подключения
docker exec -it postgres-1c psql -U postgres -c "SELECT 1"

# Список БД
docker exec -it postgres-1c psql -U postgres -c "\l"
```

[🔝 Наверх](#top)

---

## FAQ

### ❓ Как добавить новую базу данных для 1С?

```powershell
docker exec -it postgres-1c psql -U postgres
CREATE DATABASE "YourDatabaseName";
\q
```

### ❓ Как изменить пароль PostgreSQL?

⚠️ **Внимание:** Это удалит все данные!

```powershell
# 1. Остановите PostgreSQL
docker-compose stop postgres

# 2. Измените DB_PASSWORD в .env

# 3. Удалите volume (данные удалятся!)
docker volume rm 1c-home-infrastructure_postgres-data

# 4. Запустите заново
docker-compose up -d postgres
```

### ❓ Как изменить порты сервисов?

Отредактируйте `docker-compose.yml`:

```yaml
ports:
  - "3003:3000"  # Внешний:Внутренний
```

После изменения — пересоздайте контейнер: `docker-compose up -d --force-recreate <service>`

### ❓ Где хранятся данные?

| Сервис | Хранилище |
|--------|-----------|
| PostgreSQL | Docker volume `1c-home-infrastructure_postgres-data` |
| Grafana | Docker volume `1c-home-infrastructure_grafana-data` |
| Prometheus | Docker volume `1c-home-infrastructure_prometheus-data` |
| pgAdmin | Docker volume `1c-home-infrastructure_pgadmin-data` |
| Portainer | Docker volume `1c-home-infrastructure_portainer-data` |
| VoceChat | Docker volume `1c-home-infrastructure_vocechat-notify-data` |

> 💡 Имя volume = `<имя-папки-проекта>_<имя-volume>`. Если папка называется иначе, префикс будет другим.

### ❓ Как сделать бэкап PostgreSQL вручную?

```powershell
# Создать дамп
docker exec postgres-1c pg_dump -U postgres template1c > backup.sql

# Восстановить
cat backup.sql | docker exec -i postgres-1c psql -U postgres
```

### ❓ Как сделать бэкап всей инфраструктуры?

```powershell
# Запустить интерактивный скрипт
cd scripts/backup-scripts
.\full-backup-rus.ps1
```

Бэкапы сохраняются в папке `Backups/` внутри проекта.

### ❓ Как очистить неиспользуемые ресурсы Docker?

```powershell
# Остановленные контейнеры
docker container prune

# Неиспользуемые volumes (осторожно!)
docker volume prune

# Полная очистка (⚠️ удалит всё неиспользуемое!)
docker system prune -a --volumes
```

### ❓ Как добавить новый алерт?

1. Откройте [http://localhost:3002/alerting/list](http://localhost:3002/alerting/list)
2. Нажмите **New alert rule**
3. Настройте Query (PromQL), Condition, Labels, Annotations
4. Сохраните

> 💡 Примеры PromQL-запросов: [COMMANDS.md → Prometheus Queries](COMMANDS.md)

### ❓ Почему порты `0.0.0.0` в docker-compose.yml безопасны?

Потому что доступ возможен **только через Tailscale VPN**. Без подключённого туннеля порты не видны из публичного интернета.

[🔝 Наверх](#top)

---

## Полезные ресурсы

### 📚 Официальная документация

- [1С:ИТС releases](https://releases.1c.ru)
- [PostgreSQL docs](https://postgrespro.ru/docs)
- [Docker Desktop](https://www.docker.com/products/docker-desktop)
- [Prometheus](https://prometheus.io)
- [Grafana](https://grafana.com)
- [Blackbox Exporter](https://github.com/prometheus/blackbox_exporter)

### 🛠️ Инструменты

- [Tailscale VPN](https://tailscale.com)
- [Portainer](https://docs.portainer.io)
- [Обновлятор 1С](https://helpme1s.ru) (опционально, для бэкапа 1С-ных баз)

### 🌐 Ресурсы автора

- 📘 [InfoStart профиль](https://infostart.ru/profile/348559/)
- 💬 [ВКонтакте: "Автоматизация бизнес-процессов"](https://vk.com/club230942526)
- 🎥 [Rutube плейлисты](https://rutube.ru/channel/766472/)
- 🔗 [GitHub репозиторий](https://github.com/VladimirProgrammist1C)

### 📄 Документация проекта

| Файл | Описание |
|------|----------|
| [📘 infrastructure-guide.md](infrastructure-guide.md) | Полное руководство (этот файл) |
| [⚡ COMMANDS.md](COMMANDS.md) | Шпаргалка по командам |
| [🏠 README.md](../README.md) | Быстрый старт |
| [📜 scripts/README.md](../scripts/README.md) | Система бэкапов |

> 💡 **Полная история проекта** (тайминг, проблемы, инсайты, статистика):  
> Опубликована в статье на InfoStart:  
> 🔗 [DevOps для 1С на практике: как я развернул домашний сервер за 14 дней и 32 часа](https://infostart.ru/1c/articles/2658161/)

[🔝 Наверх](#top)

---

## 📝 Changelog

### 2026-08-16 (v3.0)
- ✅ Очищен от специфичных сервисов (Gitea, Analytics, Email fallback)
- ✅ Удалены бизнес-метрики 1С (требуют расширенного postgres-exporter)
- ✅ Добавлен 4-этапный интерактивный скрипт бэкапа
- ✅ Приведены в соответствие имена сервисов (vocechat-notifications)
- ✅ Актуализированы имена volumes с динамическим поиском

### 2026-04-03 (v2.4)
- ✅ Добавлен мониторинг (Grafana + Prometheus + Blackbox)
- ✅ Добавлен postgres-exporter для метрик PostgreSQL
- ✅ Настроены алерты с уведомлениями в VoceChat

### 2026-03-30 (v2.3)
- ✅ Tailscale VPN для безопасного удалённого доступа
- ✅ Финальный docker-compose.yml с оркестрацией
- ✅ Подключение 1С:Предприятие к PostgreSQL в Docker

### 2026-03-27 (v2.2)
- ✅ Финализация СУБД (PostgreSQL + Portainer + pgAdmin)
- ✅ Healthcheck для PostgreSQL
- ✅ Именованные volumes для переносимости

[🔝 Наверх](#top)

---

## 📞 Поддержка

При возникновении проблем:

1. Проверьте [⚡ COMMANDS.md](COMMANDS.md) — шпаргалка по командам
2. Изучите раздел [Диагностика](#диагностика)
3. Проверьте логи сервисов: `docker-compose logs <service> --tail 100`
4. Проверьте статус алертов: [http://localhost:3002/alerting/list](http://localhost:3002/alerting/list)
5. Проверьте targets в Prometheus: [http://localhost:9090/targets](http://localhost:9090/targets)

[🔝 Наверх](#top)
