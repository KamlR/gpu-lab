## Исследование DRA
Исследование направлено на то, чтобы изучить понятие DRA и как базово с ним работать в Kubernetes.

### Как работает без DRA
Без DRA GPU выдаётся через NVIDIA Device Plugin. Pod просто запрашивает ресурс:
```
resources:
  limits:
    nvidia.com/gpu: 1
```
Kubernetes видит GPU как числовой ресурс: 0, 1, 4 и т.д. Именно так работали  эксперименты Exclusive и Time-slicing.

### Зачем придумали DRA
DRA придумали, чтобы описывать устройства гибче: не просто “дай 1 GPU”, а “дай GPU нужного класса/характеристик”, использовать claims, делить устройства, учитывать атрибуты и переносить больше логики выбора устройства в драйвер. Kubernetes описывает DRA как механизм для запроса и совместного использования ресурсов.

### Зачем придумали DRA

DRA полезен для:
- GPU/TPU/FPGA workloads;
- ML/AI задач;
- кластеров с разными моделями GPU;
- сценариев, где нужно явно описывать требования к устройству;
- будущих сценариев с MIG, sharing, device attributes.

### Что нужно, чтобы это работало
- Kubernetes с поддержкой DRA;
- установленный DRA driver от вендора устройства;
- для NVIDIA — NVIDIA DRA Driver for GPUs;
- ResourceClass;
- ResourceClaim или ResourceClaimTemplate;
- Pod, который использует этот claim.

### Проверка на VM
В рамках бонусного задания была проверена поддержка Dynamic Resource Allocation в установленном Kubernetes-кластере.

Проверка доступных API-ресурсов показала наличие объектов группы resource.k8s.io/v1:

- deviceclasses
- resourceclaims
- resourceclaimtemplates
- resourceslices

Это означает, что установленная версия Kubernetes 1.34.8 поддерживает DRA API.

Также была выполнена проверка объекта ResourceClass:
```
kubectl get resourceclass
```
Команда завершилась ошибкой:
```
error: the server doesn't have a resource type "resourceclass"
```
Это связано с тем, что в используемой версии Kubernetes применяется актуальный объект DeviceClass, а не устаревший ResourceClass.

Дополнительно была проверена конфигурация NVIDIA GPU Operator версии 25.10.1:
```
helm show values nvidia/gpu-operator --version v25.10.1 | grep -i dra -A20 -B5
```

В выводе не были обнаружены параметры, связанные с DRA. Следовательно, в текущей конфигурации NVIDIA DRA Driver не был установлен.

Таким образом, кластер поддерживает DRA на уровне Kubernetes API, однако полноценное использование DRA для NVIDIA GPU требует установки отдельного NVIDIA DRA Driver или версии GPU Operator с поддержкой DRA. 