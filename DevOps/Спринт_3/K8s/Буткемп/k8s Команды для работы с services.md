  
```bash
kubectl get services                        # список сервисов
kubectl describe service web                # детали, селектор, эндпоинты
kubectl get endpoints web                   # адреса подов за сервисом
kubectl run tmp --rm -it --image=busybox:1.36 -- sh   # временный под для проверки
kubectl expose deployment web --port=80     # создать сервис императивно
kubectl port-forward service/web 8080:80    # туннель до сервиса для отладки
kubectl delete service web                  # удалить сервис
```