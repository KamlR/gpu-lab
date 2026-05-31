## Порядок выполнения работы

### 1. Установка Kubernetes

Добавление репозитория Kubernetes и установка компонентов кластера.

```bash
sudo mkdir -p /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Проверка установленных версий.

```bash
kubeadm version
kubectl version --client
kubelet --version
```

### 2. Инициализация Kubernetes-кластера

Создание одновузлового кластера.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

Настройка kubectl для текущего пользователя.

```bash
mkdir -p "$HOME/.kube"
sudo cp -i /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
```

Разрешение запуска workload на control-plane узле.

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane- || true
```

### 3. Установка сетевого плагина Flannel

Установка CNI.

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Проверка состояния кластера.

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

### 4. Установка Helm

Установка менеджера пакетов Helm.

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

Проверка версии.

```bash
helm version
```

### 5. Установка NVIDIA GPU Operator

Создание namespace.

```bash
kubectl create namespace gpu-operator
kubectl label --overwrite namespace gpu-operator pod-security.kubernetes.io/enforce=privileged
```

Создание ConfigMap для GPU sharing.

```bash
kubectl apply -f device-plugin-sharing-config.yaml
```

Установка GPU Operator.

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

helm install gpu-operator nvidia/gpu-operator \
  -n gpu-operator \
  --version=v25.10.1 \
  --set driver.enabled=true \
  --set driver.version=580.105.08 \
  --set toolkit.enabled=true \
  --set dcgmExporter.enabled=true \
  --set devicePlugin.config.name=device-plugin-sharing-config \
  --set devicePlugin.config.default=any
```

Проверка состояния компонентов.

```bash
kubectl get pods -n gpu-operator
```

Проверка обнаружения GPU.

```bash
NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"

kubectl describe node "$NODE" | sed -n '/Capacity:/,/Allocatable:/p'
```

### 6. Создание рабочего namespace

Создание отдельного пространства имён для экспериментов.

```bash
kubectl apply -f namespace.yaml
kubectl get namespace gpu-lab
```

### 7. Проверка GPU через JupyterLab

Запуск JupyterLab.

```bash
kubectl apply -f jupyterlab.yaml
kubectl get pods -n gpu-lab -w
```

Проброс порта.

```bash
kubectl port-forward -n gpu-lab svc/jupyterlab 8888:8888
```

Проверка доступа к GPU.

```bash
kubectl exec -n gpu-lab deployment/jupyterlab -- nvidia-smi
```

Удаление JupyterLab после проверки.

```bash
kubectl delete deployment jupyterlab -n gpu-lab
```

### 8. Установка мониторинга

Установка kube-prometheus-stack.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring
```

Проверка компонентов мониторинга.

```bash
kubectl get pods -n monitoring
```

Настройка сбора GPU-метрик.

```bash
kubectl apply -f dcgm-servicemonitor.yaml
```

Получение пароля Grafana.

```bash
kubectl get secret -n monitoring monitoring-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d && echo
```

### 9. Эксперимент 1 — Exclusive GPU

Переключение GPU в режим exclusive.

```bash
NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"

kubectl label node "$NODE" nvidia.com/device-plugin.config- || true

kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
```

Запуск benchmark.

```bash
kubectl delete job -n gpu-lab --ignore-not-found torch-benchmark

kubectl apply -f torch-benchmark-exclusive.yaml

kubectl logs -n gpu-lab -f job/torch-benchmark
```

### 10. Эксперимент 2 — Time-Slicing

Включение профиля time-slicing.

```bash
NODE="$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')"

kubectl label node "$NODE" nvidia.com/device-plugin.config=t4-timeslicing-4 --overwrite

kubectl rollout restart -n gpu-operator daemonset/nvidia-device-plugin-daemonset
```

Проверка ресурсов GPU.

```bash
kubectl describe node "$NODE" | sed -n '/Capacity:/,/Allocatable:/p'
```

Запуск четырёх параллельных задач.

```bash
for i in 1 2 3 4; do
  kubectl delete job -n gpu-lab --ignore-not-found "torch-benchmark-ts-$i"

  sed "s/name: torch-benchmark/name: torch-benchmark-ts-$i/" \
    torch-benchmark-shared.yaml | kubectl apply -f -
done
```

Просмотр логов.

```bash
for i in 1 2 3 4; do
  echo "===== torch-benchmark-ts-$i ====="
  kubectl logs -n gpu-lab "job/torch-benchmark-ts-$i"
done
```
