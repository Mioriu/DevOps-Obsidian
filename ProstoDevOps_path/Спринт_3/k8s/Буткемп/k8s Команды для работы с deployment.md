kubectl apply -f deployment.yaml                  # создать или обновить
kubectl get deployments                           # список deployment
kubectl get rs                                    # его ReplicaSet
kubectl scale deployment web --replicas=5         # масштабировать (вне файла!)
kubectl rollout status deployment/web             # ход обновления
kubectl rollout history deployment/web            # список ревизий
kubectl rollout history deployment/web --revision=2
kubectl rollout undo deployment/web               # откат на прошлую ревизию
kubectl rollout undo deployment/web --to-revision=1
kubectl rollout restart deployment/web            # перекатить поды без смены версии
