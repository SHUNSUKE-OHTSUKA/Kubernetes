# kubernetes

## 開発環境

k8s on minikube on Docker on Ubunts というわかりにくい環境となっています。

- OS : Ubuntu 22.04.3 LTS on WSL2(Windows 11 Home)
- Docker : v24.0.5
- minikube : v1.31.1
- Kubernetes : v1.27.3
- pip : 22.0.2
- ArgoCD CLI : v2.8.0

## 開発環境構築

```
pip install -r requirements.txt
pre-commit install
```

その他、以下を前提とします。

- ワイルドカードの名前解決設定 ( *.minikube.local <-> $(minikube ip) )
- Webブラウザ環境 ( Google Chrome )

## ArgoCDのセットアップ

### ArgoCDのk8sリソース作成

```
kubectl apply -f ./init
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
ADMIN_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d); echo $ADMIN_PASS
```

### 自己証明書作成/SSL化

```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -out ./k8s-setup/k8s.crt -keyout ./k8s-setup/k8s.key -subj "/CN=*.minikube.local/O=org"
kubectl create secret tls argocd-cert-secret --key ./k8s-setup/k8s.key --cert ./k8s-setup/k8s.crt --dry-run=client -n argocd -o yaml > ./k8s-setup/argocd-cert-secret.yaml
kubectl apply -f ./k8s-setup
```

### ユーザ作成/RBAC設定/パスワード修正

```
kubectl apply -n argocd -f argocd-manage
argocd login argocd.minikube.local:443 --grpc-web --username admin --password $ADMIN_PASS
argocd account update-password --grpc-web --account dev --current-password $ADMIN_PASS --new-password $ADMIN_PASS
argocd account update-password --grpc-web --account ope --current-password $ADMIN_PASS --new-password $ADMIN_PASS
argocd account update-password --grpc-web --account sync --current-password $ADMIN_PASS --new-password $ADMIN_PASS
```

### ログイン確認

WebブラウザにてArgoCDにログインする。

* URL: https://argocd.minikube.local/
* Username: admin/dev/ope
* Password: <echo $ADMIN_PASS>

## ArgoCDでGitOps

### Project/Application作成

```
kubectl apply -f argocd-apps/webservice1/
```
