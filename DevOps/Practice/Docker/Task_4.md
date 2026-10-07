**Файл `.env`**
```env
DB_HOST=db
DB_PORT=5432
APP_NAME=backend
POSTGRES_PASSWORD=secret
POSTGRES_USER=dbuser
POSTGRES_DB=mydb
BACKEND_URL=http://backend:5000

```

**docker-compose.yaml**

```yaml
services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: backend_app
    environment:
      - APP_NAME=${APP_NAME}
      - DB_HOST=${DB_HOST}
      - DB_PORT=${DB_PORT}
    networks:
      - app-network
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_USER=${POSTGRES_USER:-postgres}
      - POSTGRES_DB=${POSTGRES_DB:-postgres}
    volumes:
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
      - pgdata:/var/lib/postgresql/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "${POSTGRES_USER}", "-d", "${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 3
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: frontend_app
    ports:
      - "8080:8080"
    environment:
      - APP_NAME=${APP_NAME}
      - BACKEND_URL=${BACKEND_URL}
    networks:
      - app-network
    depends_on:
      - backend
volumes:
  pgdata:
networks:
  app-network:
    driver: bridge
```

**Запуск**

docker compose up -d --build

**Проверка работы**

```bash
docker compose ps

NAME             IMAGE                COMMAND                  SERVICE    CREATED          STATUS                    PORTS
backend_app      practice4-backend    "python3 app.py"         backend    17 minutes ago   Up 17 minutes             5000/tcp
frontend_app     practice4-frontend   "python app.py"          frontend   17 minutes ago   Up 17 minutes   0.0.0.0:8081->8080/tcp, [::]:8081->8080/tcp
practice4-db-1   postgres:15-alpine   "docker-entrypoint.s…"   db         17 minutes ago   Up 17 minutes (healthy)   5432/tcp
```

Открыл в браузере: `http://localhost:8080`, всё ок
