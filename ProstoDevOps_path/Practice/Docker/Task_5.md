**Что сделано**
- Установил эмуляторы QEMU  
  `docker run --rm --privileged multiarch/qemu-user-static --reset -p yes`
- Сбилдил сначала локально для своей архитектуры (для теста). Локально грузим только для своей архитектуры, для мульти нужно пушить в регистри.  
  `docker buildx build --platform linux/amd64 -t multiarch:local --load .`
- Залогинился в Docker Hub для пуша образа  
  `docker login -u 1mackk`
- Собрал и запушил мультиархитектурный образ  
```
  docker buildx build \
    --platform linux/amd64,linux/arm64 \
    -t 1mackk/multi_arch_pract:latest \
    --push \
    .
```

- Проверка образа на мультиархитектурность  
    `docker buildx imagetools inspect 1mackk/multi_arch_demo:latest`
    
- Запустил контейнер из мультиархитектурного образа и проверил работу
```
docker run -d -p 8081:8080 1mackk/multi_arch_pract:latest
curl -f http://localhost:8081
```
- Для теста попробовал загрузить в файл
```
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t multiarch:local \
  -o type=oci dest=./multiarch.tar \ 
  .
```
- Загружаем образ
`docker load -i multiarch.tar`
