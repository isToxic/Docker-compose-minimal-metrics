# Docker Compose: минимальная обвязка для логов, метрик и трейсов

Набор готовых Docker Compose-конфигураций и конфигов для быстрого развёртывания observability-стека. Репозиторий хранится как личная «копилка» для повторного использования в проектах: можно взять нужный compose/конфиг и подключить к своему приложению.

## Что входит в стек

`docker/compose.yaml` поднимает следующие сервисы:

| Сервис | Образ | Назначение | Порты (host) |
|---|---|---|---|
| `postgres` | `postgres:17` | База данных для приложения | `5432` |
| `app` | собирается из `../Dockerfile` | Ваше приложение (шаблон) | `8080` |
| `otel-collector` | `otel/opentelemetry-collector-contrib` | Приём OTLP-телеметрии, батчинг и экспорт трейсов | `4317` (gRPC) |
| `tempo` | `grafana/tempo:latest` | Хранилище и API трассировок | `3200`, `5327`→`4317`, `4318` |
| `prometheus` | `prom/prometheus:latest` | Сбор метрик | `9090` |
| `grafana` | `grafana/grafana:10.4.1` | Визуализация метрик/логов/трейсов | `3000` |
| `loki` | `grafana/loki:latest` | Хранилище логов | `3100` |
| `promtail` | `grafana/promtail:latest` | Сбор Docker-логов и отправка в Loki | — |

## Схема потоков данных

- **Метрики**: Prometheus скрейпит `otel-collector:8889` (job `otel-collector`) и `app:8080/actuator/prometheus` (job `app`, формат Spring Boot Actuator).  
  См. [`docker/prometheus.yml`](docker/prometheus.yml).
- **Трейсы**: приложение/клиенты отправляют OTLP → `otel-collector` → экспорт в `tempo:4317`.  
  См. [`docker/otel-collector-config.yaml`](docker/otel-collector-config.yaml), [`docker/tempo.yaml`](docker/tempo.yaml).
- **Логи**: Promtail читает логи Docker-контейнеров через `/var/run/docker.sock` и `/var/lib/docker/containers`, добавляет label `container`, отправляет в Loki.  
  См. [`docker/promtail-config.yaml`](docker/promtail-config.yaml).
- **Grafana** зависит от Prometheus и Tempo; provisioning ожидается в `docker/provisioning` (в репозитории не включён).

## Структура репозитория

```text
.
├── docker/
│   ├── compose.yaml                # основной compose-файл
│   ├── prometheus.yml              # scrape-конфиг Prometheus
│   ├── otel-collector-config.yaml  # OTLP receiver → batch → Tempo/debug
│   ├── tempo.yaml                  # конфиг Tempo (OTLP + local storage)
│   └── promtail-config.yaml        # сбор Docker-логов → Loki
├── .gitignore
├── LICENSE
└── README.md
```

## Перед запуском

`compose.yaml` рассчитан как шаблон и ссылается на файлы/ресурсы, которых нет в этом репозитории (они игнорируются или создаются отдельно в целевом проекте):

1. **Секреты** (пути относительно `docker/`):
   - `./secrets/postgres-secrets.env`
   - `./secrets/grafana-secrets.env`
   - `./secrets/app-secrets.env`

   Пример содержимого (адаптируйте под себя):

   ```env
   # secrets/postgres-secrets.env
   POSTGRES_DB=app
   POSTGRES_USER=app
   POSTGRES_PASSWORD=change-me
   ```

   ```env
   # secrets/grafana-secrets.env
   GF_SECURITY_ADMIN_USER=admin
   GF_SECURITY_ADMIN_PASSWORD=change-me
   ```

2. **Приложение `app`**: сервис собирается из `../Dockerfile` (относительно `docker/`) и монтирует volume `gradle-cache`. Если нужен только observability-стек, удалите/закомментируйте сервис `app` и job `app` в `prometheus.yml`.

3. **Grafana provisioning**: подключается `./provisioning:/etc/grafana/provisioning`. Создайте каталог `docker/provisioning` при необходимости.

4. **Права доступа Promtail**: контейнер монтирует Docker socket и каталог контейнеров (`/var/lib/docker/containers`) в режиме чтения. На некоторых системах могут потребоваться дополнительные права.

## Запуск

```bash
cd docker
docker compose up -d
```

Остановить:

```bash
docker compose down
```

С удалением томов (данные Postgres/Grafana/Loki/Tempo будут потеряны):

```bash
docker compose down -v
```

## Полезные адреса (по умолчанию)

- Grafana: http://localhost:3000
- Prometheus: http://localhost:9090
- Tempo API: http://localhost:3200
- Loki: http://localhost:3100
- OTLP gRPC (collector): `localhost:4317`
- OTLP gRPC/HTTP (Tempo, проброшено на host): `localhost:5327` / `localhost:4318`

## Замечания

- В `compose.yaml` и `otel-collector`/`tempo` есть пересечения по OTLP-портам на хосте (`4317` используется коллектором; Tempo также пробрасывает `5327:4317` и `4318:4318`). При интеграции в другой проект проверьте конфликты портов.
- Часть образов использует тег `latest` (`tempo`, `prometheus`, `loki`, `promtail`, `otel-collector-contrib`). Для воспроизводимости в production лучше закрепить версии.
- Для Tempo в текущем конфиге используется локальное хранилище в `/tmp/tempo/...` внутри контейнера; том `tempo_data` объявлен, но не подключён к сервису `tempo`. При необходимости подключите volume для персистентности.
- Секреты не должны попадать в git: правило `.gitignore` исключает `**/docker/secrets/**`.

## Лицензия

Проект распространяется по лицензии [Apache License 2.0](LICENSE).
