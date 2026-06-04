**Dockerfile:**
```dockerfile
FROM mostafamoradian/xk6-kafka:latest
# Переключение на рута для скачивания nano
USER root
RUN apk update && apk add --no-cache nano
# Через docker history посмотрел базовый образ и установил юзера из базового образа,
# чтобы не оставлять рута
USER 12345
ENTRYPOINT ["/bin/sh"]
```

**Сборка и запуск:**

```
docker build -t mostafamoradian/xk6-kafka:custom .
# Запускаем обязательно в интерактивном режиме -it, иначе контейнер сразу остановится
docker run -dit --name kafka-test mostafamoradian/xk6-kafka:custom
```