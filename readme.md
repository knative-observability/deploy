## 部署

### 前置条件

* 装有 Knative Serving、Knative Eventing 的 Kubernetes 集群。
  * 该集群使用 Magic DNS 作为解析服务。
  * 该集群使用 In-Memory Channel 作为消息层通道服务。
  * 该集群使用 MT-Channel-based 作为 Broker 层。
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
# └── Grafana (Plugin HostPath: /data/plugins)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/kube-prometheus-stack -n observability -f values/kube-prometheus-stack.yaml
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
## ------------------ Run them in EVERY node -----------------┐
### Set HostPath permissions
sudo useradd -u 10001 loki
sudo mkdir -p /data/loki
sudo chown -R loki:loki /data/loki
## -----------------------------------------------------------┘
kubectl apply -f loki-pv.yaml # Create a persistent volume
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install --values values/loki.yaml loki --namespace=observability grafana/loki
## Promtail
helm upgrade --values values/promtail.yaml --install promtail --namespace=observability grafana/promtail
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
mage -v			# Build Backend
npm install		# Install Frontend Dependency
npm run build	# Build Frontend

sudo mkdir -p /data/plugins/lyz-knative-datasource
sudo cp -R dist /data/plugins/lyz-knative-datasource
sudo mv /data/plugins/lyz-knative-datasource/dist /data/plugins/lyz-knative-datasource/lyz-knative-datasource
chmod -R 777 /data/plugins/lyz-knative-datasource
kubectl delete pod -n observability -l app.kubernetes.io/name=grafana
```

