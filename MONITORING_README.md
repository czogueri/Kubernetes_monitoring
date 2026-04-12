# 📊 Kubernetes Observability Stack — Prometheus + Grafana

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34.3-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/)
[![Grafana](https://img.shields.io/badge/Grafana-F46800?style=for-the-badge&logo=grafana&logoColor=white)](https://grafana.com/)
[![Helm](https://img.shields.io/badge/Helm-v3-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![AlertManager](https://img.shields.io/badge/Alertmanager-E6522C?style=for-the-badge&logo=prometheus&logoColor=white)](https://prometheus.io/docs/alerting/alertmanager/)
[![MetalLB](https://img.shields.io/badge/MetalLB-LoadBalancer-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://metallb.universe.tf/)

> A production-grade observability stack deployed on a self-managed 4-node bare-metal Kubernetes cluster — providing full visibility into cluster health, node resources, pod performance, and automated alerting.

---

## 📸 Live Project Screenshots

### Grafana — Kubernetes Compute Resources / Cluster
![Grafana Cluster Dashboard](screenshots/grafana-cluster.png)
*Live cluster-wide memory usage by namespace — monitoring stack consuming 834 MiB, ArgoCD at 310 MiB, Jenkins at 365 MiB. Network bandwidth tracked per namespace showing MetalLB handling 835 kb/s transmit throughput.*

---

### Grafana — Kubernetes Kubelet Dashboard
![Grafana Kubelet Dashboard](screenshots/grafana-kubelet.png)
*Real-time kubelet stats across all 4 nodes — 4 running kubelets, 42 running pods, 88 running containers, 103 volumes mounted with zero config errors. Operation rate and pod start duration tracked at the 99th percentile.*

---

### Grafana — Kubernetes Networking / Namespace (Pods)
![Grafana Networking Dashboard](screenshots/grafana-networking.png)
*Per-pod network monitoring for the ArgoCD namespace — live gauge showing 10.2 kb/s receive and 4.36 kb/s transmit bandwidth. Full breakdown of receive/transmit bandwidth per ArgoCD component (application-controller, repo-server, redis, dex-server).*

---

## 📐 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        4-Node K3s Cluster                           │
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐  │
│  │  ubuntu-vm  │  │    minty    │  │    pve2     │  │  node4   │  │
│  │Control Plane│  │   Worker    │  │   Worker    │  │  Worker  │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────┬─────┘  │
│         │                │                │               │        │
│  ┌──────▼──────────────────────────────────────────────────▼─────┐ │
│  │                    Node Exporters (x4)                        │ │
│  │         Scrape: CPU, Memory, Disk, Network per node           │ │
│  └──────────────────────────┬────────────────────────────────────┘ │
│                             │                                       │
│  ┌──────────────────────────▼────────────────────────────────────┐ │
│  │              kube-state-metrics                               │ │
│  │    Scrape: Pod status, Deployment health, Resource limits     │ │
│  └──────────────────────────┬────────────────────────────────────┘ │
│                             │                                       │
│  ┌──────────────────────────▼────────────────────────────────────┐ │
│  │                    Prometheus Server                          │ │
│  │         Collects, stores, and queries all metrics             │ │
│  │                  192.168.1.247:9090                           │ │
│  └──────────┬──────────────────────────┬──────────────────────── ┘ │
│             │                          │                            │
│  ┌──────────▼──────────┐   ┌──────────▼──────────────────────────┐ │
│  │    Alertmanager     │   │            Grafana                  │ │
│  │  Routes alerts to   │   │   Visualizes metrics & dashboards   │ │
│  │  email/Slack/etc    │   │          192.168.1.244:80           │ │
│  │  192.168.1.246:9093 │   └─────────────────────────────────────┘ │
│  └─────────────────────┘                                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Stack & Tools

| Component | Tool | Version | Purpose |
|---|---|---|---|
| **Metrics Collection** | Prometheus | Latest | Scrapes and stores time-series metrics |
| **Visualization** | Grafana | Latest | Dashboards and data visualization |
| **Alert Routing** | Alertmanager | Latest | Routes alerts to notification channels |
| **Node Metrics** | Node Exporter | Latest | Host-level CPU, memory, disk, network |
| **K8s Metrics** | kube-state-metrics | Latest | Kubernetes object health and status |
| **Operator** | kube-prometheus-operator | Latest | Manages Prometheus/Alertmanager config |
| **Package Manager** | Helm v3 | v3.x | Deployed entire stack via single chart |
| **Load Balancing** | MetalLB | v0.15.3 | Static external IPs for all services |
| **Cluster** | K3s | v1.34.3 | Lightweight Kubernetes on bare metal |

---

## 🖥️ Cluster Topology

| Node | Role | IP | Hosted Services |
|---|---|---|---|
| `ubuntu-vm` | Control Plane | 192.168.1.181 | Prometheus, Grafana, kube-state-metrics |
| `minty` | Worker | 192.168.1.210 | Alertmanager, Node Exporter |
| `pve2` | Worker | 192.168.1.206 | Node Exporter |
| `node4` | Worker | 192.168.1.x | Node Exporter |

### Network IP Allocation (MetalLB Pool: 192.168.1.240–250)

| Service | External IP | Port |
|---|---|---|
| Grafana | 192.168.1.244 | 80 |
| Prometheus | 192.168.1.247 | 9090 |
| Alertmanager | 192.168.1.246 | 9093 |

---

## 🚀 Deployment

### Prerequisites
- K3s cluster running
- Helm v3 installed
- MetalLB configured with IP pool

### Install the kube-prometheus-stack

```bash
# Add Helm repo
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace monitoring

# Install full observability stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.service.type=LoadBalancer \
  --set prometheus.service.type=LoadBalancer \
  --set alertmanager.service.type=LoadBalancer
```

### Assign Static IPs via MetalLB

```bash
# Grafana
kubectl annotate svc prometheus-grafana \
  -n monitoring \
  metallb.universe.tf/loadBalancerIPs=192.168.1.244 --overwrite

# Prometheus
kubectl annotate svc prometheus-kube-prometheus-prometheus \
  -n monitoring \
  metallb.universe.tf/loadBalancerIPs=192.168.1.247 --overwrite

# Alertmanager
kubectl annotate svc prometheus-kube-prometheus-alertmanager \
  -n monitoring \
  metallb.universe.tf/loadBalancerIPs=192.168.1.246 --overwrite
```

### Verify Deployment

```bash
kubectl get pods -n monitoring
kubectl get svc -n monitoring | grep LoadBalancer
```

---

## 📈 Key Metrics & PromQL Queries

These queries are used in the Grafana dashboards:

### Node CPU Usage
```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Node Memory Usage
```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Pod Restart Count
```promql
sum by (pod, namespace) (kube_pod_container_status_restarts_total)
```

### Running Pods Per Node
```promql
count by (node) (kube_pod_info{pod_phase="Running"})
```

### Disk Usage Per Node
```promql
100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)
```

### Pods Not Running
```promql
count by (namespace, pod) (kube_pod_status_phase{phase!="Running", phase!="Succeeded"})
```

---

## 📊 Grafana Dashboards

The kube-prometheus-stack deploys these pre-built dashboards automatically:

| Dashboard | What it Shows |
|---|---|
| **Kubernetes / Nodes** | CPU, memory, disk, network per node |
| **Kubernetes / Pods** | Pod health, restarts, resource usage |
| **Kubernetes / Compute Resources / Cluster** | Cluster-wide resource consumption |
| **Kubernetes / Workloads** | Deployment, StatefulSet, DaemonSet status |
| **Node Exporter / Full** | Detailed host-level system metrics |
| **Alertmanager** | Active alerts and alert history |

---

## 🔔 Alerting

Alertmanager is configured to handle alerts fired by Prometheus. Default alert rules included:

| Alert | Condition | Severity |
|---|---|---|
| `KubePodCrashLooping` | Pod restarting > 5 times in 15min | Critical |
| `KubePodNotReady` | Pod not ready for > 15min | Warning |
| `NodeMemoryPressure` | Node memory > 90% | Critical |
| `NodeDiskPressure` | Node disk > 85% | Warning |
| `KubeNodeNotReady` | Node not ready for > 15min | Critical |
| `TargetDown` | Scrape target unreachable | Critical |

---

## 🔧 Real-World Issues Solved

### 1. MetalLB IP Conflicts
**Problem:** Monitoring services were auto-assigned IPs that conflicted with existing services (Jenkins, ArgoCD, Nginx Ingress).

**Fix:** Annotated each service with explicit static IPs from the MetalLB pool:
```bash
kubectl annotate svc prometheus-grafana \
  -n monitoring \
  metallb.universe.tf/loadBalancerIPs=192.168.1.244 --overwrite
```

### 2. Failed LXC Container Node Addition
**Problem:** Attempted to add a Proxmox LXC container as a 4th node. K3s agent failed because kernel modules `overlay` and `br_netfilter` were missing from Proxmox kernel `6.17.2-1-pve`.

**Diagnosis:**
```bash
modprobe overlay
# FATAL: Module overlay not found in directory /lib/modules/6.17.2-1-pve
```

**Fix:** Used a full VM instead of LXC container — VMs have their own kernel with all required modules available.

---

## 📁 Repository Structure

```
monitoring/
├── screenshots/
│   ├── grafana-cluster.png        # Grafana compute resources cluster view
│   ├── grafana-kubelet.png        # Grafana kubelet dashboard
│   └── grafana-networking.png     # Grafana namespace networking view
└── README.md                      # This file
```

---

## 📚 Skills Demonstrated

- ✅ **Observability** — Full metrics pipeline from collection to visualization
- ✅ **Prometheus** — PromQL queries, scrape configs, alert rules
- ✅ **Grafana** — Dashboard creation, data source configuration, visualization
- ✅ **Helm** — Production deployment of complex multi-component stacks
- ✅ **Alerting** — Alertmanager configuration and alert rule management
- ✅ **Kubernetes Internals** — kube-state-metrics, node exporters, operators
- ✅ **Networking** — Static IP assignment, MetalLB conflict resolution
- ✅ **Troubleshooting** — Kernel module failures, IP conflicts, node joining issues
- ✅ **High Availability** — 4-node cluster with workload distribution

---

## 🔗 Related Projects

- [CI/CD Pipeline](https://github.com/czogueri/CI-CD-pipeline) — Jenkins + ArgoCD GitOps pipeline on the same cluster
- [Wazuh SIEM](https://github.com/czogueri/wazuh-kubernetes) — Security monitoring stack previously deployed on this cluster

---

## 👤 Author

**czogueri**
- GitHub: [@czogueri](https://github.com/czogueri)
- Email: czogueri@gmail.com

---

*Built on a self-managed bare-metal Kubernetes cluster — no managed cloud services.*
