## 4 балла

Команда для запуска

```sh
docker build -t hw3-counter:latest .
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

