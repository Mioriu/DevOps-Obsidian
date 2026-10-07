kubectl get nodes                      # ноды кластера
kubectl get pods                       # поды
kubectl get pods -o wide               # поды с нодами и IP
kubectl run web --image=nginx:1.25     # запустить под императивно
kubectl apply -f pod.yaml              # применить манифест
kubectl describe pod web               # детали и события
kubectl logs -f web                    # логи
kubectl exec -it web -- bash           # зайти внутрь
kubectl port-forward pod/web 8080:80   # туннель для отладки
kubectl get pod web -o yaml            # объект целиком, spec и status
kubectl delete -f pod.yaml             # удалить по манифесту
