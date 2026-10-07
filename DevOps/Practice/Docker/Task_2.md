**Dockerfile:**

```dockerfile
FROM cr.yandex/mirror/python:3.11-slim
WORKDIR /app
ENV REGISTRY="Yandex Mirror"
COPY app.py .
RUN groupadd -r pyuser && \
    useradd -r -g pyuser -u 1001 pyuser && \
    chown -R pyuser:pyuser /app
USER 1001
EXPOSE 8080
CMD ["python3","app.py"]
```

**Проверка работы:**

- `docker build -t my_ya_app .`
    
- `docker run -d --name yandex-app -p 8080:8080 my_ya_app`
    
- `curl http://localhost:8080`

```
<html>
        <head><title>Task 2</title></head>
        <body style="font-family: Arial; text-align: center; padding: 50px;">
            <h1>Задание 2 выполнено!</h1>
            <h2>Yandex Mirror Registry</h2>
            <p>Registry: <strong>Yandex Mirror</strong></p>
            <p>Базовый образ: <code>cr.yandex/mirror/python:3.11-slim</code></p>
        </body>
</html>
```

- `docker exec yandex-app env | grep REGISTRY`  
    `REGISTRY=Yandex Mirror`