
## systemReserved

为操作系统守护进程（如 sshd、cron 等）预留资源（CPU、内存等），防止 Kubernetes 工作负载占用这些关键系统进程所需的资源。

```yaml
systemReserved:
  cpu: "500m"
  memory: "1Gi"
```

资源从 Kubernetes 调度器的可用资源中扣除（节点 Allocatable 会减少）。

## kubeReserved

为 Kubernetes 系统组件（如 kubelet、容器运行时、kube-proxy 等）预留资源，确保它们不会被工作负载挤占。

```yaml
kubeReserved:
  cpu: "1000m"
  memory: "2Gi"
```

资源从 Kubernetes 调度器的可用资源中扣除（节点 Allocatable 会减少）。

## reservedSystemCPUs

独占式预留指定的 CPU 核，仅供系统组件（非 Kubernetes Pod）使用。这些 CPU 不会被 Kubernetes 调度任何 Pod。

```yaml
reservedSystemCPUs: "0-1"  # 保留前两个物理核
```

与 systemReserved/kubeReserved 不同，这里是物理核隔离（通过 cpuset 实现），而非逻辑预留。

kubelet开启绑核需要指定这个字段.

优先级高于 systemReserved/kubeReserved, 如果这里指定"0-1"(即预留2个核), 不管"systemReserved.cpu + kubeReserved.cpu"的值是多少(大于2还是小于2), "Capacity.cpu - Allocatable.cpu"都等于2.
