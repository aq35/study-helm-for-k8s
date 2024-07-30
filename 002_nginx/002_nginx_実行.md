**Docker DesktopでKubernetesを有効にする**
Docker Desktopを開き、設定メニューから「Kubernetes」を選択し、「Enable Kubernetes」にチェックを入れます。

**Helmをインストールする**
brew install helm

**Helmチャートを作成する**
helm create nginx-chart
→ nginx-chartというディレクトリが作成され、その中にHelmチャートの基本構造が生成されます。

**Helmチャートのデプロイ**
cd ./002_nginx/nginx-chart
helm install my-nginx .

```
NAME: my-nginx
LAST DEPLOYED: Tue Jul 30 11:55:03 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
1. Get the application URL by running these commands:

以下を貼り付けて実行してください。
  export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/name=nginx-chart,app.kubernetes.io/instance=my-nginx" -o jsonpath="{.items[0].metadata.name}")
  export CONTAINER_PORT=$(kubectl get pod --namespace default $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
  echo "Visit http://127.0.0.1:8080 to use your application"
  kubectl --namespace default port-forward $POD_NAME 8080:$CONTAINER_PORT
```
・POD_NAME変数にNginxポッドの名前を取得して格納
・CONTAINER_PORT変数にNginxコンテナのポート番号を取得して格納
・アクセスURLの表示
・ローカルホストのポート8080をポッドのコンテナポートにフォワード


kubectl get pods
kubectl get services