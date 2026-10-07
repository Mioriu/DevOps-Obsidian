- Поднимаем: `docker compose up -d`
- Смотрим взаимодействие сервисов по сети
```bash
docker exec frontend ping -c 2 api
ping: bad address 'api'
docker exec worker ping -c 2 redis
ping: bad address 'redis'
```

**Тут всё ок, контейнеры в одной сети**
```bash
docker exec api ping -c 2 db
PING db (172.25.0.4): 56 data bytes
64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.071 ms
64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.058 ms
```

- Проверяем сеть `backend_net`
Тут по заданию требуется убрать `internal: true`, хотя в целом это нам не будет мешать для взаимодействия контейнеров внутри этой сети — они просто не будут доступны с хоста или извне (и наружу), но взаимодействовать между собой могут.

```bash
docker network inspect practice9_backend_net --format '{{.Internal}}'
true
```

**Восстанавливаем взаимодействие:**

1. Для Redis прописываем сеть `backend_net`, сеть `redis_net` не нужна
    
2. Добавляем для frontend сеть `backend_net`, чтобы он мог взаимодействовать с api
    
3. Заменяем IP адрес в коде сервиса worker на имя сервиса redis: `"Trying to connect to redis:6379"`
    
4. Убираем `internal: true` у сети `backend_net` (хотя в данном случае с ней тоже работало бы)
    

- Перепроверяем взаимодействие между сервисами
```bash
docker exec frontend ping -c 2 api
PING api (172.24.0.5): 56 data bytes
64 bytes from 172.24.0.5: seq=0 ttl=64 time=0.068 ms
64 bytes from 172.24.0.5: seq=1 ttl=64 time=0.332 ms

docker exec worker ping -c 2 redis
PING redis (172.24.0.4): 56 data bytes
64 bytes from 172.24.0.4: seq=0 ttl=64 time=0.072 ms
64 bytes from 172.24.0.4: seq=1 ttl=64 time=0.057 ms

docker exec api ping -c 2 db
PING db (172.24.0.3): 56 data bytes
64 bytes from 172.24.0.3: seq=0 ttl=64 time=0.103 ms
64 bytes from 172.24.0.3: seq=1 ttl=64 time=0.056 ms
```

**Итоговый `docker-compose.yaml`**

```yaml
version: '3.8'
services:
  redis:
    image: redis:alpine
    container_name: redis
    networks:
      - backend_net
  api:
    image: nginx:alpine
    container_name: api
    networks:
      - backend_net
  frontend:
    image: nginx:alpine
    container_name: frontend
    ports:
      - "8080:80"
    networks:
      - frontend_net
      - backend_net
  worker:
    image: alpine:latest
    container_name: worker
    command: sh -c "while true; do echo 'Trying to connect to redis:6379'; sleep 5; done"
    networks:
      - backend_net
  db:
    image: postgres:15-alpine
    container_name: db
    environment:
      POSTGRES_PASSWORD: secret
    networks:
      - backend_net
networks:
  redis_net:
    driver: bridge
  backend_net:
    driver: bridge
  frontend_net:
    driver: bridge
```

