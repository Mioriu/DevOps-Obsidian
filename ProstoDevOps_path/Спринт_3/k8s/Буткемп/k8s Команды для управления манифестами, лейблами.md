## Шпаргалка

Bash

Копировать

```bash
kubectl api-resources                          # типы объектов, их группы и сокращения
kubectl explain pod.spec.containers            # документация по полю
kubectl apply -f manifests/                    # применить директорию
kubectl get pods --show-labels                 # поды с метками
kubectl get pods -l app=web                    # выборка по метке
kubectl label pod NAME key=value               # навесить метку
kubectl label pod NAME key-                    # снять метку
kubectl apply -f file.yaml --dry-run=server    # проверить без применения
kubectl diff -f file.yaml                      # что изменится
kubectl create deployment web --image=nginx --dry-run=client -o yaml   # скелет манифеста