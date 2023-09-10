# kubernetes

## 開発環境

k8s on minikube on Docker on Ubunts というわかりにくい環境となっています。

- OS : Ubuntu 22.04.2 LTS on WSL2(Windows 11 Home)
- Docker : v24.0.6
- minikube : v1.31.1
- Kubernetes : v1.27.4
- pip : 22.0.2
- ArgoCD CLI : v2.8.0

## 開発環境構築

```
pip install -r requirements.txt
pre-commit install
```

その他、以下を前提とします。

- ワイルドカードの名前解決設定 ( *.minikube.local <-> $(minikube ip) )
- Webブラウザ環境 ( Google Chrome, Mozilla Firefox )

## セットアップ

### minikubeクラスター作成

```
minikube start --driver=docker --memory=12288 --cpus=4
```

### Ingressコントローラー 有効化

```
minikube addons enable ingress
```

### ArgoCD k8sリソース作成

```
kubectl apply -f ./k8s-setup/namespace.yml
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
ARGOCD_ADMIN_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d); echo $ARGOCD_ADMIN_PASS
```

## ECK k8sリソース作成

```
# ECK
## https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html
kubectl create -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml

# Elasticsearch,Kibana
kubectl apply -f eck/
KIBANA_PASS=$(kubectl get secret quickstart-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode); echo $KIBANA_PASS
```

### 自己証明書作成/SSL化

```
# openssl genpkey -out ./k8s-setup/k8s.key -algorithm RSA -pkeyopt rsa_keygen_bits:2048
# openssl req -new -key ./k8s-setup/k8s.key -out ./k8s-setup/k8s.csr -subj "/CN=*.minikube.local"
# openssl x509 -in ./k8s-setup/k8s.csr -out ./k8s-setup/k8s.crt -req -signkey ./k8s-setup/k8s.key -days 365 -subj "/CN=*.minikube.local" -copy_extensions copy -extfile ./k8s-setup/subjectnames.txt

kubectl create secret tls argocd-cert-secret --key ./k8s-setup/k8s.key --cert ./k8s-setup/k8s.crt --dry-run=client -n argocd -o yaml > ./k8s-setup/argocd-cert-secret.yaml
kubectl create secret tls kibana-cert-secret --key ./k8s-setup/k8s.key --cert ./k8s-setup/k8s.crt --dry-run=client -o yaml > ./k8s-setup/kibana-cert-secret.yaml
kubectl apply -f ./k8s-setup
```

### ArgoCD ユーザ作成/RBAC設定/パスワード修正

```
kubectl apply -n argocd -f argocd-manage
argocd login argocd.minikube.local:443 --grpc-web --username admin --password $ARGOCD_ADMIN_PASS
argocd account update-password --grpc-web --account dev --current-password $ARGOCD_ADMIN_PASS --new-password $ARGOCD_ADMIN_PASS
argocd account update-password --grpc-web --account ope --current-password $ARGOCD_ADMIN_PASS --new-password $ARGOCD_ADMIN_PASS
argocd account update-password --grpc-web --account sync --current-password $ARGOCD_ADMIN_PASS --new-password $ARGOCD_ADMIN_PASS
```

### ログイン確認

WebブラウザにてArgoCD,Kibanaにログインする。

* ArgoCD
    * URL: https://argocd.minikube.local/
    * Username: admin/dev/ope
    * Password: <echo $ARGOCD_ADMIN_PASS>

* Kibana
    * URL: https://kibana.minikube.local/
    * Username: elastic
    * Password: <echo $KIBANA_PASS>

## GitOps

### Project/Application作成

```
kubectl apply -f argocd-apps/webservice1/
```
