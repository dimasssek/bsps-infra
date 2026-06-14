# BSPS Infrastructure

Единая точка запуска локальной инфраструктуры микросервисного проекта BSPS.

## Что поднимает compose

| Сервис | Порт | Назначение |
|--------|------|------------|
| postgres-client | 5488 | БД `client-service` |
| postgres-application | 5489 | БД `application-service` |
| postgres-report | 5487 | БД `report-service` |
| rabbitmq | 5672, 15672 | Message broker |
| jsreport | 5485 | Генерация отчётов |
| loki | 3100 | Хранилище логов |
| grafana | 3001 | UI для просмотра логов |

## Быстрый старт

```powershell
copy .env.example .env
docker compose up -d
docker compose ps
```

Подробная инструкция: [docs/LOCAL_SETUP.md](docs/LOCAL_SETUP.md)

```powershell
.\scripts\build-all.ps1
```

Или по шагам:

```powershell
docker compose build
docker compose up -d
```
