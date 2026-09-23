# ClickHouse HA + Monitoring POC on Kubernetes — Detailed Implementation Guide

## 0. Document Purpose

This document is the detailed implementation record for the ClickHouse High Availability and Monitoring POC built on Kubernetes.

The goal is not only to show the final architecture, but to explain **what was configured, why it was configured, what each important command does, what the important YAML fields mean, how the components communicate, how the HA test was performed, how monitoring was connected, and what troubleshooting was required**.

### Scope clarification

This POC demonstrates:

- ClickHouse HA using **1 shard + 3 replicas**
- ClickHouse Keeper as the coordination layer
- Persistent storage for ClickHouse and Keeper
- Replica distribution across three Kubernetes workers/AZs
- ClickHouse replication using `ReplicatedMergeTree`
- ClickHouse pod failure/recovery
- Monitoring using Node Exporter, kubelet cAdvisor, kube-state-metrics, ClickHouse metrics and Keeper metrics
- vmagent scraping and remote writing to an existing VictoriaMetrics server
- Grafana as the visualization layer

This POC **does not make the Kubernetes control plane highly available**. There is one Kubernetes control-plane node.

---

# 1. ClickHouse Basics Before the POC

## 1.1 What is ClickHouse?

ClickHouse is an open-source, column-oriented database management system designed primarily for analytical workloads.

A simple way to remember it:

> ClickHouse is optimized for asking questions about a very large amount of data quickly.

Example:

```sql
SELECT
    service,
    count(*) AS requests
FROM logs
GROUP BY service;
```

This is an analytical query because it can read and aggregate a large number of records.

## 1.2 OLTP vs OLAP

### OLTP

OLTP means Online Transaction Processing.

Examples:

- Create an order
- Update a user's email
- Process a payment
- Update an employee record

A typical OLTP query may modify one or a small number of rows:

```sql
UPDATE users
SET email = 'new@email.com'
WHERE user_id = 101;
```

### OLAP

OLAP means Online Analytical Processing.

Examples:

- How many requests happened in the last 24 hours?
- What is the average latency per service?
- Which service generated the most errors?
- How many events happened in each hour?

Example:

```sql
SELECT
    service,
    avg(latency)
FROM logs
WHERE timestamp >= now() - INTERVAL 1 DAY
GROUP BY service;
```

ClickHouse is primarily designed for OLAP-style workloads.

---

# 2. Why Column-Oriented Storage Matters

A row-oriented database conceptually stores complete records together:

```text
Row 1: id, name, service, latency, timestamp
Row 2: id, name, service, latency, timestamp
Row 3: id, name, service, latency, timestamp
```

A column-oriented system organizes values by columns:

```text
id column
name column
service column
latency column
timestamp column
```

If an analytical query needs only `service` and `latency`, a column-oriented design can avoid reading unrelated columns.

This is one reason ClickHouse is effective for large analytical datasets.

---

# 3. ClickHouse Architecture Concepts Used in This POC

## 3.1 Node

A node is one ClickHouse server instance.

In this POC:

```text
CH-0
CH-1
CH-2
```

are three ClickHouse nodes.

## 3.2 Cluster

A cluster is a group of ClickHouse nodes that work together.

Our cluster:

```text
ClickHouse Cluster
 |
 +-- CH-0
 +-- CH-1
 +-- CH-2
```

## 3.3 Shard

A shard represents one distributed portion of a dataset.

If there are two shards:

```text
Shard 1 -> part of data
Shard 2 -> another part of data
```

Our POC uses:

```text
1 shard
```

Therefore the data is not horizontally split across multiple shards.

## 3.4 Replica

A replica is another copy of the data for the same shard.

Our POC:

```text
Shard 1
 |
 +-- Replica 1
 +-- Replica 2
 +-- Replica 3
```

Therefore:

```text
1 shard × 3 replicas = 3 ClickHouse pods
```

## 3.5 Sharding vs Replication

These are different concepts.

```text
Sharding:
split data

Replication:
copy data
```

Example:

```text
1 TB data

Sharding:
Shard 1 = 500 GB
Shard 2 = 500 GB

Replication:
Shard 1 Replica A = 500 GB
Shard 1 Replica B = 500 GB
```

This POC focuses on replication/HA rather than multi-shard scaling.

---

# 4. ReplicatedMergeTree

`ReplicatedMergeTree` is the ClickHouse table-engine family used for replicated data.

In this POC the table was created as:

```sql
CREATE TABLE ha_test.events ON CLUSTER default
(
    event_id UInt64,
    event_name String,
    event_time DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/ha_test/events',
    '{replica}'
)
ORDER BY event_id;
```

Important concepts:

### ZooKeeper/Keeper path

```text
/clickhouse/tables/{shard}/ha_test/events
```

This identifies the coordination path for the replicated table.

### Replica identifier

```text
{replica}
```

The operator-generated replica identity distinguishes the ClickHouse replicas.

### ORDER BY

```text
ORDER BY event_id
```

defines the sorting key used by the MergeTree family.

---

# 5. ClickHouse Keeper

ClickHouse Keeper is the coordination service used by ClickHouse distributed/replicated deployments.

It is based on the Raft consensus algorithm.

The important distinction is:

```text
ClickHouse
    |
    +-- stores table data
    |
Keeper
    |
    +-- coordinates distributed/replicated state
```

Keeper is **not the table-data storage layer**.

In this POC:

```text
Keeper-0
Keeper-1
Keeper-2
```

form a 3-node Keeper cluster.

Three nodes are used so the coordination layer can tolerate a node failure while retaining quorum.

---

# 6. Final POC Architecture

```text
                         AWS
                          |
                Kubernetes Cluster
                          |
              +-----------+-----------+
              |                       |
        k8s-master               3 Workers
       Control Plane                 |
              |          +-----------+-----------+
              |          |           |           |
              |       worker1     worker2     worker3
              |          |           |           |
              |          |           |           |
              |       CH-1        CH-0        CH-2
              |       Keeper-1    Keeper-0    Keeper-2
              |                       |
              |                 vmagent on worker3
              |
              +---------------- Kubernetes API

Monitoring:

Node Exporter ───────┐
cAdvisor ────────────┤
KSM ─────────────────┤
ClickHouse :9363 ────┤
Keeper :9090 ────────┤
                     ▼
                   vmagent
                     |
              remote_write
                     |
                     ▼
              VictoriaMetrics
                     |
                     ▼
                   Grafana
```

---

# 7. AWS Infrastructure

| Node | vCPU | RAM | Disk | Private IP | AZ | Role |
|---|---:|---:|---:|---|---|---|
| k8s-master | 2 | 4 GB | 30 GB | 172.31.27.247 | us-east-1c | Kubernetes control plane |
| k8s-worker1 | 2 | 4 GB | 30 GB | 172.31.20.176 | us-east-1c | ClickHouse + Keeper |
| k8s-worker2 | 2 | 4 GB | 30 GB | 172.31.4.214 | us-east-1a | ClickHouse + Keeper |
| k8s-worker3 | 2 | 4 GB | 30 GB | 172.31.83.90 | us-east-1b | ClickHouse + Keeper + vmagent |

The worker placement intentionally uses three AZs:

```text
worker1 → us-east-1c
worker2 → us-east-1a
worker3 → us-east-1b
```

This gives the POC a topology that demonstrates replica distribution across failure domains.

---

# 8. Hostname Configuration

The following mapping was configured on all nodes:

```text
127.0.0.1 localhost

172.31.27.247  k8s-master
172.31.20.176  k8s-worker1
172.31.4.214   k8s-worker2
172.31.83.90   k8s-worker3
```

Why?

Stable hostname resolution makes node-to-node administration and troubleshooting easier.

Verify:

```bash
hostname
getent hosts k8s-worker1
getent hosts k8s-worker2
getent hosts k8s-worker3
```

---

# 9. Kubernetes Prerequisites

## 9.1 Disable Swap

Kubernetes kubelet expects swap to be disabled for this setup.

Run on every node:

```bash
sudo swapoff -a
```

Make it persistent:

```bash
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify:

```bash
free -h
swapon --show
```

`swapon --show` should return no active swap device.

---

# 10. Kernel Modules

Create:

```bash
sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF
```

Load immediately:

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

Verify:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

### Why these modules?

`overlay` is used by container storage.

`br_netfilter` allows Linux bridge traffic to be processed by netfilter/iptables, which is important for Kubernetes networking.

---

# 11. Kubernetes Network Sysctl

Create:

```bash
sudo tee /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

Apply:

```bash
sudo sysctl --system
```

Verify:

```bash
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.ipv4.ip_forward = 1
```

---

# 12. Container Runtime: containerd

Docker was installed on the machines, but Kubernetes was configured to use **containerd directly**.

The runtime path is:

```text
Kubernetes
    |
  kubelet
    |
   CRI
    |
containerd
    |
   runc
    |
containers
```

`cri-dockerd` was therefore not required.

Generate containerd configuration:

```bash
sudo cp /etc/containerd/config.toml /etc/containerd/config.toml.bak

containerd config default | \
sudo tee /etc/containerd/config.toml > /dev/null
```

Enable systemd cgroups:

```bash
sudo sed -i \
's/SystemdCgroup = false/SystemdCgroup = true/' \
/etc/containerd/config.toml
```

Restart:

```bash
sudo systemctl restart containerd
sudo systemctl enable containerd
```

Verify:

```bash
sudo systemctl status containerd
```

CRI verification:

```bash
crictl \
--runtime-endpoint=unix:///var/run/containerd/containerd.sock \
info
```

Expected:

```text
RuntimeReady: true
```

---

# 13. Kubernetes Installation

Kubernetes version used:

```text
v1.29.15
```

Install prerequisites:

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

Add repository key:

```bash
curl -fsSL \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | \
sudo gpg --dearmor \
-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add repository:

```bash
echo \
'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | \
sudo tee /etc/apt/sources.list.d/kubernetes.list
```

Install:

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

Hold versions:

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

---

# 14. Initialize the Kubernetes Control Plane

This is a single control-plane POC.

Command:

```bash
sudo kubeadm init \
  --apiserver-advertise-address=172.31.27.247 \
  --pod-network-cidr=192.168.0.0/16 \
  --cri-socket=unix:///var/run/containerd/containerd.sock
```

Important options:

### `--apiserver-advertise-address`

Tells kubeadm which node IP the API server should advertise.

### `--pod-network-cidr`

Defines the pod CIDR used by the CNI.

### `--cri-socket`

Explicitly tells kubeadm to use containerd.

---

# 15. Configure kubectl

```bash
mkdir -p $HOME/.kube

sudo cp -i /etc/kubernetes/admin.conf \
$HOME/.kube/config

sudo chown \
$(id -u):$(id -g) \
$HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

---

# 16. Join Worker Nodes

Use the `kubeadm join` command generated by `kubeadm init`.

For this environment the important runtime argument is:

```text
--cri-socket=unix:///var/run/containerd/containerd.sock
```

After joining:

```bash
kubectl get nodes -o wide
```

Final expected state:

```text
k8s-master    Ready
k8s-worker1   Ready
k8s-worker2   Ready
k8s-worker3   Ready
```

---

# 17. Calico Networking

Calico was used as the Kubernetes CNI.

The final working network mode was:

```text
backend: bird
IP-in-IP: enabled
VXLAN: disabled
```

Conceptually:

```text
Pod on worker1
    |
    | IP-in-IP
    v
worker2
    |
    v
Pod on worker2
```

The final configuration included:

```text
calico_backend: bird
ipipMode: Always
vxlanMode: Never
```

The Calico nodes reached `1/1 Running`, BGP peers were established, and cross-worker pod communication was verified.

---

# 18. Why We Did Not Change Networking After Validation

Once:

- cross-worker pod ping worked
- DNS worked
- BGP was established
- ClickHouse and Keeper communicated
- monitoring traffic worked

the networking layer was treated as validated.

Later cAdvisor problems were therefore investigated at TCP/SG level rather than changing Calico unnecessarily.

This is an important troubleshooting principle:

> Once a lower layer is proven healthy, investigate the failing layer instead of changing working components blindly.

---

# 19. Namespaces

Final namespace manifest:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: clickhouse
  labels:
    purpose: clickhouse
---
apiVersion: v1
kind: Namespace
metadata:
  name: observability
  labels:
    purpose: observability
```

Apply:

```bash
kubectl apply -f namespaces.yml
```

Verify:

```bash
kubectl get namespaces
```

---

# 20. ClickHouse Operator

The ClickHouse Operator manages ClickHouse custom resources.

Install:

```bash
kubectl apply --server-side --force-conflicts \
-f https://github.com/ClickHouse/clickhouse-operator/releases/latest/download/clickhouse-operator.yaml
```

Verify:

```bash
kubectl get pods -n clickhouse-operator-system
```

Verify CRDs:

```bash
kubectl get crd | grep clickhouse
```

Expected:

```text
clickhouseclusters.clickhouse.com
keeperclusters.clickhouse.com
```

The API used in the POC is:

```text
clickhouse.com/v1alpha1
```

---

# 21. cert-manager Compatibility Note

During installation, the latest cert-manager version had a CRD compatibility problem with Kubernetes 1.29 involving:

```text
spec.versions[0].selectableFields
```

For this POC, the incompatible version was removed and cert-manager `v1.18.3` was used as a compatibility workaround.

This is recorded as a POC-specific compatibility decision, not as a general recommendation for new production deployments.

---

# 22. Storage Architecture

The POC used local Kubernetes PersistentVolumes.

StorageClass:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: clickhouse-local
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

Apply:

```bash
kubectl apply -f storageclass.yml
```

### Why `WaitForFirstConsumer`?

With local storage, the volume is physically tied to a node.

`WaitForFirstConsumer` allows Kubernetes scheduling to consider the pod's node placement before binding the local PV.

This is important because:

```text
CH-0 on worker2
    ↓
must use worker2 storage

CH-1 on worker1
    ↓
must use worker1 storage
```

---

# 23. Local Storage Directories

On each worker:

```bash
sudo mkdir -p /data/clickhouse
sudo chmod 777 /data/clickhouse

sudo mkdir -p /data/keeper
sudo chmod 777 /data/keeper
```

The POC used:

```text
/data/clickhouse → ClickHouse data
/data/keeper     → Keeper data
```

The final PV state was:

```text
clickhouse-pv-worker1 → 8Gi → worker1
clickhouse-pv-worker2 → 8Gi → worker2
clickhouse-pv-worker3 → 8Gi → worker3

keeper-pv-worker1 → 4Gi → worker1
keeper-pv-worker2 → 4Gi → worker2
keeper-pv-worker3 → 4Gi → worker3
```

All were `RWO` with `Retain` reclaim policy.

---

# 24. Keeper Cluster YAML

File:

```text
~/clickhouse-poc/keeper-cluster.yml
```

Final content:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: KeeperCluster
metadata:
  name: clickhouse-keeper
  namespace: clickhouse
spec:
  replicas: 3
  dataVolumeClaimSpec:
    storageClassName: clickhouse-local
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 4Gi
  podTemplate:
    topologyZoneKey: topology.kubernetes.io/zone
```

Apply:

```bash
kubectl apply -f ~/clickhouse-poc/keeper-cluster.yml
```

Verify:

```bash
kubectl get keepercluster -n clickhouse
```

Expected:

```text
NAME                READY   STATUS             READYREPLICAS   REPLICAS
clickhouse-keeper   True    Cluster is ready   3               3
```

---

# 25. Keeper Topology

Verify:

```bash
kubectl get pods -n clickhouse -o wide | grep keeper
```

Final observed topology:

```text
clickhouse-keeper-keeper-0-0 → worker2
clickhouse-keeper-keeper-1-0 → worker1
clickhouse-keeper-keeper-2-0 → worker3
```

This gives:

```text
Keeper-0 → us-east-1a
Keeper-1 → us-east-1c
Keeper-2 → us-east-1b
```

The three nodes are therefore distributed across the three workers/AZs.

---

# 26. Keeper Communication Ports

Keeper service:

```text
clickhouse-keeper-keeper-headless
```

Exposes:

```text
9234/TCP
9090/TCP
2181/TCP
```

The relevant roles are:

```text
9234 → Keeper/Raft communication
9090 → Prometheus metrics
2181 → Keeper client/ZooKeeper-compatible port
```

The exact metrics configuration was verified inside Keeper rather than assumed.

---

# 27. Keeper Prometheus Configuration

Inside the Keeper pod:

```bash
kubectl exec \
-n clickhouse \
clickhouse-keeper-keeper-0-0 -- \
sh -c \
'grep -R -n "<prometheus>\|prometheus:" \
/etc/clickhouse-keeper \
/etc/clickhouse-server 2>/dev/null | head -30'
```

The configuration file was:

```text
/etc/clickhouse-keeper/config.d/00-config.yaml
```

Verified configuration:

```yaml
prometheus:
  endpoint: /metrics
  port: 9090
  metrics: true
  events: true
  asynchronous_metrics: true
```

Therefore:

```text
Keeper metrics endpoint:
http://<keeper>:9090/metrics
```

---

# 28. Verify Keeper Metrics

Run:

```bash
kubectl run keeper-metrics-test \
  -n observability \
  --rm -it \
  --restart=Never \
  --image=curlimages/curl \
  -- \
  curl -s \
  http://clickhouse-keeper-keeper-headless.clickhouse.svc.cluster.local:9090/metrics \
  | head -20
```

Expected metrics include:

```text
# HELP ClickHouse_Info
# TYPE ClickHouse_Info gauge
ClickHouse_Info{...} 1

# HELP ClickHouseProfileEvents_FileOpen
# TYPE ClickHouseProfileEvents_FileOpen counter
```

This proves the Kubernetes service can reach the Keeper metrics endpoint.

---

# 29. ClickHouse Cluster YAML

File:

```text
~/clickhouse-poc/clickhouse-cluster.yml
```

Final content:

```yaml
apiVersion: clickhouse.com/v1alpha1
kind: ClickHouseCluster

metadata:
  name: clickhouse
  namespace: clickhouse

spec:
  replicas: 3
  shards: 1

  keeperClusterRef:
    name: clickhouse-keeper

  dataVolumeClaimSpec:
    storageClassName: clickhouse-local
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 8Gi

  podTemplate:
    topologyZoneKey: topology.kubernetes.io/zone
    nodeHostnameKey: kubernetes.io/hostname

  containerTemplate:
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
      limits:
        cpu: "1"
        memory: "2Gi"
```

Apply:

```bash
kubectl apply -f ~/clickhouse-poc/clickhouse-cluster.yml
```

---

# 30. Understanding the ClickHouse YAML

## `replicas: 3`

Creates three ClickHouse replicas.

```text
CH-0
CH-1
CH-2
```

## `shards: 1`

Creates one shard.

Therefore the topology is:

```text
1 shard
3 replicas
```

## `keeperClusterRef`

```yaml
keeperClusterRef:
  name: clickhouse-keeper
```

Connects the ClickHouse cluster to the Keeper cluster.

## `storageClassName`

```yaml
storageClassName: clickhouse-local
```

Requests the local storage class created for the POC.

## `ReadWriteOnce`

```text
RWO
```

The local volume is mounted read/write by the pod using that volume.

## `topologyZoneKey`

```text
topology.kubernetes.io/zone
```

Used by the operator's topology handling to distribute replicas across zones.

## `nodeHostnameKey`

```text
kubernetes.io/hostname
```

Helps enforce node-level separation.

## Resource requests

```text
CPU: 500m
Memory: 1Gi
```

These are the minimum requested resources for scheduling.

## Resource limits

```text
CPU: 1
Memory: 2Gi
```

These define the upper limits for the container.

---

# 31. Verify ClickHouse Cluster

```bash
kubectl get clickhousecluster -n clickhouse
```

Expected:

```text
NAME         READY   STATUS                 READYREPLICAS   REPLICAS
clickhouse   True    All shards are ready   3               3
```

Pods:

```bash
kubectl get pods -n clickhouse -o wide
```

Final observed topology:

```text
CH-0 → worker2
CH-1 → worker1
CH-2 → worker3
```

PVCs:

```bash
kubectl get pvc -n clickhouse
```

Expected:

```text
CH-0 → clickhouse-pv-worker2 → 8Gi
CH-1 → clickhouse-pv-worker1 → 8Gi
CH-2 → clickhouse-pv-worker3 → 8Gi

Keeper-0 → keeper-pv-worker2 → 4Gi
Keeper-1 → keeper-pv-worker1 → 4Gi
Keeper-2 → keeper-pv-worker3 → 4Gi
```

---

# 32. Understand the Headless Service

The ClickHouse headless service was:

```text
clickhouse-clickhouse-headless
```

Ports included:

```text
8123
9009
9001
9002
9363
9000
```

The important monitoring port is:

```text
9363
```

Keeper headless service:

```text
clickhouse-keeper-keeper-headless
```

Monitoring port:

```text
9090
```

Headless services allow Kubernetes DNS to resolve individual pod endpoints.

---

# 33. Verify ClickHouse Cluster Topology from SQL

Run:

```sql
SELECT
    cluster,
    shard_num,
    replica_num,
    host_name
FROM system.clusters
WHERE cluster = 'default';
```

Expected conceptual result:

```text
default  1  1  clickhouse-clickhouse-0-0-0
default  1  2  clickhouse-clickhouse-0-1-0
default  1  3  clickhouse-clickhouse-0-2-0
```

This confirms that ClickHouse knows about all three replicas.

---

# 34. Create Replicated Database

```sql
CREATE DATABASE IF NOT EXISTS ha_test
ON CLUSTER default;
```

`ON CLUSTER default` means the DDL is executed across the configured cluster.

---

# 35. Create Replicated Table

```sql
CREATE TABLE ha_test.events ON CLUSTER default
(
    event_id UInt64,
    event_name String,
    event_time DateTime
)
ENGINE = ReplicatedMergeTree(
    '/clickhouse/tables/{shard}/ha_test/events',
    '{replica}'
)
ORDER BY event_id;
```

Important:

```text
ReplicatedMergeTree
       |
       +-- replication mechanism
       |
       +-- coordinated through Keeper
```

---

# 36. Insert Test Data

On CH-0:

```sql
INSERT INTO ha_test.events
(event_id, event_name, event_time)
VALUES
(1, 'user_login', now()),
(2, 'order_created', now()),
(3, 'payment_completed', now());
```

Three rows were inserted.

---

# 37. Verify Replication

Query from the replicas:

```sql
SELECT *
FROM ha_test.events
ORDER BY event_id;
```

The three replicas returned:

```text
1  user_login
2  order_created
3  payment_completed
```

Replication status included:

```text
total_replicas = 3
active_replicas = 3
queue_size = 0
inserts_in_queue = 0
absolute_delay = 0
```

Interpretation:

```text
total_replicas = 3
```

There are three configured replicas.

```text
active_replicas = 3
```

All three were active.

```text
queue_size = 0
```

No pending replication work at verification time.

```text
absolute_delay = 0
```

No observed replication delay at verification time.

---

# 38. ClickHouse HA Failure Test

Baseline:

```text
CH-0 → Running
CH-1 → Running
CH-2 → Running
```

Delete one pod:

```bash
kubectl delete pod \
clickhouse-clickhouse-0-1-0 \
-n clickhouse
```

What happens?

The ClickHouse Operator observes that the expected pod is missing and recreates it.

Verify:

```bash
kubectl get pods -n clickhouse -o wide
```

Verify cluster:

```bash
kubectl get clickhousecluster -n clickhouse
```

The cluster returned to:

```text
All shards are ready
3/3 replicas ready
```

The surviving ClickHouse replica retained the test data.

### What this test proves

It proves the Kubernetes/operator layer automatically recreates the deleted ClickHouse pod and the ClickHouse cluster can return to its expected three-replica state.

It is a **pod-level failure/recovery test**.

It should not be described as proof of full Kubernetes control-plane HA.

---

# 39. ClickHouse Metrics

ClickHouse exposes Prometheus metrics.

The relevant configuration was:

```yaml
prometheus:
  type: prometheus
  description: prometheus
```

Metrics endpoint:

```text
http://<clickhouse-pod>:9363/metrics
```

The endpoint returned metrics such as:

```text
ClickHouse_Info
ClickHouseProfileEvents_*
```

All three ClickHouse replica endpoints were verified.

---

# 40. Monitoring Architecture

The monitoring pipeline is:

```text
Metric Sources
      |
      +-- Node Exporter
      +-- cAdvisor
      +-- kube-state-metrics
      +-- ClickHouse
      +-- ClickHouse Keeper
      |
      v
    vmagent
      |
      | remote_write
      v
VictoriaMetrics
      |
      v
    Grafana
```

Each source answers a different question.

| Component | What it tells us |
|---|---|
| Node Exporter | Linux/EC2 host health |
| cAdvisor | Container resource usage |
| KSM | Kubernetes object state |
| ClickHouse metrics | Database/server activity |
| Keeper metrics | Coordination service activity |
| vmagent | Scraping and forwarding |
| VictoriaMetrics | Metric storage/query |
| Grafana | Visualization |

---

# 41. Node Exporter

Node Exporter collects Linux host-level metrics.

Examples:

```text
CPU
Memory
Disk
Filesystem
Network
Load
```

It was deployed as a DaemonSet so that each worker gets one Node Exporter pod.

Image:

```text
quay.io/prometheus/node-exporter:v1.10.2
```

Important settings:

```yaml
hostNetwork: true
hostPID: true
```

Arguments:

```text
--path.procfs=/host/proc
--path.sysfs=/host/sys
--path.rootfs=/host/root
```

Port:

```text
9100
```

Host paths:

```text
/proc
/sys
/
```

The three worker Node Exporter endpoints were verified successfully.

---

# 42. Why DaemonSet for Node Exporter?

A DaemonSet ensures that Kubernetes schedules one pod on each eligible node.

Conceptually:

```text
worker1 → node-exporter
worker2 → node-exporter
worker3 → node-exporter
```

This is appropriate because we want host metrics from every worker.

---

# 43. cAdvisor

cAdvisor provides container-level resource metrics.

In this POC, a separate cAdvisor deployment was **not** used.

Kubernetes kubelet exposes cAdvisor metrics at:

```text
https://<node-ip>:10250/metrics/cadvisor
```

Example verification:

```bash
kubectl get --raw \
"/api/v1/nodes/k8s-worker1/proxy/metrics/cadvisor" \
| head -20
```

Metrics included:

```text
cadvisor_version_info
container_blkio_device_usage_total
```

This proves kubelet is exposing container metrics.

---

# 44. Why vmagent Uses HTTPS for cAdvisor

The kubelet endpoint uses HTTPS on port 10250.

Therefore vmagent configuration contains:

```yaml
scheme: https
metrics_path: /metrics/cadvisor
```

and the Kubernetes service-account token:

```yaml
bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
```

TLS configuration:

```yaml
tls_config:
  ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  insecure_skip_verify: true
```

The token allows authenticated access to the kubelet metrics endpoint.

---

# 45. cAdvisor Troubleshooting

Initially:

```text
worker1 cAdvisor → DOWN
worker2 cAdvisor → DOWN
worker3 cAdvisor → UP
master cAdvisor  → UP
```

The vmagent target error was:

```text
context deadline exceeded
```

We tested directly from worker3 because vmagent was running on worker3.

Commands:

```bash
nc -zv -w 5 172.31.20.176 10250
nc -zv -w 5 172.31.4.214 10250
nc -zv -w 5 172.31.20.176 9100
nc -zv -w 5 172.31.4.214 9100
```

Observed:

```text
worker3 → worker1:10250 → timeout
worker3 → worker2:10250 → timeout

worker3 → worker1:9100 → success
worker3 → worker2:9100 → success
```

This was a very useful isolation test.

It proved:

```text
Worker-to-worker networking = working
Node Exporter = reachable
Specific kubelet port 10250 = blocked
```

The issue was therefore not treated as a general Calico problem.

---

# 46. cAdvisor Root Cause and Fix

Root cause:

```text
Worker Security Group
did not allow
TCP 10250
from Worker Security Group
```

Final rule:

```text
Inbound
Protocol: TCP
Port: 10250
Source: Worker Security Group
```

No public `0.0.0.0/0` rule was required.

After the rule was added:

```text
worker1 cAdvisor → UP
worker2 cAdvisor → UP
worker3 cAdvisor → UP
master cAdvisor  → UP
```

This is an important troubleshooting result for the POC.

---

# 47. kube-state-metrics

kube-state-metrics exposes Kubernetes object/state metrics.

Examples:

```text
Pod state
Deployment state
DaemonSet state
Node state
Namespace state
Service state
PVC state
```

The final deployment used:

```text
ServiceAccount
ClusterRole
ClusterRoleBinding
Deployment
Service
```

Image:

```text
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0
```

Service:

```text
kube-state-metrics.observability.svc.cluster.local:8080
```

---

# 48. Final kube-state-metrics YAML

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-state-metrics
  namespace: observability

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-state-metrics
rules:
  - apiGroups:
      - ""
    resources:
      - pods
      - nodes
      - namespaces
      - services
      - configmaps
      - secrets
      - resourcequotas
      - replicationcontrollers
      - persistentvolumes
      - persistentvolumeclaims
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - apps
    resources:
      - deployments
      - daemonsets
      - replicasets
      - statefulsets
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - batch
    resources:
      - jobs
      - cronjobs
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-state-metrics
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-state-metrics
subjects:
  - kind: ServiceAccount
    name: kube-state-metrics
    namespace: observability

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kube-state-metrics
  namespace: observability
  labels:
    app: kube-state-metrics

spec:
  replicas: 1

  selector:
    matchLabels:
      app: kube-state-metrics

  template:
    metadata:
      labels:
        app: kube-state-metrics

    spec:
      serviceAccountName: kube-state-metrics

      containers:
        - name: kube-state-metrics
          image: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.17.0

          ports:
            - name: metrics
              containerPort: 8080

---
apiVersion: v1
kind: Service
metadata:
  name: kube-state-metrics
  namespace: observability
  labels:
    app: kube-state-metrics

spec:
  selector:
    app: kube-state-metrics

  ports:
    - name: metrics
      port: 8080
      targetPort: 8080
```

---

# 49. Why RBAC is Required for KSM

KSM reads Kubernetes API objects.

For example, to expose:

```text
kube_pod_info
kube_node_info
kube_deployment_status_*
```

it needs permission to:

```text
get
list
watch
```

the relevant resources.

The ClusterRole provides these permissions.

The ClusterRoleBinding attaches them to the KSM service account.

---

# 50. vmagent

vmagent is the metric scraper and forwarding component.

It performs two important functions:

```text
1. Scrape Prometheus-compatible endpoints
2. Remote-write samples to VictoriaMetrics
```

Architecture:

```text
Sources
   |
   v
vmagent
   |
   | remote_write
   v
VictoriaMetrics
```

vmagent was deployed in the `observability` namespace.

It runs on worker3 in the final setup.

---

# 51. vmagent RBAC

Final RBAC:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: vmagent
  namespace: observability

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: vmagent
rules:
  - apiGroups:
      - ""
    resources:
      - nodes
      - nodes/proxy
      - nodes/metrics
      - services
      - endpoints
      - pods
    verbs:
      - get
      - list
      - watch

  - apiGroups:
      - discovery.k8s.io
    resources:
      - endpointslices
    verbs:
      - get
      - list
      - watch

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vmagent
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: vmagent
subjects:
  - kind: ServiceAccount
    name: vmagent
    namespace: observability
```

Verify:

```bash
kubectl auth can-i get nodes/metrics \
--as=system:serviceaccount:observability:vmagent
```

Expected:

```text
yes
```

---

# 52. Final vmagent ConfigMap

File:

```text
~/clickhouse-poc/monitoring/vmagent-config.yml
```

Final configuration:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vmagent-config
  namespace: observability

data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      scrape_timeout: 10s

    scrape_configs:

      # =========================
      # Node Exporter
      # =========================
      - job_name: "node-exporter"

        static_configs:
          - targets:
              - "172.31.20.176:9100"
              - "172.31.4.214:9100"
              - "172.31.83.90:9100"

      # =========================
      # kube-state-metrics
      # =========================
      - job_name: "kube-state-metrics"

        static_configs:
          - targets:
              - "kube-state-metrics.observability.svc.cluster.local:8080"

      # =========================
      # ClickHouse
      # =========================
      - job_name: "clickhouse"

        static_configs:
          - targets:
              - "clickhouse-clickhouse-0-0-0.clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9363"
              - "clickhouse-clickhouse-0-1-0.clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9363"
              - "clickhouse-clickhouse-0-2-0.clickhouse-clickhouse-headless.clickhouse.svc.cluster.local:9363"

      # =========================
      # Kubernetes cAdvisor
      # =========================
      - job_name: "cadvisor"

        scheme: https

        metrics_path: /metrics/cadvisor

        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
          insecure_skip_verify: true

        kubernetes_sd_configs:
          - role: node

        relabel_configs:
          - source_labels:
              - __meta_kubernetes_node_name
            target_label: node

      # =========================
      # ClickHouse Keeper
      # =========================
      - job_name: "clickhouse-keeper"

        static_configs:
          - targets:
              - "clickhouse-keeper-keeper-0-0.clickhouse-keeper-keeper-headless.clickhouse.svc.cluster.local:9090"
              - "clickhouse-keeper-keeper-1-0.clickhouse-keeper-keeper-headless.clickhouse.svc.cluster.local:9090"
              - "clickhouse-keeper-keeper-2-0.clickhouse-keeper-keeper-headless.clickhouse.svc.cluster.local:9090"
```

---

# 53. Explain Each vmagent Job

## Node Exporter

```text
job_name: node-exporter
```

Three static worker IPs are scraped on:

```text
9100
```

## KSM

```text
kube-state-metrics.observability.svc.cluster.local:8080
```

Kubernetes DNS is used instead of a fixed pod IP.

## ClickHouse

Each ClickHouse replica has its own stable headless-service DNS name.

Port:

```text
9363
```

## cAdvisor

Uses Kubernetes service discovery:

```yaml
kubernetes_sd_configs:
  - role: node
```

Therefore it discovers Kubernetes nodes automatically.

The node name is copied to:

```text
node
```

label.

## Keeper

Each Keeper pod is scraped individually on:

```text
9090
```

This is important because monitoring each Keeper replica separately lets us see which Keeper instance is healthy.

---

# 54. vmagent Deployment

Final deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vmagent
  namespace: observability

spec:
  replicas: 1

  selector:
    matchLabels:
      app: vmagent

  template:
    metadata:
      labels:
        app: vmagent

    spec:
      serviceAccountName: vmagent

      containers:
        - name: vmagent
          image: victoriametrics/vmagent:v1.151.0

          args:
            - "-promscrape.config=/etc/vmagent/prometheus.yml"
            - "-remoteWrite.url=http://172.31.16.39:8428/api/v1/write"

          ports:
            - name: http
              containerPort: 8429

          volumeMounts:
            - name: vmagent-config
              mountPath: /etc/vmagent

      volumes:
        - name: vmagent-config
          configMap:
            name: vmagent-config
```

Important:

```text
8429
```

is vmagent's HTTP/API port.

The remote write destination is:

```text
172.31.16.39:8428
```

---

# 55. Why remote_write is a vmagent Argument

An early configuration attempt placed `remote_write` in the Prometheus configuration incorrectly.

The final working approach passes:

```text
-remoteWrite.url=http://172.31.16.39:8428/api/v1/write
```

as a vmagent command-line argument.

This is the configuration used in the final POC.

---

# 56. Apply vmagent Configuration

Validate:

```bash
kubectl apply --dry-run=client \
-f ~/clickhouse-poc/monitoring/vmagent-config.yml
```

Expected:

```text
configmap/vmagent-config configured (dry run)
```

Apply:

```bash
kubectl apply \
-f ~/clickhouse-poc/monitoring/vmagent-config.yml
```

Restart:

```bash
kubectl rollout restart deployment/vmagent \
-n observability
```

Verify:

```bash
kubectl get pods \
-n observability \
-l app=vmagent
```

Expected:

```text
1/1 Running
```

---

# 57. vmagent Target API

Port-forward:

```bash
kubectl port-forward \
deployment/vmagent \
-n observability \
8429:8429
```

Keep that terminal open.

In another terminal:

```bash
curl -s \
http://localhost:8429/api/v1/targets
```

This returns:

- discovered labels
- target URL
- last scrape time
- scrape duration
- samples scraped
- health
- last error

This API was used extensively to troubleshoot monitoring.

---

# 58. Final Monitoring Target Count

The final configuration contains:

```text
Node Exporter       3
cAdvisor            4
ClickHouse          3
ClickHouse Keeper   3
KSM                 1
                    ---
Total               14
```

The four cAdvisor targets are:

```text
k8s-master
k8s-worker1
k8s-worker2
k8s-worker3
```

The three Node Exporter targets are the three workers.

---

# 59. Final Target Verification

The final vmagent target output showed:

```text
cAdvisor worker1 → UP → 1771 samples
cAdvisor worker2 → UP → 1837 samples
cAdvisor worker3 → UP → 2147 samples
cAdvisor master  → UP → 1957 samples
```

Node Exporter:

```text
worker1 → UP
worker2 → UP
worker3 → UP
```

ClickHouse:

```text
CH-0 → UP → ~3270 samples
CH-1 → UP → ~3270 samples
CH-2 → UP → ~3270 samples
```

Keeper:

```text
Keeper-0 → UP → 726 samples
Keeper-1 → UP → 726 samples
Keeper-2 → UP → 726 samples
```

KSM:

```text
KSM → UP → 2314 samples
```

Therefore:

```text
14 / 14 monitoring targets = UP
```

---

# 60. VictoriaMetrics

VictoriaMetrics is running on a separate server.

Private IP:

```text
172.31.16.39
```

Port:

```text
8428
```

vmagent sends:

```text
http://172.31.16.39:8428/api/v1/write
```

The Kubernetes worker-to-VictoriaMetrics connectivity was verified successfully.

Required security group rule:

```text
VictoriaMetrics Server SG
Inbound TCP 8428
Source: Worker Security Group
```

The public internet should not be used as the source for this internal POC traffic.

---

# 61. What VictoriaMetrics Does

vmagent collects samples.

VictoriaMetrics stores and serves those samples.

Conceptually:

```text
Exporter
   |
   | /metrics
   v
vmagent
   |
   | remote_write
   v
VictoriaMetrics
   |
   | query
   v
Grafana
```

---

# 62. Grafana Role

Grafana is the visualization layer.

It does not replace vmagent.

It queries the metrics stored in VictoriaMetrics and turns them into:

- graphs
- gauges
- tables
- stat panels
- alerts

For this POC the existing Grafana server can use VictoriaMetrics as its data source.

---

# 63. Suggested Grafana Dashboard Sections

The final dashboard can be divided into:

### Infrastructure

- CPU utilization
- Memory utilization
- Filesystem usage
- Network traffic
- Node availability

### Kubernetes

- Node Ready status
- Pod count
- Pod restart count
- Deployment availability
- PVC status
- Container CPU
- Container memory

### ClickHouse

- Replica availability
- Query activity
- Insert activity
- Merge activity
- Profile events
- Background tasks

### Keeper

- Keeper target availability
- Raft-related metrics
- Keeper events
- Keeper server activity

### HA / Replication

- Active replicas
- Replication queue
- Replication delay
- Read-only status
- Replica health

---

# 64. Important Verification Commands

## Kubernetes

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
```

## ClickHouse

```bash
kubectl get clickhousecluster -n clickhouse
kubectl get pods -n clickhouse -o wide
kubectl get pvc -n clickhouse
```

## Keeper

```bash
kubectl get keepercluster -n clickhouse
kubectl get pods -n clickhouse -o wide | grep keeper
```

## Monitoring

```bash
kubectl get pods -n observability
```

## vmagent

```bash
kubectl port-forward deployment/vmagent \
-n observability 8429:8429

curl -s http://localhost:8429/api/v1/targets
```

---

# 65. Useful ClickHouse Replication Queries

Check replicas:

```sql
SELECT
    database,
    table,
    is_leader,
    is_readonly,
    total_replicas,
    active_replicas
FROM system.replicas
WHERE database = 'ha_test';
```

Check replication queue:

```sql
SELECT
    database,
    table,
    queue_size,
    inserts_in_queue,
    absolute_delay
FROM system.replicas
WHERE database = 'ha_test';
```

Check table engine:

```sql
SELECT
    database,
    name,
    engine,
    replica_name,
    zookeeper_path
FROM system.tables
WHERE database = 'ha_test';
```

---

# 66. Troubleshooting: Keeper Stale Storage

During the Keeper setup, stale state was encountered.

The cleanup sequence used for the POC was:

Delete ClickHouse:

```bash
kubectl delete clickhousecluster clickhouse -n clickhouse
```

Delete Keeper:

```bash
kubectl delete keepercluster clickhouse-keeper -n clickhouse
```

Delete Keeper PVCs:

```bash
kubectl delete pvc \
clickhouse-storage-volume-clickhouse-keeper-keeper-0-0 \
clickhouse-storage-volume-clickhouse-keeper-keeper-1-0 \
clickhouse-storage-volume-clickhouse-keeper-keeper-2-0 \
-n clickhouse
```

On each worker:

```bash
sudo rm -rf /data/keeper/*
```

Release PV claim references:

```bash
kubectl patch pv keeper-pv-worker1 \
--type=json \
-p='[{"op":"remove","path":"/spec/claimRef"}]'

kubectl patch pv keeper-pv-worker2 \
--type=json \
-p='[{"op":"remove","path":"/spec/claimRef"}]'

kubectl patch pv keeper-pv-worker3 \
--type=json \
-p='[{"op":"remove","path":"/spec/claimRef"}]'
```

Then Keeper was recreated cleanly.

Final result:

```text
Keeper cluster ready
3/3 replicas
```

---

# 67. Troubleshooting: ClickHouse Authentication

A `Not authenticated` problem was encountered during early ClickHouse testing.

After the ClickHouse/Operator configuration was corrected and the cluster was recreated in the clean final state, the ClickHouse logs showed normal operation and the replication test succeeded.

The final cluster state did not show the authentication error.

---

# 68. Troubleshooting: Calico

An initial Calico/Tigera Operator path failed on Kubernetes 1.29 because the selected Calico version attempted to use a CEL function that was not supported in that Kubernetes environment.

The failed installation was removed.

A raw Calico v3.27.0 manifest was then used.

The initial VXLAN path also had decapsulation/connectivity problems in the AWS environment.

The final working mode was:

```text
IP-in-IP
BIRD backend
VXLAN disabled
```

After migration:

- `tunl0` existed
- BGP peers were established
- cross-worker pod pings worked
- DNS worked
- ClickHouse/Keeper communication worked

This is documented as a troubleshooting history; the final POC uses the working configuration.

---

# 69. Troubleshooting: vmagent Configuration

An early vmagent configuration incorrectly attempted to place remote-write configuration in the Prometheus scrape configuration.

The final configuration uses:

```text
-remoteWrite.url=...
```

as a vmagent argument.

This successfully initialized the remote-write client and connected vmagent to VictoriaMetrics.

---

# 70. Troubleshooting: KSM Direct curl

A direct attempt to use `wget` inside the KSM container failed because that image did not contain `wget`.

This was not a KSM failure.

The monitoring path was instead validated through vmagent scraping the KSM Service.

The important lesson:

> A missing utility inside a minimal container does not mean the application endpoint is broken.

Use an external temporary curl container when needed.

---

# 71. Security Group Summary

The POC used incremental internal security-group rules.

Important final rules included:

### Kubernetes API

```text
Master SG
TCP 6443
Source: Worker SG
```

### Kubelet

```text
Worker SG
TCP 10250
Source: Master SG
Worker SG
```

### Node Exporter

```text
Worker SG
TCP 9100
Source: Worker SG
```

A temporary Master-to-worker Node Exporter access rule was also used for direct testing.

### VictoriaMetrics

```text
VictoriaMetrics SG
TCP 8428
Source: Worker SG
```

### Calico IP-in-IP

```text
IP protocol 4
```

was allowed between the required node security groups.

### Important security principle

Do not expose:

```text
10250
9100
8428
```

to:

```text
0.0.0.0/0
```

for this internal POC.

Use security-group references or restricted internal sources.

---

# 72. Why the POC Uses a Single Control Plane

The project requirement was ClickHouse HA.

Therefore:

```text
Kubernetes control plane:
1 node

ClickHouse:
3 replicas

Keeper:
3 nodes
```

This means a Kubernetes control-plane failure is outside the HA scope of this POC.

The HA test is focused on the ClickHouse data/replica layer and its coordination service.

---

# 73. Failure Domains

Final placement:

```text
worker1 → us-east-1c
worker2 → us-east-1a
worker3 → us-east-1b
```

ClickHouse:

```text
CH-1 → worker1 → us-east-1c
CH-0 → worker2 → us-east-1a
CH-2 → worker3 → us-east-1b
```

Keeper:

```text
Keeper-1 → worker1 → us-east-1c
Keeper-0 → worker2 → us-east-1a
Keeper-2 → worker3 → us-east-1b
```

This is useful for demonstrating that replicas are not intentionally concentrated on one worker.

---

# 74. End-to-End Data Flow

A write into ClickHouse can be understood conceptually as:

```text
Application
    |
    v
ClickHouse replica
    |
    v
ReplicatedMergeTree
    |
    +------> local data part
    |
    +------> replication coordination
                 |
                 v
              Keeper
                 |
                 v
        other ClickHouse replicas
```

Monitoring is separate:

```text
ClickHouse / Keeper / Kubernetes / Node
                |
                v
             metrics
                |
                v
              vmagent
                |
                v
         VictoriaMetrics
                |
                v
              Grafana
```

---

# 75. What Each Component Is Responsible For

| Component | Responsibility |
|---|---|
| Kubernetes | Runs and schedules workloads |
| Containerd | Runs containers |
| Calico | Provides pod networking |
| ClickHouse Operator | Manages ClickHouse custom resources |
| ClickHouse | Stores and queries analytical data |
| MergeTree | ClickHouse storage engine family |
| ReplicatedMergeTree | Replicated ClickHouse table engine |
| Keeper | Coordination and distributed state |
| Local PV | Persistent node-local storage |
| Node Exporter | Host metrics |
| cAdvisor | Container metrics |
| KSM | Kubernetes object/state metrics |
| vmagent | Scraping + remote write |
| VictoriaMetrics | Metric storage/query |
| Grafana | Visualization |

---

# 76. Final POC Validation Checklist

## Kubernetes

- [x] Master initialized
- [x] Three workers joined
- [x] Containerd used as runtime
- [x] Calico networking working
- [x] DNS working

## ClickHouse

- [x] Operator installed
- [x] Storage configured
- [x] Keeper cluster created
- [x] 3 Keeper replicas healthy
- [x] ClickHouse cluster created
- [x] 1 shard / 3 replicas
- [x] ReplicatedMergeTree created
- [x] Data replicated
- [x] Replication queue verified
- [x] ClickHouse pod failure/recovery tested

## Monitoring

- [x] Node Exporter deployed
- [x] cAdvisor verified through kubelet
- [x] KSM deployed
- [x] ClickHouse metrics enabled
- [x] Keeper metrics enabled
- [x] vmagent deployed
- [x] vmagent RBAC configured
- [x] VictoriaMetrics remote write configured
- [x] All targets verified

Final:

```text
14 / 14 monitoring targets UP
```

---

# 77. Final Architecture in One Diagram

```text
                           +----------------------+
                           |      Grafana         |
                           +----------+-----------+
                                      |
                                      | Query
                                      v
                           +----------------------+
                           |   VictoriaMetrics    |
                           |      :8428           |
                           +----------^-----------+
                                      |
                               remote_write
                                      |
                           +----------+-----------+
                           |       vmagent        |
                           |       :8429           |
                           +----------^-----------+
                                      |
              +-----------------------+-----------------------+
              |           |            |          |            |
              |           |            |          |            |
              v           v            v          v            v
         Node Exporter  cAdvisor     KSM      ClickHouse     Keeper
           :9100        :10250       :8080       :9363        :9090
              |           |            |          |            |
              +-----------+------------+----------+------------+
                                      |
                              Kubernetes Cluster
                                      |
                  +-------------------+-------------------+
                  |                   |                   |
               worker1             worker2             worker3
                  |                   |                   |
                CH-1                CH-0                CH-2
              Keeper-1            Keeper-0            Keeper-2
                  |                   |                   |
                  +---------- Replication / -------------+
                             Coordination
                                  |
                                Keeper
```

---

# 78. Final Outcome

The POC successfully implemented a ClickHouse cluster with:

```text
1 shard
3 replicas
3 Keeper nodes
3 Kubernetes workers
```

Data replication was verified using `ReplicatedMergeTree`.

A ClickHouse pod was deleted and automatically recreated by the operator, and the cluster returned to a healthy 3/3 replica state.

Monitoring was implemented for:

```text
3 Node Exporters
4 cAdvisor targets
1 KSM target
3 ClickHouse targets
3 Keeper targets
```

Total:

```text
14 monitoring targets
```

Final verification showed all 14 targets in the `UP` state.

The complete monitoring flow is:

```text
Kubernetes / ClickHouse / Keeper
              |
              v
           vmagent
              |
              v
       VictoriaMetrics
              |
              v
            Grafana
```

---

# 79. Short Fresher Explanation

If you need to explain the complete POC to someone in an interview or review:

> We created a Kubernetes cluster with one control plane and three workers. On the three workers we deployed a one-shard, three-replica ClickHouse cluster using the ClickHouse Operator. Each ClickHouse replica has its own persistent storage and is distributed across the workers. ClickHouse Keeper runs as a three-node coordination cluster and handles coordination for replicated ClickHouse tables. We created a ReplicatedMergeTree table, inserted data into one replica, and verified that the data appeared on the other replicas. We also deleted one ClickHouse pod and verified that the operator recreated it and the cluster returned to three healthy replicas.
>
> For monitoring, we used Node Exporter for host metrics, kubelet cAdvisor for container metrics, kube-state-metrics for Kubernetes object metrics, ClickHouse's Prometheus endpoint for database metrics, and Keeper's Prometheus endpoint for Keeper metrics. vmagent scrapes all of these targets and sends the metrics to an existing VictoriaMetrics server using remote write. Grafana can then query VictoriaMetrics and visualize the health and performance of the complete environment.

---

# 80. Important Notes About This Document

This document is based on the implementation and verification performed during the POC.

Where a configuration was directly verified, the final value is documented.

The exact local PV manifest text was not preserved in the retrieved conversation artifacts, so the storage section documents the verified deployed PV topology and parameters rather than claiming an unretrieved YAML file is byte-for-byte identical to the original.

For a production deployment, storage would normally be designed with a production-grade CSI-backed storage system rather than node-local hostPath-style storage.
