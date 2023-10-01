# kubernetes

## 開発環境

k8s on minikube on Docker on Ubunts というわかりにくい環境となっています。

- OS : Ubuntu 22.04.2 LTS on WSL2(Windows 11 Home)
- Docker : v24.0.6
- minikube : v1.31.1
- Kubernetes : v1.27.4
- pip : 22.0.2
- ArgoCD CLI : v2.8.0
- kustomize : v5.1.1

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
```

## ECK k8sリソース作成

```
# ECK
## https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html
kubectl create -f https://download.elastic.co/downloads/eck/2.9.0/crds.yaml
kubectl apply -f https://download.elastic.co/downloads/eck/2.9.0/operator.yaml

# Elasticsearch,Kibana,Filebeat,Metricbeat,ElasticAPM
kubectl apply -f eck/
KIBANA_PASS=$(kubectl get secret monitoring-elasticsearch-es-elastic-user -n elastic-monitoring -o=jsonpath='{.data.elastic}' | base64 --decode); echo $KIBANA_PASS
APM_TOKEN=$( kubectl get secret/apm-server-apm-token -n elastic-monitoring -o go-template='{{index .data "secret-token" | base64decode}}' ); echo $APM_TOKEN
```

### k8s cert-manager導入,Server証明書作成,Ingress SSL/TLS化

```
# kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
kubectl apply -f ./k8s-setup/ca.yaml
kustomize build ./k8s-setup/overlays/argocd/ | kubectl apply -f -
kustomize build ./k8s-setup/overlays/elastic-apm/ | kubectl apply -f -
kustomize build ./k8s-setup/overlays/kibana/ | kubectl apply -f -

kubectl get secrets argocd-cert-secret -n argocd -o jsonpath='{.data.ca\.crt}' | base64 -d > ./k8s-setup/argocd-ca.crt
kubectl get secrets elastic-apm-cert-secret -n elastic-monitoring -o jsonpath='{.data.ca\.crt}' | base64 -d > ./k8s-setup/elastic-apm-ca.crt
kubectl get secrets elastic-apm-cert-secret -n elastic-monitoring -o jsonpath='{.data.tls\.crt}' | base64 -d > ./k8s-setup/elastic-apm-tls.crt
kubectl get secrets kibana-cert-secret -n elastic-monitoring -o jsonpath='{.data.ca\.crt}' | base64 -d > ./k8s-setup/kibana-ca.crt

cp -p  ./k8s-setup/elastic-apm-ca.crt  ./k8s-setup/elastic-apm-ca.pem
kubectl apply -f ./k8s-setup/ingress.yml
```

* Webブラウザに *-ca.crt を登録する。
* ElasticAPMを利用する場合はelastic-apm-ca.crt、elastic-apm-tls.crtをアプリケーションから参照する。

### ArgoCD ユーザ作成/RBAC設定/パスワード修正

```
kubectl apply -n argocd -f argocd-manage
ARGOCD_ADMIN_PASS=$(kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d); echo $ARGOCD_ADMIN_PASS
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

## Monitoring

* APM(Elastic APM)
    * URL :
        * https://www.elastic.co/guide/en/apm/guide/current/apm-quick-start.html#add-apm-integration-agents/
        * https://www.elastic.co/guide/en/apm/agent/go/current/configuration.html
    * 環境変数
        * ELASTIC_APM_SERVER_URL : https://elastic-apm.minikube.local/
        * ELASTIC_APM_SECRET_TOKEN : <echo $APM_TOKEN>
        * ELASTIC_APM_SERVER_CERT : elastic-apm-tls.crt のパス
        * ELASTIC_APM_SERVER_CA_CERT_FILE : elastic-apm-ca.crt のパス
        * ELASTIC_APM_VERIFY_SERVER_CERT : true
