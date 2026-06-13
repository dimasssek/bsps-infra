# BSPS Infrastructure

Единая точка запуска локальной инфраструктуры микросервисного проекта BSPS.

## Что поднимает compose

| Сервис | Порт | Назначение |
|--------|------|------------|
| postgres-client | 5488 | БД `client-service` |
| postgres-application | 5489 | БД `application-service` |
| postgres-report | 5487 | БД `report-service` (заготовка) |
| rabbitmq | 5672, 15672 | Message broker |
| jsreport | 5485 | Генерация отчётов (заготовка) |
| loki | 3100 | Хранилище логов |
| grafana | 3001 | UI для просмотра логов |

## Быстрый старт

```powershell
copy .env.example .env
docker compose up -d
docker compose ps
```

Подробная инструкция: [docs/LOCAL_SETUP.md](docs/LOCAL_SETUP.md)

Промпт для агента по Kubernetes: [docs/K8S_AGENT_PROMPT.md](docs/K8S_AGENT_PROMPT.md)

Деплой demo на VPS: [docs/DEMO_SERVER_DEPLOY.md](docs/DEMO_SERVER_DEPLOY.md)

## Структура репозиториев

```
IdeaProjects/
├── bsps-infra/           ← этот репозиторий
├── application-ui/
├── application-gateway/
├── application-service/
└── client-service/
```

## Фазы

- **Фаза 1:** инфраструктура в Docker, приложения на хосте (Maven / npm)
- **Фаза 2 (текущая):** полный стек в Docker — `docker compose up --build`

## Полный стек (фаза 2)

```powershell
.\scripts\build-all.ps1
```

Или по шагам:

```powershell
docker compose build
docker compose up -d
```
