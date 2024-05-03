## 部署

### 前置条件

* 装有 Knative Serving、Knative Eventing 的 Kubernetes 集群。
  * 该集群使用 Magic DNS 作为解析服务。
  * 该集群使用 In-Memory Channel 作为消息层通道服务。
  * 该集群使用 MT-Channel-based 作为 Broker 层。
  * 该集群安装了 NFS CSI 动态制备卷接口实现，并且接入了 NFS 服务器。
* MetalLB，作为集群的 LoadBalancer 实现。
* cert-manager，作为 Jaeger 的依赖。
* Knative CLI 和 Knative Func CLI，作为函数部署的工具。
* Helm，作为 Kubernetes 集群的包管理器。
* Go，作为 Knative Agent 和 Knative Datasource 后端的编译器。
* Mage，Grafana Datasource 后端的构建工具。
* Node.js + NPM，作为 Knative Datasource 前端的构建工具。

### 部署流程（从源代码部署）

#### 克隆代码仓库

```bash
git clone https://github.com/knative-observability/lyz-knative-datasource.git
git clone https://github.com/knative-observability/knative-agent.git
git clone https://github.com/knative-observability/deploy.git
...（待补充）
```

在克隆完成后，请切换到 `deploy` 目录进行部署。

#### 部署 Knative 原生观测组件

该可观测性平台利用了 Knative 原生的观测能力，因此需要对这些观测组件进行部署。这些组件均采用了简单部署方式，旨在对功能进行验证和测试。

```bash
# Create new namespace: observability
kubectl create namespace observability

# Metric observability
# kube-prometheus-stack
# ├── Prometheus
# └── Grafana
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack -n observability -f values/kube-prometheus-stack.yaml
!!! chown to 472:472 and chmod to 777
# Create Prometheus ServiceMonitor CRs
kubectl apply -f servicemonitors.yaml

# Trace observability
## Jaeger
kubectl create -f https://github.com/jaegertracing/jaeger-operator/releases/download/v1.56.0/jaeger-operator.yaml -n observability
kubectl apply -f jaeger.yaml
## ConfigMaps
kubectl apply -f config-tracing.yaml

# Log observability
## Loki
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install --values values/loki.yaml loki --namespace=observability grafana/loki
!!! chown to 10001:10001
## Promtail
helm upgrade --values values/promtail.yaml --install promtail --namespace=observability grafana/promtail
```

#### 验证原生组件部署

```bash
# deploy bomb
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n observability 9090:9090 &
kubectl port-forward svc/jaeger-query -n observability 16686:16686 &
## Port forward to host
## ...
k port-forward -n observability svc/prometheus-grafana 3000:80
```



#### 部署 Knative Agent

```bash
# Change directory to knative-agent
cd ../knative-agent
# Set $REGISTRY variable in build.sh to your image registry
vim ./build.sh
# Build and push image to $REGISTRY
./build.sh
# Change back to "deploy" directory
cd ../deploy
# Set container "knative-agent" image to $REGISTRY/knative-agent:latest
vim ./agent.yaml
# Deploy knative-agent
kubectl apply -f agent.yaml
```

#### 部署 Knative Datasource

```bash
cd ../lyz-knative-datasource
npm install		# Install Frontend Dependency
mage -v			# Build Backend
npm run build	# Build Frontend

export PLUGIN_PATH=/storage/pvc-48ed67e0-8845-47c7-9d69-a15f99cb9e6e/plugins

sudo rm -rf ${PLUGIN_PATH}/lyz-knative-datasource
sudo mkdir -p ${PLUGIN_PATH}/lyz-knative-datasource
sudo cp -R dist ${PLUGIN_PATH}/lyz-knative-datasource
sudo mv ${PLUGIN_PATH}/lyz-knative-datasource/dist ${PLUGIN_PATH}/lyz-knative-datasource/lyz-knative-datasource
chmod -R 777 ${PLUGIN_PATH}/lyz-knative-datasource
chown -R 472:472 ${PLUGIN_PATH}
kubectl delete pod -n observability -l app.kubernetes.io/name=grafana
```

#### 配置 Grafana 面板

```bash
k port-forward -n observability svc/prometheus-grafana 3000:80
## port forward to host (ssh -L or VSCode)
## + Prometheus: knative-prometheus
##   http://prometheus-kube-prometheus-prometheus:9090
## + Loki: knative-loki
##   http://loki:3100
## + Jaeger: knative-jaeger
##   http://jaeger-query:16686
## + Knative Agent: knative-agent
##   http://knative-agent:9091

## new Dashboard folder: knative

```

