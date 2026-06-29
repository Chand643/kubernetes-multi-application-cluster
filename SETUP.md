````markdown
# Kubernetes Multi-Application Cluster Setup Guide

## Project Architecture

This project is built on a 4-node Kubernetes cluster using Ubuntu virtual machines.

| Node          | IP Address   | Role              |
|---            |---          |---                 |
| Master Node   | 192.168.0.2 | Control Plane      |
| Worker Node 1 | 192.168.0.3 | Application Worker |
| Worker Node 2 | 192.168.0.4 | Application Worker |
| Worker Node 3 | 192.168.0.5 | Application Worker |

The master node manages the Kubernetes cluster, while the worker nodes run application workloads such as Nginx, Apache, Node.js, MySQL, Redis, Prometheus, and Grafana.

## Architecture Flow

                   ```text
                  Internet
                     |
               Public IP / NAT
                     |
            Master Node - 192.168.0.2
                     |
           Kubernetes Control Plane
                     |
------------------------------------------------
|                    |                         |
Worker 1             Worker 2                 Worker 3
192.168.0.3          192.168.0.4              192.168.0.5
    |                    |                         |
Nginx Pods           Apache Pods              Node.js Pods
 MySQL Pod            Redis Pod               Monitoring Pods
````

## Components Used

| Component  | Purpose                            |
| ---------- | ---------------------------------- |
| kubeadm    | Kubernetes cluster initialization  |
| kubelet    | Node agent that runs on every node |
| kubectl    | Kubernetes command-line tool       |
| containerd | Container runtime                  |
| Calico     | Kubernetes networking plugin       |
| Nginx      | Web application                    |
| Apache     | Web application                    |
| Node.js    | Backend API                        |
| MySQL      | Database                           |
| Redis      | Cache                              |
| Prometheus | Monitoring                         |
| Grafana    | Visualization dashboard            |

---

# Step-by-Step Setup Commands

## 1. Basic Setup on All VMs

Run these commands on master and all worker nodes.

```bash
apt update -y
apt install -y apt-transport-https ca-certificates curl gpg containerd
```

Disable swap:

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

Load required kernel modules:

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
```

Apply Kubernetes networking settings:

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF

sysctl --system
```

---

## 2. Configure containerd on All VMs

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
```

---

## 3. Install Kubernetes Packages on All VMs

```bash
mkdir -p -m 755 /etc/apt/keyrings

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
tee /etc/apt/sources.list.d/kubernetes.list
```

Install kubeadm, kubelet, and kubectl:

```bash
apt update -y
apt install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl
systemctl enable kubelet
```

---

## 4. Initialize Kubernetes Master Node

Run only on the master node.

```bash
kubeadm init --apiserver-advertise-address=192.168.0.2 --pod-network-cidr=192.168.0.0/16
```

Configure kubectl:

```bash
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown root:root /root/.kube/config
```

Install Calico network plugin:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
```

---

## 5. Join Worker Nodes

After running `kubeadm init`, Kubernetes provides a join command.

Run the join command on all worker nodes.

Example:

```bash
kubeadm join 192.168.0.2:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

If the join command is lost, generate it again on the master node:

```bash
kubeadm token create --print-join-command
```

---

## 6. Verify Cluster

Run on the master node:

```bash
kubectl get nodes -o wide
kubectl get pods -A
```

Expected result:

```text
master    Ready
worker1   Ready
worker2   Ready
worker3   Ready
```

---

# Application Deployment

## 7. Deploy Nginx Web Application

```bash
kubectl apply -f nginx-website.yaml
```

Nginx was deployed with:

* Custom landing page
* ConfigMap
* 6 replicas
* NodePort service

Scale Nginx:

```bash
kubectl scale deployment nginx-app --replicas=6
```

---

## 8. Deploy Apache Web Application

```bash
kubectl apply -f apache-website.yaml
```

Apache was deployed with:

* Custom HTML page
* Deployment
* NodePort service

---

## 9. Deploy Node.js Backend API

```bash
kubectl apply -f nodejs-api.yaml
```

Node.js was deployed as a backend API service with multiple replicas.

---

## 10. Deploy MySQL Database

```bash
kubectl apply -f mysql-clean.yaml
```

MySQL uses:

* Secret for root password
* ClusterIP service
* Internal database access

Test MySQL:

```bash
kubectl exec -it deployment/mysql-db -- mysql -uroot -proot123
```

Inside MySQL:

```sql
SHOW DATABASES;
USE kubernetesdb;
CREATE TABLE students (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(50),
  project VARCHAR(100)
);
INSERT INTO students (name, project)
VALUES ('Chand', 'Kubernetes Cluster Project');
SELECT * FROM students;
```

---

## 11. Deploy Redis Cache

```bash
kubectl apply -f redis.yaml
```

Test Redis:

```bash
kubectl exec -it deployment/redis-cache -- redis-cli
```

Inside Redis:

```bash
PING
SET project "Kubernetes Advanced Cluster"
GET project
EXIT
```

---

# Monitoring Setup

## 12. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

## 13. Install Helm

```bash
apt update -y
apt install -y snapd
snap install helm --classic
```

Check Helm:

```bash
helm version
```

## 14. Add Prometheus Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 15. Install Prometheus and Grafana

```bash
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```

Check monitoring pods:

```bash
kubectl get pods -n monitoring
```

## 16. Expose Grafana

```bash
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
kubectl get svc -n monitoring
```

Get Grafana admin password:

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

Login details:

```text
Username: admin
Password: generated-password
```

Open Grafana using any worker IP and Grafana NodePort:

```text
http://192.168.0.3:<GRAFANA_NODEPORT>
```

---

# NAT and Networking Note

For this project, SNAT was configured for the full private subnet:

```text
192.168.0.0/24 -> Public IP
```

This allowed all worker nodes to access the internet and pull container images.

DNAT was used only for inbound access such as SSH, Nginx, Apache, and Grafana.

---

# Useful Kubernetes Commands

```bash
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get pods -A
kubectl get svc -A
kubectl get deployments
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl scale deployment nginx-app --replicas=6
kubectl delete pod <pod-name>
```

---

# Project Outcome

This setup successfully demonstrates:

* Multi-node Kubernetes cluster
* Application deployment
* Scaling
* Self-healing
* Load balancing
* Database deployment
* Redis caching
* Monitoring with Prometheus
* Visualization using Grafana

```
```
