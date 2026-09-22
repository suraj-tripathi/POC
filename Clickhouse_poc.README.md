# ClickHouse High Availability and Monitoring POC

## 1. Overview

This Proof of Concept (POC) demonstrates how to deploy a highly available ClickHouse cluster on Kubernetes and monitor the ClickHouse environment using an observability stack.

The POC uses:

- Kubernetes
- containerd
- Calico CNI
- ClickHouse
- ClickHouse Operator
- ClickHouse Keeper
- Persistent Storage
- VictoriaMetrics
- vmagent
- Grafana
- Kubernetes exporters

The primary objective is to understand how ClickHouse can be deployed as a highly available distributed database on Kubernetes and how its infrastructure and database metrics can be monitored.

---

# 2. POC Objectives

The main objectives of this POC are:

1. Deploy ClickHouse on Kubernetes.
2. Create a multi-node ClickHouse cluster.
3. Configure ClickHouse replication.
4. Deploy ClickHouse Keeper for cluster coordination.
5. Distribute ClickHouse replicas across different Kubernetes worker nodes.
6. Configure persistent storage for ClickHouse.
7. Test ClickHouse availability during pod failure.
8. Test ClickHouse recovery after failure.
9. Collect ClickHouse and Kubernetes metrics.
10. Store metrics in VictoriaMetrics.
11. Visualize metrics using Grafana.
12. Create a monitoring dashboard for ClickHouse and Kubernetes.

---

# 3. Scope

## Included

- ClickHouse High Availability
- ClickHouse replication
- ClickHouse Keeper
- Kubernetes deployment
- Persistent storage
- ClickHouse pod distribution
- Failure and recovery testing
- ClickHouse monitoring
- Kubernetes monitoring
- VictoriaMetrics
- vmagent
- Grafana
- Exporters

## Not Included

Kubernetes control-plane High Availability is outside the scope of this POC.

The Kubernetes cluster contains a single control-plane node. High availability is implemented at the ClickHouse application/database layer.

---

# 4. Kubernetes Architecture

The Kubernetes cluster consists of:

| Node | Role | Private IP | Availability Zone |
|------|------|------------|-------------------|
| k8s-master | Control Plane | 172.31.27.247 | us-east-1c |
| k8s-worker1 | Worker | 172.31.20.176 | us-east-1c |
| k8s-worker2 | Worker | 172.31.4.214 | us-east-1a |
| k8s-worker3 | Worker | 172.31.83.90 | us-east-1b |

The Kubernetes control plane is hosted on `k8s-master`.

The ClickHouse workloads will be distributed across the three worker nodes.

---

# 5. Kubernetes Runtime

Kubernetes uses containerd as the container runtime.

The container execution flow is:

```text
Kubernetes
     |
     v
   kubelet
     |
     v
    CRI
     |
     v
 containerd
     |
     v
    runc
     |
     v
 Containers
