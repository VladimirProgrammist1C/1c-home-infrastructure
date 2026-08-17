# 🏠 1C Home Infrastructure

Инфраструктура домашнего сервера для 1С-разработки на базе **Geekom A9 Max** (Ryzen AI 9 HX 370, Windows 11 Pro).

[![InfoStart](https://img.shields.io/badge/InfoStart-статья-blue)](https://infostart.ru/1c/articles/2658161/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

> **Цель:** Создание изолированной среды для 1С (PostgreSQL, Monitoring, Backups, VPN) с приближением к продакшену.

## 📦 Компоненты

### Основные сервисы
- **СУБД:** PostgreSQL 18.1-2.1C (сборка 1С, русская локаль `ru_RU.UTF-8`)
- **Мониторинг:** Prometheus + Grafana + Blackbox Exporter + cAdvisor
- **Уведомления:** VoceChat (локальный мессенджер)
- **Управление:** Portainer CE, pgAdmin 4

### Автоматизация
- **Бэкапы:** Интерактивный 4-этапный скрипт (конфиги, БД, volumes)
- **Доступ:** Tailscale VPN (без проброса портов)

## 🚀 Быстрый старт

### Требования
- Windows 10/11 Pro с поддержкой WSL2 / Docker Desktop
- ~16 ГБ свободной оперативной памяти
- Дистрибутив PostgreSQL для 1С (скачивается с ИТС)

### Установка

1. **Клонируйте репозиторий:**
   ```bash
   git clone <url-репозитория>
   cd 1c-home-infrastructure
   ```

2. **Настройте переменные окружения:**
   ```powershell
   Copy-Item .env.example .env
   # Отредактируйте .env, указав свои пароли
   ```

3. **Подготовьте PostgreSQL 1С:**
   - Скачайте архив `postgresql_18.1_2_ubuntu_22.04_x86_64_package.tar.bz2` с [releases.1c.ru](https://releases.1c.ru).
   - Создайте папку `docker/postgres-1c/` и положите архив туда.
   - Добавьте `Dockerfile` и `entrypoint.sh` (см. [полное руководство](docs/infrastructure-guide.md)).
   - Соберите образ:
     ```powershell
     docker build -t postgres:18.1-2.1C-ubuntu2204 ./docker/postgres-1c
     ```

4. **Запустите инфраструктуру:**
   ```powershell
   docker-compose up -d
   ```

5. **Проверьте статус:**
   ```powershell
   docker-compose ps
   ```

## 🌐 Доступ к сервисам

| Сервис | Адрес | Логин / Пароль |
| :--- | :--- | :--- |
| **PostgreSQL** | `localhost:5432` | `postgres` / из `.env` |
| **Portainer** | http://localhost:9000 | *Создаётся при первом входе* |
| **pgAdmin** | http://localhost:5050 | из `.env` |
| **Grafana** | http://localhost:3002 | из `.env` |
| **Prometheus** | http://localhost:9090 | *без авторизации* |
| **VoceChat** | http://localhost:3001 | из `.env` |
| **Blackbox** | http://localhost:9115 | *без авторизации* |

> **Удалённый доступ:** через **Tailscale VPN** по адресу `100.x.x.x:port` — безопасно, без проброса портов.

## 📂 Структура проекта

```text
📦 1c-home-infrastructure
├── docker-compose.yml          # Оркестрация всех сервисов
├── .env.example                # Шаблон переменных окружения
├── .gitignore                  # Исключения для Git
├── monitoring/                 # Prometheus, Grafana, алерты, blackbox
├── scripts/                    # Скрипты автоматизации
│   ├── README.md               # Описание скриптов
│   └── backup-scripts/         # Интерактивный бэкап (4 этапа)
├── docs/                       # Документация
│   ├── infrastructure-guide.md # Полное руководство
│   ├── COMMANDS.md             # Шпаргалка по командам
│   ├── backup-test-demo.md     # Видео: тестовый режим бэкапа
│   └── backup-full-demo.md     # Видео: полный бэкап
└── README.md                   # Этот файл
```

## 💾 Система бэкапов

В репозитории есть **интерактивный скрипт** `scripts/backup-scripts/full-backup-rus.ps1` с 4 этапами:

1. **Конфиги проекта** (robocopy)
2. **Не-1С базы PostgreSQL** (pg_dump + gzip, с whitelist)
3. **Конфиги PostgreSQL** (postgresql.conf, pg_hba.conf)
4. **Docker volumes** (tar.gz архивы)

1С-ные базы бэкапятся через Обновлятор 1С или любой другой сервис/скрипт/инструмент.

📹 [Видеодемонстрация работы скрипта](docs/backup-full-demo.md)

## 📚 Документация

- 📘 **[Полное руководство](docs/infrastructure-guide.md)** — детальное описание развёртывания и настройки.
- ⚡ **[Шпаргалка по командам](docs/COMMANDS.md)** — быстрый справочник (Docker, PowerShell, SQL).
- 🎥 **Видео:**
  - [Тестовый режим бэкапа](docs/backup-test-demo.md) (~3 мин)
  - [Полный бэкап](docs/backup-full-demo.md) (~10 мин)

## 📰 Опубликовано на InfoStart

[![InfoStart](https://infostart.ru/bitrix/templates/sandbox_empty/assets/tpl/abo/img/logo.svg)](https://infostart.ru/1c/articles/2658161/)

- 📄 **[DevOps для 1С на практике: как я развёртывал домашний сервер за 14 дней и 32 часа](https://infostart.ru/1c/articles/2658161/)** — полная история проекта с таймингом и «граблями».
- 📄 **[Интеграция 1С:ЗУП 3.1 с локальным мессенджером VoceChat](https://infostart.ru/1c/articles/2680511/)** — бизнес-уведомления из 1С в собственный чат.

## 🔗 Связанные проекты

- **[1C ZUP VoceChat Integration](https://github.com/VladimirProgrammist1C/1c-zup-vocechat-integration)** — расширение для интеграции ЗУП с мессенджером VoceChat.
- **[Grafinya Monitoring Stack](https://github.com/VladimirProgrammist1C/grafinya-monitoring-stack)** — импортозамещённый контур мониторинга (Графиня + Victoria Metrics), параллельный стек.

## 🛠️ Технологии

- Docker & Docker Compose
- PostgreSQL (1C Build, ru_RU.UTF-8)
- Prometheus + Grafana + Blackbox Exporter
- Tailscale (WireGuard VPN)
- PowerShell (Automation)
- VoceChat (on-premise мессенджер)

## 👤 Автор

**Vladimir Bessonov**

- 📧 Email: bessonov_1989@list.ru
- 🔗 [InfoStart](https://infostart.ru/profile/348559/)
- 🔗 [GitHub](https://github.com/VladimirProgrammist1C)
- 🔗 [ВКонтакте](https://vk.com/club230942526)
- 🔗 [Rutube](https://rutube.ru/channel/766472/)

## 📄 Лицензия

MIT — используйте, модифицируйте, делитесь.

---

**Время развёртывания:** ~30 минут (при наличии дистрибутива PostgreSQL 1С)  
**Последнее обновление:** 15 августа 2026