  
```bash
kubectl create configmap NAME --from-literal=KEY=value    # из значения
kubectl create configmap NAME --from-file=nginx.conf      # из файла
kubectl create secret generic NAME --from-literal=KEY=value
kubectl get configmaps                                    # список
kubectl get secret NAME -o yaml                           # секрет целиком (base64!)
kubectl rollout restart deployment/NAME                   # перекатить после правки конфига
kubectl get namespaces                                    # пространства имен
kubectl create namespace staging
kubectl get pods -n staging                               # команды в пространстве
kubectl config set-context --current --namespace=staging  # рабочее пространство
```