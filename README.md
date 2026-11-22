## 4 балла

Команда для запуска

```sh
docker build -f Dockerfile1 -t hw3-counter:latest .
docker run -d \
  --name hw3-counter-app \
  -p 8080:8000 \
  -e 'GREETING=hello' \
  hw3-counter:latest
```

Проверить состояние контейнера:

```sh
docker ps --filter name=hw3-counter-app
```

Проверка приложения

```sh
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS http://127.0.0.1:8080/
curl -fsS http://127.0.0.1:8080/readyz
```

Фактический результат:

```text
GET /              -> greeting="Привет из Docker!", count=0
POST /increment    -> greeting="Привет из Docker!", count=1
POST /increment    -> greeting="Привет из Docker!", count=2
GET /              -> greeting="Привет из Docker!", count=2
GET /readyz        -> status="ready", storage="sqlite"
```

Команда:

```sh
docker logs hw3-counter-app
```

Логи:

```text
2026-11-22 17:39:17,932 listening port=8000 storage=sqlite
2026-11-22 17:39:58,800 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-22 17:40:01,387 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-22 17:40:04,163 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-22 17:40:06,238 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-22 17:40:08,211 http 172.17.0.1 "GET /readyz HTTP/1.1" 200 -
```

Остановить и удалить контейнер после проверки:

```sh
docker stop hw3-counter-app
docker rm hw3-counter-app
```

## 6 баллов

Запуск:

```sh
docker compose -f compose2.yaml up -d --build
```

При настройках из `.env.example` приложение доступно по адресу
`http://127.0.0.1:8080`.

Для запуска примера можно указать файл:

```sh
docker compose --env-file .env.example -f compose2.yaml up -d --build
```

Фактический вывод `docker compose -f compose2.yaml ps`:

```text
NAME                  IMAGE               COMMAND           SERVICE   STATUS       PORTS
hw_3_template-app-1   hw_3_template-app   "python app.py"   app       Up           0.0.0.0:8080->8000/tcp, [::]:8080->8000/tcp
```

### Проверка приветствия и счётчика

```sh
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS http://127.0.0.1:8080/
```

Фактический результат до пересоздания контейнера:

```text
GET /           -> greeting="hello", count=0
POST /increment -> greeting="hello", count=1
POST /increment -> greeting="hello", count=2
POST /increment -> greeting="hello", count=3
GET /           -> greeting="hello", count=3,
                   hostname="96d541f22d18"
```

### Проверка сохранения данных

Удалить контейнер и сеть, не удаляя именованный volume:

```sh
docker compose -f compose2.yaml down
```

Проверить, что volume остался:

```sh
docker volume ls --filter name=hw_3_template_counter-data
```

Фактический результат:

```text
DRIVER    VOLUME NAME
local     hw_3_template_counter-data
```

Повторно создать контейнер и прочитать счётчик:

```sh
docker compose -f compose2.yaml up -d
curl -fsS http://127.0.0.1:8080/
```

Ответ нового контейнера:

```json
{"greeting": "hello", "count": 3, "hostname": "6b7ab3fb092a", "storage": "sqlite"}
```
### Логи после повторного создания

```sh
docker compose -f compose2.yaml logs app
```

```text
app-1  | 2026-11-22 18:01:41,006 listening port=8000 storage=sqlite
app-1  | 2026-11-22 18:01:51,964 http 172.20.0.1 "GET / HTTP/1.1" 200 -
```

## 8 баллов

### Запуск

Собрать образы, запустить сервисы и дождаться обоих healthcheck:

```sh
docker compose --env-file .env.example -f compose3.yaml up -d --build \
  --wait --wait-timeout 120
```

Посмотреть состояние сервисов:

```sh
docker compose --env-file .env.example -f compose3.yaml ps
```

Фактический результат:

```text
NAME                    IMAGE               SERVICE   STATUS                    PORTS
hw_3_template-app-1     hw_3_template-app   app       Up (healthy)              0.0.0.0:8080->8000/tcp
hw_3_template-redis-1   redis:7.4-alpine    redis     Up (healthy)              6379/tcp
```

У Redis в столбце `PORTS` нет привязки вида `0.0.0.0:...`: порт `6379`
доступен только внутри Compose-сети. На хост опубликован только порт `app`.

Проверить включение AOF:

```sh
docker compose --env-file .env.example -f compose3.yaml exec -T redis \
  redis-cli CONFIG GET appendonly
```

```text
appendonly
yes
```

### Проверка счётчика

```sh
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS -X POST http://127.0.0.1:8080/increment
curl -fsS http://127.0.0.1:8080/
```

Фактический результат:

```text
GET /           -> storage="redis", count=0
POST /increment -> storage="redis", count=1
POST /increment -> storage="redis", count=2
POST /increment -> storage="redis", count=3
POST /increment -> storage="redis", count=4
GET /           -> storage="redis", count=4, hostname="02cc256f5829"
```

Для проверки сохранения данных записано `N=4`.

### Проверка сохранения данных после пересоздания

Удалить контейнеры и сеть, не удаляя volume:

```sh
docker compose --env-file .env.example -f compose3.yaml down
```

Проверить наличие Redis-volume:

```sh
docker volume ls --filter name=hw_3_template_redis-data
```

```text
DRIVER    VOLUME NAME
local     hw_3_template_redis-data
```

Повторно создать сервисы, дождаться готовности и прочитать счётчик:

```sh
docker compose --env-file .env.example -f compose3.yaml up -d \
  --wait --wait-timeout 120
curl -fsS http://127.0.0.1:8080/
```

Фактический ответ:

```json
{"greeting": "hello", "count": 4, "hostname": "a33b4d7d1fae", "storage": "redis"}
```

Новый контейнер вернул сохранённое значение `N=4`.

### Проверка при остановленном Redis

Остановить только Redis:

```sh
docker compose --env-file .env.example -f compose3.yaml stop redis
```

Проверить HTTP-коды. Тайм-аут 30 секунд оставляет Redis-клиенту приложения
время завершить встроенные попытки подключения:

```sh
curl -sS -i --max-time 30 http://127.0.0.1:8080/healthz
curl -sS -i --max-time 30 http://127.0.0.1:8080/readyz
curl -sS -i --max-time 30 -X POST http://127.0.0.1:8080/increment
```

Фактический результат:

| Запрос | HTTP-код | Ответ |
| --- | ---: | --- |
| `GET /healthz` | 200 | `{"status": "alive"}` |
| `GET /readyz` | 503 | `{"error": "storage_unavailable"}` |
| `POST /increment` | 503 | `{"error": "storage_unavailable"}` |

Приложение не остановилось: независимый от хранилища `/healthz` продолжил
отвечать с кодом 200.

### Проверка восстановления Redis

Зафиксировать ID контейнера приложения и запустить только Redis:

```sh
docker compose --env-file .env.example -f compose3.yaml ps -q app
docker compose --env-file .env.example -f compose3.yaml start redis
```

Дождаться готовности существующих контейнеров без их пересоздания:

```sh
docker compose --env-file .env.example -f compose3.yaml up -d --no-recreate \
  --wait --wait-timeout 120
docker compose --env-file .env.example -f compose3.yaml ps
docker compose --env-file .env.example -f compose3.yaml ps -q app
```

ID приложения до и после запуска Redis совпал:

```text
a33b4d7d1fae2fed15d85a1474ac5256143306213edeacb14ab4f27e6ddf9d9c
```

Следовательно, приложение не перезапускалось. Проверка данных:

```sh
curl -fsS http://127.0.0.1:8080/
curl -fsS -X POST http://127.0.0.1:8080/increment
```

```text
GET /           -> count=4
POST /increment -> count=5
```

После восстановления Redis приложение вернуло прежнее `N=4`, а следующий
инкремент — `N+1=5`.

### Логи проверки

```sh
docker compose --env-file .env.example -f compose3.yaml logs app redis
```

Фрагмент логов приложения при отказе и восстановлении Redis:

```text
app-1  | 2026-11-22 18:18:17,340 storage_unavailable: ConnectionError
app-1  | 2026-11-22 18:18:17,341 http 172.20.0.1 "GET /readyz HTTP/1.1" 503 -
app-1  | 2026-11-22 18:18:28,318 storage_unavailable: ConnectionError
app-1  | 2026-11-22 18:18:28,319 http 172.20.0.1 "POST /increment HTTP/1.1" 503 -
app-1  | 2026-11-22 18:19:18,679 http 172.20.0.1 "GET / HTTP/1.1" 200 -
app-1  | 2026-11-22 18:19:22,800 http 172.20.0.1 "POST /increment HTTP/1.1" 200 -
```

Фрагмент логов Redis подтверждает загрузку данных с диска:

```text
redis-1  | 1:M 22 Nov 2026 18:18:45.744 * DB loaded from base file appendonly.aof.1.base.rdb: 0.001 seconds
redis-1  | 1:M 22 Nov 2026 18:18:45.745 * DB loaded from incr file appendonly.aof.1.incr.aof: 0.000 seconds
redis-1  | 1:M 22 Nov 2026 18:18:45.745 * DB loaded from append only file: 0.001 seconds
redis-1  | 1:M 22 Nov 2026 18:18:45.745 * Ready to accept connections tcp
```
