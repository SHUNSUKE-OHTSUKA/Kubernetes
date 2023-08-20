# kubernetes

## 開発環境

k8s on minikube on Docker on Ubunts というわかりにくい環境となっています。

- OS : Ubuntu 22.04.3 LTS on WSL2(Windows 11 Home)
- Docker : v24.0.5
- minikube : v1.31.1
- Kubernetes : v1.27.3
- pip : 22.0.2
- ArgoCD CLI : v2.8.0

## 環境構築

```
pip install -r requirements.txt
pre-commit install
```

## ArgoCD起動

```
kubectl apply -f ./init
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
ADMIN_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d); echo $ADMIN_PASS
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

http://localhost:8080にWebブラウザでログインする。

* Username: admin
* Password: <echo $ADMIN_PASS>

## ユーザ作成/RBAC設定/AppProjectの作成

```
kubectl apply -n argocd -f argocd
```

### ユーザのパスワード修正
```
argocd login --insecure localhost:8080 --username admin --password $ADMIN_PASS
argocd account update-password --account dev --current-password $ADMIN_PASS --new-password $ADMIN_PASS
argocd account update-password --account ope --current-password $ADMIN_PASS --new-password $ADMIN_PASS
argocd account update-password --account sync --current-password $ADMIN_PASS --new-password $ADMIN_PASS
```
