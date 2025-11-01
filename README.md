# Network-Aware Kubernetes Scheduler

> **A production-grade Kubernetes scheduler extension that makes intelligent pod placement decisions based on real-time network topology and microservice communication patterns.**

[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.32+-blue)](https://kubernetes.io/)
[![Go](https://img.shields.io/badge/Go-1.19+-00ADD8)](https://golang.org/)
[![Python](https://img.shields.io/badge/Python-3.8+-3776AB)](https://python.org/)
[![License](https://img.shields.io/badge/License-Apache%202.0-green)](LICENSE)

---

## 📋 Overview

This project implements a **network-aware scheduling framework** for Kubernetes that optimizes pod placement by integrating real-time network metrics (latency, bandwidth, packet loss) with microservice dependency graphs. Unlike the default Kubernetes scheduler, which focuses primarily on resource availability, this system makes **communication-aware placement decisions** that reduce inter-service latency, improve application throughput, and enhance overall system performance.

**Key Innovation**: The scheduler dynamically queries live network topology data and service dependency graphs during the scoring phase, enabling it to co-locate frequently communicating microservices and place them in network-optimal zones.

---

## 🎯 Motivation

Modern microservices architectures deploy hundreds of interdependent services across distributed clusters. The default Kubernetes scheduler treats all nodes as network-equivalent, leading to suboptimal placements that can result in:

- **Increased tail latency** due to cross-zone/remote node communication
- **Reduced bandwidth utilization** when high-traffic services communicate over constrained links
- **Poor locality** where dependent services are scattered across the cluster

This project addresses these challenges by building a **network topology-aware scheduling system** that:

1. **Continuously measures** inter-node network characteristics using lightweight probe agents
2. **Models microservice dependencies** through declarative AppGroup CRDs
3. **Computes placement scores** that favor network-efficient arrangements
4. **Integrates seamlessly** with existing Kubernetes scheduling infrastructure

---

## ✨ Key Features & Highlights

### 🔧 Technical Capabilities

- **Custom Kubernetes Scheduler Plugin**: Extends the official Kubernetes Scheduling Framework with a `NetworkScore` plugin that evaluates nodes based on network metrics
- **Real-Time Network Monitoring**: Distributed probe agents (DaemonSet) continuously measure latency, bandwidth, and packet loss between all cluster nodes using `ping` and `iperf3`
- **Declarative Dependency Modeling**: Two Custom Resource Definitions (CRDs) enable operators to:
  - Define network topology snapshots (`NetworkTopology`)
  - Specify microservice communication graphs with priority weights (`AppGroup`)
- **Hybrid Scheduling Modes**: Supports pure network-aware scheduling or hybrid approaches (e.g., 50/50 blend with default scheduler scoring)
- **Centralized Aggregation**: Network aggregator service collects probe data, normalizes metrics, and exposes them via REST API for scheduler consumption
- **Zero-Intrusion Design**: Compatible with existing workloads; only pods with `appgroup` labels are scheduled using network-aware logic

### 🏗️ System Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                           │
│                                                                   │
│  ┌──────────────┐      ┌──────────────┐      ┌──────────────┐  │
│  │   Node 1     │      │   Node 2     │      │   Node N     │  │
│  │              │      │              │      │              │  │
│  │ ┌──────────┐ │      │ ┌──────────┐ │      │ ┌──────────┐ │  │
│  │ │  Probe   │ │◄────►│ │  Probe   │ │◄────►│ │  Probe   │ │  │
│  │ │  Agent   │ │      │ │  Agent   │ │      │ │  Agent   │ │  │
│  │ └──────────┘ │      │ └──────────┘ │      │ └──────────┘ │  │
│  └──────┬───────┘      └──────┬───────┘      └──────┬───────┘  │
│         │                      │                      │          │
│         └──────────────────────┼──────────────────────┘          │
│                                ▼                                  │
│                    ┌──────────────────────┐                       │
│                    │ Network Aggregator   │                       │
│                    │  (Flask REST API)    │                       │
│                    └──────────┬───────────┘                       │
│                               │                                    │
│                               ▼                                    │
│                    ┌──────────────────────┐                       │
│                    │ NetworkTopology CRD  │                       │
│                    └──────────────────────┘                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Custom kube-scheduler (extended)               │    │
│  │                                                           │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │   Filter     │  │    Score     │  │    Bind      │  │    │
│  │  │   Phase      │─►│    Phase     │─►│    Phase     │  │    │
│  │  └──────────────┘  └──────┬───────┘  └──────────────┘  │    │
│  │                            │                            │    │
│  │                    ┌───────▼────────┐                   │    │
│  │                    │ NetworkScore   │                   │    │
│  │                    │    Plugin      │                   │    │
│  │                    └───────┬────────┘                   │    │
│  │                            │                            │    │
│  │          ┌─────────────────┴─────────────────┐          │    │
│  │          ▼                                   ▼          │    │
│  │  ┌──────────────┐              ┌──────────────────┐   │    │
│  │  │ AppGroup     │              │ NetworkTopology  │   │    │
│  │  │ Controller   │              │      CRD         │   │    │
│  │  │  (Python)    │              │                  │   │    │
│  │  └──────────────┘              └──────────────────┘   │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 📊 Core Components

| Component | Technology | Responsibility |
|-----------|-----------|----------------|
| **NetworkScore Plugin** | Go (Kubernetes Scheduling Framework) | Scores nodes during pod scheduling based on network metrics and service dependencies |
| **Probe Agent** | Python (DaemonSet) | Measures RTT (ping), bandwidth (iperf3), and packet loss every 30 seconds |
| **Network Aggregator** | Python Flask | Aggregates probe data from all nodes, normalizes metrics, updates NetworkTopology CRD |
| **AppGroup Controller** | Python Flask | Watches AppGroup CRDs and serves dependency graphs via REST API |
| **Custom Scheduler** | Go | Extended kube-scheduler binary with NetworkScore plugin registered |

---

## 🛠️ Technical Implementation

### Custom Resource Definitions (CRDs)

#### 1. NetworkTopology

Represents the current network state of the cluster as a matrix of node-to-node metrics:

```yaml
apiVersion: scheduling.mygroup.io/v1
kind: NetworkTopology
metadata:
  name: cluster-network
spec:
  latency:        # RTT in milliseconds (node → node mapping)
  bandwidth:      # Throughput in Mbps (node → node mapping)
  lossrate:       # Packet loss percentage (node → node mapping)
  maxLatency:     # Normalization bounds
  minLatency:
  maxBandwidth:
  minBandwidth:
  # ... etc
```

#### 2. AppGroup

Models microservice dependencies with weighted communication preferences:

```yaml
apiVersion: scheduling.mygroup.io/v1
kind: AppGroup
metadata:
  name: boutique
spec:
  workloads:
    - name: frontend
      weight: 1.0          # Importance/traffic weight
      dependencies:
        - name: recommendationservice
          metrics:
            latency: 0.6    # Priority weight for latency optimization
            bandwidth: 0.3
            lossrate: 0.1
```

### Scoring Algorithm

The `NetworkScore` plugin computes a node score (0-100) for each pod by:

1. **Fetching AppGroup graph** for the pod's `appgroup` label
2. **Retrieving NetworkTopology** matrix
3. **Evaluating forward dependencies**: For each service the pod depends on, compute average network cost to existing replicas
4. **Evaluating reverse dependencies**: For each service that depends on this pod, compute cost from their replicas to this node
5. **Weighted aggregation**: Sum scores weighted by service importance (`weight` field)
6. **Normalization**: Normalize latency, bandwidth (log-scale), and loss rate metrics to [0,1] range

**Formula** (simplified):

```text
score = Σ (service_weight × avg_network_score_to_dependencies) 
      + Σ (dependent_weight × avg_network_score_from_dependents)
```

Where `network_score` considers:

- **Same-node placement**: Automatic high score (0.8 baseline)
- **Latency**: Inversely normalized (lower latency → higher score)
- **Bandwidth**: Log-normalized (higher bandwidth → higher score)
- **Packet loss**: Inversely normalized (lower loss → higher score)

### Network Measurement Pipeline

1. **Probe Agents** (DaemonSet, one per node):
   - Every 30s: `ping -c 5` to all other nodes → RTT + packet loss
   - Every 360s: `iperf3 -t 2` to all other nodes → bandwidth (cached between runs)
   - POST metrics to aggregator service

2. **Network Aggregator**:
   - Receives reports from all probe agents
   - Maintains in-memory topology matrices
   - Every 60s: Updates `NetworkTopology` CRD with aggregated data
   - Exposes `/topology` REST endpoint for scheduler queries

3. **AppGroup Controller**:
   - Watches `AppGroup` CRD events (ADDED/MODIFIED/DELETED)
   - Maintains in-memory graph cache
   - Exposes `/graph/<appgroup>` REST endpoint

---

## 🧪 Experiment Setup & Results

### Experimental Environment

- **Cluster**: 19-node Kubernetes cluster (1 master + 18 workers)
- **Network Emulation**: Linux `tc` (traffic control) to create zone-based topologies:
  - **Zone A/B/C**: Low latency (<2ms), high bandwidth (>900 Mbps) within zone
  - **Cross-zone**: Higher latency (4-5ms), reduced bandwidth (30-50 Mbps)
  - **Far zones**: Very high latency (20ms+), low bandwidth (<5 Mbps), packet loss
- **Benchmark Application**: [Google Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) (11 microservices with complex dependencies)
- **Load Testing**: Locust for generating realistic HTTP traffic patterns

### Experimental Configurations

1. **Default Scheduler**: Standard Kubernetes scheduler (baseline)
2. **Custom Scheduler (100%)**: Pure network-aware scheduling
3. **Hybrid 50/50**: 50% network score + 50% default scheduler score
4. **Hybrid 80/20**: 80% network score + 20% default scheduler score

### Key Results

#### Placement Quality Improvement

| Metric | Default Scheduler | Custom Scheduler (100%) | Improvement |
|--------|------------------|------------------------|-------------|
| **Total Placement Score** | 1,600.9 | 2,285.0 | **+42.7%** |
| **Average Score per App** | 64.3 | 91.8 | **+42.8%** |

*Placement score reflects aggregated network cost considering latency, bandwidth, and loss rate weighted by service dependencies.*

#### Application-Level Benefits

- **Improved Pod Locality**: Services with high communication frequency (e.g., `frontend` → `recommendationservice`) were co-located more frequently
- **Reduced Cross-Zone Traffic**: Network-aware scheduler placed dependent services in the same or adjacent network zones
- **Better Resource Utilization**: Placement decisions respected both resource constraints (CPU/memory) and network topology

#### System Overhead

- **Probe Agent**: <5 mCPU, <20 MiB memory per node
- **Network Aggregator**: <10 mCPU, <50 MiB memory
- **AppGroup Controller**: <5 mCPU, <30 MiB memory
- **Scheduling Latency**: <100ms additional overhead per pod (network queries + scoring)

*Measured on production-like workloads with 100+ pods across 19 nodes.*

---

## 📁 Repository Structure

```text
.
├── scheduler-plugins/          # Custom Kubernetes scheduler with NetworkScore plugin
│   ├── cmd/scheduler/main.go   # Scheduler binary entrypoint
│   └── pkg/networkscore/       # NetworkScore plugin implementation
│
├── probe/                      # Network probe agent (DaemonSet)
│   ├── probe.py               # Ping/iperf3 measurement logic
│   └── dockerfile             # Container image
│
├── net-aggregator/            # Network metrics aggregation service
│   ├── app.py                # Flask REST API + CRD updater
│   └── dockerfile
│
├── appgroup_controller/       # AppGroup CRD controller
│   ├── main.py               # Kubernetes watch loop + REST API
│   └── dockerfile
│
├── manifest/                  # Kubernetes manifests
│   ├── networktopology-crd.yaml
│   ├── appgroup-crd.yaml
│   ├── network-probe-agent.yaml
│   ├── network-aggregator.yaml
│   └── appgroup-controller.yaml
│
├── expr/                      # Experiment configurations
│   ├── online_boutique/      # Google Online Boutique benchmark
│   └── scheduler/            # Scheduler profile configs
│
├── ansible_tc/               # Network emulation automation
│   └── deploy_tc.yaml        # Ansible playbook for tc configuration
│
└── stats/                     # Analysis and visualization scripts
    ├── compute_cost.py       # Placement score evaluation
    └── compare_scheduler_cost.py
```

---

## 🚀 Deployment & Usage

### Prerequisites

- Kubernetes cluster (v1.28+)
- `kubectl` configured with cluster access
- `iperf3` installed on all nodes (for bandwidth measurement)

### Quick Start

1. **Install CRDs**:

   ```bash
   kubectl apply -f manifest/networktopology-crd.yaml
   kubectl apply -f manifest/appgroup-crd.yaml
   ```

2. **Deploy Monitoring Infrastructure**:

   ```bash
   # Deploy probe agents (DaemonSet)
   kubectl apply -f manifest/network-probe-agent.yaml
   
   # Deploy network aggregator
   kubectl apply -f manifest/network-aggregator.yaml
   
   # Deploy AppGroup controller
   kubectl apply -f manifest/appgroup-controller.yaml
   ```

3. **Build and Deploy Custom Scheduler**:

   ```bash
   # Build scheduler image (see scheduler-plugins/dockerfile)
   docker build -t network-aware-scheduler:latest ./scheduler-plugins
   
   # Deploy scheduler (replace default kube-scheduler or run alongside)
   kubectl apply -f expr/scheduler/custom-scheduler.yaml
   ```

4. **Define AppGroup for Your Application**:

   ```bash
   kubectl apply -f expr/online_boutique/appgroup.yaml
   ```

5. **Deploy Application with AppGroup Labels**:

   ```yaml
   metadata:
     labels:
       appgroup: boutique  # Must match AppGroup name
       app: frontend       # Must match AppGroup workload name
   ```

### Network Emulation (Optional)

For controlled experiments with varying network conditions:

```bash
# Deploy network emulation via Ansible
ansible-playbook -i ansible_tc/inventory.ini ansible_tc/deploy_tc.yaml

# Clear emulation
ansible-playbook -i ansible_tc/inventory.ini ansible_tc/clear_tc.yaml
```

---

## 👤 Contributions / My Role

This project was developed as a **graduate-level research and implementation effort** demonstrating expertise in:

### Technical Skills Demonstrated

- **Kubernetes Internals**: Deep understanding of the Scheduling Framework, CRDs, plugin architecture, and cluster operations
- **Systems Programming**: Go development for scheduler plugin; Python for distributed monitoring services
- **Distributed Systems Design**: Real-time metric collection, aggregation pipelines, and eventual consistency patterns
- **Network Engineering**: Understanding of RTT, bandwidth, packet loss; experience with Linux `tc` for network emulation
- **Performance Evaluation**: Systematic experimentation with controlled variables, benchmarking, and quantitative analysis

### Key Achievements

- ✅ Designed and implemented a production-ready Kubernetes scheduler extension
- ✅ Built a scalable network monitoring system with minimal overhead (<5% resource usage)
- ✅ Demonstrated measurable performance improvements (42% better placement scores)
- ✅ Integrated seamlessly with existing Kubernetes infrastructure (zero breaking changes)
- ✅ Created reusable CRD abstractions for network-aware scheduling

### Technologies Used

- **Languages**: Go, Python
- **Frameworks**: Kubernetes Scheduling Framework, Flask, Kubernetes Python Client
- **Tools**: Docker, Ansible, iperf3, tc (traffic control), Locust
- **Concepts**: Custom Resource Definitions, DaemonSets, REST APIs, metric aggregation

---

## 🔮 Future Work

Potential enhancements and research directions:

1. **Adaptive Weight Tuning**: Machine learning-based optimization of metric weights based on application performance feedback
2. **Multi-Cluster Support**: Extend network topology awareness across Kubernetes clusters (federation scenarios)
3. **Historical Trend Analysis**: Incorporate network metric history to predict optimal placements under varying conditions
4. **Scheduler Performance**: Optimize scoring algorithm complexity for larger clusters (1000+ nodes)
5. **Integration with Service Mesh**: Leverage Istio/Linkerd telemetry data as additional network awareness signals
6. **Cost-Aware Placement**: Combine network optimization with cloud provider cost models (cross-zone pricing)

---

## 📄 License

This project is licensed under the Apache License 2.0. See LICENSE file for details.

---

## 📚 References

- [Kubernetes Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Google Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo)
- [Kubernetes Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)

---

## 🔗 Repository

**GitHub**: [https://github.com/YongyuLiu03/CS525_Project](https://github.com/YongyuLiu03/CS525_Project)

---

Built with ❤️ for better Kubernetes scheduling
