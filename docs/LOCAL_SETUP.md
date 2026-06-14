# Локальный запуск BSPS

Пошаговое руководство для первого запуска инфраструктуры и приложений.

## 1. Предварительные требования

| Инструмент | Версия | Зачем |
|------------|--------|-------|
| [Docker Desktop](https://www.docker.com/products/docker-desktop/) | актуальная | Postgres, RabbitMQ, Loki, Grafana |
| Java JDK | 21 | Spring Boot сервисы |
| Maven | 3.9+ (или `mvnw` в репозиториях) | Сборка backend |
| Node.js | 20+ | React UI |
| Git | любая | Submodule `contracts` |

Структура папок — все репозитории рядом:

```
IdeaProjects/
├── bsps-infra/
├── application-ui/
├── application-gateway/
├── application-service/
└── client-service/
```

## 2. Подготовка контрактов

В каждом backend-сервисе инициализируйте submodule:

```powershell
cd ..\client-service
git submodule update --init --recursive

cd ..\application-service
git submodule update --init --recursive
```

## 3. Запуск инфраструктуры (Фаза 1)

```powershell
cd bsps-infra
copy .env.example .env
docker compose up -d
```

Дождитесь, пока все контейнеры станут `healthy`:

```powershell
docker compose ps
```

Ожидаемый результат — 8 контейнеров в статусе `running`, у Postgres/RabbitMQ/Loki/Grafana — `(healthy)`.

### Что происходит под капотом

1. Docker скачивает образы (`postgres:16`, `rabbitmq:3-management` и др.)
2. Создаётся сеть `bsps-network` — контейнеры видят друг друга по имени
3. Порты пробрасываются на хост: Postgres доступен как `localhost:5488`, а не `5432`
4. Promtail читает логи контейнеров и отправляет в Loki
5. Grafana подключается к Loki автоматически (provisioning)

## 4. Запуск приложений на хосте

Откройте **4 терминала**:

**Терминал 1 — client-service:**
```powershell
cd ..\client-service
.\mvnw.cmd spring-boot:run
```

**Терминал 2 — application-service:**
```powershell
cd ..\application-service
.\mvnw.cmd spring-boot:run
```

**Терминал 3 — application-gateway:**
```powershell
cd ..\application-gateway
.\mvnw.cmd spring-boot:run
```

**Терминал 4 — application-ui:**
```powershell
cd ..\application-ui
npm install
npm start
```

> Приложения подключаются к `localhost:5488`, `localhost:5489`, `localhost:5672` — это порты, проброшенные из Docker на ваш компьютер.

## 5. Проверка работоспособности

| Что | URL / команда |
|-----|---------------|
| Gateway health | http://localhost:8088/api/smoke |
| UI | http://localhost:3000 |
| RabbitMQ Management | http://localhost:15672 (guest / guest) |
| Grafana | http://localhost:3001 (admin / admin) |
| jsreport | http://localhost:5485 |
| Loki API | http://localhost:3100/ready |

### Smoke-тест gateway

```powershell
curl http://localhost:8088/api/smoke
```

Ожидается JSON со статусом `UP`.

## 6. Просмотр логов в Grafana

### Дашборд с селектором сервисов (рекомендуется)

1. Откройте http://localhost:3001 (логин `admin` / `admin`)
2. При входе откроется дашборд **BSPS Logs** (или: меню → Dashboards → BSPS → BSPS Logs)
3. Вверху два выпадающих списка:
   - **Сервис** — `client-service`, `application-gateway`, `rabbitmq` и др.
   - **Контейнер** — уточняет до конкретного контейнера (`bsps-client-service`)
4. Выберите сервис → логи обновятся автоматически (обновление каждые 10 сек)

Формат логов Spring-сервисов:
```
2025-06-25 15:22:51.575  INFO --- [main] r.k.c.ClientServiceApplication : Started ClientServiceApplication
```

### Explore (ручной режим)

```logql
{service="rabbitmq"}
```

```logql
{service="postgres-client"} |= "ERROR"
```

```logql
{container=~"bsps-.*"}
```

> В фазе 1 в Grafana видны логи **инфра-контейнеров**. Логи Spring-приложений, запущенных на хосте через Maven, сюда не попадают — они выводятся в терминал IDE. В фазе 2 (все в Docker) логи микросервисов тоже появятся в Grafana.

## 7. Остановка

```powershell
cd bsps-infra
docker compose down
```

Удалить данные БД (осторожно — сотрёт все данные):

```powershell
docker compose down -v
```

## 8. Troubleshooting

### Порт занят

```powershell
netstat -ano | findstr :5672
netstat -ano | findstr :5488
```

Если старые контейнеры из `client-service` или `application-service` compose ещё работают:

```powershell
docker compose down   # в тех репозиториях
```

### Spring не подключается к БД

- Убедитесь, что `docker compose ps` показывает Postgres как `healthy`
- Подождите 10–15 секунд после старта compose

### Submodule contracts пустой

```powershell
git submodule update --init --recursive
```

Директория `src/contracts` должна содержать Java-файлы.

### UI: CORS error

UI обращается к gateway напрямую (`REACT_APP_API_BASE_URL=http://localhost:8088` или `http://ВАШ_IP:8088`). В Docker-профиле gateway разрешает любой origin через `GatewayCorsConfig`.

После изменений в `application-gateway`:

```bash
cd /opt/bsps/application-gateway && git pull
cd /opt/bsps/bsps-infra
docker compose up --build -d application-gateway
```

Проверка preflight (ожидается `200` и `Access-Control-Allow-Origin`, не `403`):

```bash
curl -i -X OPTIONS "http://localhost:8088/api/v1/clients/search" \
  -H "Origin: http://144.31.61.170:3000" \
  -H "Access-Control-Request-Method: POST"
```

`403 Forbidden` + `Vary: Origin` — gateway **отклоняет** origin (часто разрешён только `localhost`). Пересоберите `application-gateway` (в профиле `docker` — `application-docker.yml` и `GatewayCorsConfig`).

`Restarting` / `JsonParseException: Cannot parse JSON` — в `docker-compose` был битый `SPRING_APPLICATION_JSON`; уберите его (`git pull` в `bsps-infra`) и перезапустите gateway.

**Локально без Docker** (`npm start`): `application-ui/.env` → `REACT_APP_API_BASE_URL=http://localhost:8088`.

### Grafana пустая

- Убедитесь, что `bsps-promtail` запущен: `docker compose ps`
- Подождите 30 секунд после старта — Promtail начнёт собирать логи
- Попробуйте запрос `{container=~".+"}` без фильтра по service

### Promtail не видит Docker socket (Windows)

Docker Desktop должен быть запущен. Promtail монтирует `/var/run/docker.sock` — на Windows это работает через Docker Desktop WSL2 backend.

### jsreport не отвечает

Первый старт может занять до минуты. Проверьте: `docker compose logs jsreport`

## 9. Полезные команды

```powershell
# Логи конкретного сервиса
docker compose logs -f rabbitmq

# Перезапуск одного контейнера
docker compose restart postgres-client

# Статус healthcheck
docker inspect --format='{{.State.Health.Status}}' bsps-postgres-client
```

## 10. Полный стек в Docker (Фаза 2)

Все сервисы (UI, gateway, backend) запускаются в контейнерах. Spring-профиль `docker` подставляет hostname'ы Docker-сети.

### Сборка и запуск

```powershell
cd bsps-infra
.\scripts\build-all.ps1
```

Первая сборка занимает 10–20 минут (Maven + npm). Повторные сборки быстрее за счёт кэша Docker.

Пересборка одного сервиса:

```powershell
docker compose up --build -d client-service
docker compose up --build -d application-gateway
```

### Проверка полного стека

| Что | URL |
|-----|-----|
| UI | http://localhost:3000 |
| Gateway smoke | http://localhost:8088/api/smoke |
| client-service API docs | http://localhost:8080/swagger-ui.html |
| application-service API docs | http://localhost:8082/swagger-ui.html |
| Grafana (логи всех контейнеров) | http://localhost:3001 |

В Grafana теперь видны логи микросервисов:

```logql
{service="client-service"} |= "ERROR"
{service="application-gateway"}
```

### Переключение между фазой 1 и 2

- **Фаза 1:** поднять только инфра, приложения на хосте — `docker compose up -d postgres-client postgres-application postgres-report rabbitmq loki grafana promtail jsreport`
- **Фаза 2:** полный стек — `docker compose up --build -d`

> Не запускайте одновременно приложения на хосте и в Docker на тех же портах (8080, 8082, 8088, 3000).
