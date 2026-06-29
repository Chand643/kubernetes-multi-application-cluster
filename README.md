# 🚀 Kubernetes Multi-Application Cluster

> Enterprise-style Kubernetes Cluster deployed using **kubeadm**, **containerd**, and **Calico** on **4 Ubuntu Virtual Machines**.

![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.30-blue?logo=kubernetes)
![Ubuntu](https://img.shields.io/badge/Ubuntu-22.04-E95420?logo=ubuntu)
![Containerd](https://img.shields.io/badge/Container_Runtime-containerd-blue)
![Calico](https://img.shields.io/badge/CNI-Calico-green)
![Status](https://img.shields.io/badge/Project-Completed-success)

---

# 📖 Project Overview

This project demonstrates the deployment of a **production-style Kubernetes cluster** using **kubeadm** with one Master Node and three Worker Nodes.

Multiple containerized applications were deployed across the cluster to demonstrate:

- Kubernetes Deployments
- Pods
- ReplicaSets
- Services
- NodePort
- ClusterIP
- Secrets
- ConfigMaps
- Scaling
- Self-Healing
- Monitoring
- Container Networking

---

# 🏗 Cluster Architecture

```
                         Internet
                              │
                              │
                    ┌─────────────────┐
                    │   Master Node   │
                    │ 192.168.0.2     │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          │                  │                  │
 ┌────────────────┐  ┌────────────────┐  ┌────────────────┐
 │   Worker-1     │  │   Worker-2     │  │   Worker-3     │
 │ 192.168.0.3    │  │ 192.168.0.4    │  │ 192.168.0.5    │
 └────────────────┘  └────────────────┘  └────────────────┘
```

---

# ⚙️ Technology Stack

| Component | Technology |
|------------|------------|
| Operating System | Ubuntu Server 22.04 |
| Container Runtime | containerd |
| Kubernetes | kubeadm |
| Networking | Calico |
| Monitoring | Prometheus + Grafana |
| Database | MySQL |
| Cache | Redis |
| Web Server | Nginx |
| HTTP Server | Apache |
| Backend | Node.js |

---

# 📦 Applications Deployed

## 🌐 Nginx Web Application

- 6 Replicas
- ConfigMap
- NodePort Service
- Custom Landing Page

---

## 🔥 Apache HTTP Server

- 3 Replicas
- Custom Landing Page
- NodePort Service

---

## 🟢 Node.js Backend API

- REST Backend
- 3 Replicas
- NodePort Service

---

## 🐬 MySQL Database

- Secret Management
- ClusterIP Service
- Internal Database Communication

---

## 📊 Redis Cache

- In-memory Cache
- ClusterIP Service

---

## 📈 Monitoring Stack

- Prometheus
- Grafana
- Kubernetes Metrics
- Node Monitoring
- Pod Monitoring

---

# 📂 Repository Structure

```
.
├── apache-website.yaml
├── mysql-clean.yaml
├── nginx-website.yaml
├── nodejs-api.yaml
├── redis.yaml
├── nodes-output.txt
├── pods-output.txt
├── services-output.txt
└── README.md
```

---

# 🚀 Features

✅ Multi-node Kubernetes Cluster

✅ Container Orchestration

✅ Load Balancing

✅ Horizontal Scaling

✅ Self-Healing Pods

✅ Internal Cluster Networking

✅ Service Discovery

✅ ConfigMaps

✅ Secrets

✅ Monitoring with Prometheus

✅ Visualization with Grafana

---

# 📊 Cluster Information

```
Master Nodes : 1

Worker Nodes : 3

Applications : 5

Pods : 13+

Deployments : 5

Services : Multiple

Monitoring : Enabled
```

---

# 📸 Project Screenshots

Add screenshots here:

- Cluster Architecture
- Nodes
- Pods
- Services
- Nginx
- Apache
- MySQL
- Redis
- Grafana Dashboard
- Prometheus Dashboard

---

# 🖥 Useful Commands

```bash
kubectl get nodes

kubectl get pods -A

kubectl get svc -A

kubectl get deployments

kubectl get namespaces

kubectl top nodes

kubectl top pods
```

---

# 📈 Learning Outcomes

Through this project, I gained practical experience with:

- Kubernetes Cluster Deployment
- kubeadm
- Container Runtime (containerd)
- Calico Networking
- Pod Scheduling
- Deployments
- ReplicaSets
- Services
- ConfigMaps
- Secrets
- Monitoring Stack
- Enterprise Kubernetes Administration

---

# 👨‍💻 Author

**Chand Parveen**

B.Tech Computer Science Engineering

DevOps | Cloud | Kubernetes | Cybersecurity

GitHub: https://github.com/Chand643
