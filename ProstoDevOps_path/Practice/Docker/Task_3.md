**Создадим сеть**
```bash
docker network create myapp-net
```

**Соберём образы frontend и backend**
```bash
docker build -t myapp-backend:1.0 ./backend
docker build -t myapp-frontend:1.0 ./frontend
```
**Запустим контейнер с БД Postgres**

```bash
docker run -d \
  --name db \
  --network myapp-net \
  -e POSTGRES_PASSWORD=secret123 \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_DB=myappdb \
  --mount type=bind,source=./db/init.sql,target=/docker-entrypoint-initdb.d/init.sql \
  postgres:15-alpine
```

**Проверим работу контейнера db**
```bash
docker exec db psql -U appuser -d myappdb -c "SELECT * FROM users;"
```

**Запустим контейнер бэкенда**
```bash
docker run -d \
  --name backend \
  -e DB_HOST=db \
  -e DB_PORT=5432 \
  --network myapp-net \
  myapp-backend:1.0
```

**Запустим фронтенд с пробросом порта наружу (8080)**
```bash
docker run -d \
  --name frontend \
  --network myapp-net \
  -p 8080:8080 \
  -e BACKEND_URL=http://backend:5000 \
  myapp-frontend:1.0
```

**Убедимся, что все контейнеры запущены**
```bash
docker ps
```

**Проверяем в браузере** `localhost:8080` или `curl http://localhost:8080` — всё ок.

**Можем проверить логи и удалить контейнеры**

```bash
docker logs frontend
docker logs backend
docker logs db
```

```bash
docker stop frontend backend db && \
docker rm frontend backend db && \
docker network rm myapp-net && \
docker rmi myapp-frontend:1.0 myapp-backend:1.0
```

