# Kubernetes Multi-Application Cluster Setup Guide

## Project Architecture

This project uses a 4-node Kubernetes cluster created using Ubuntu virtual machines.

| Node          | IP Address  |     Role                |
|---            |---          |---                      |
| Master Node   | 192.168.0.2 | Kubernetes Control Plane|
| Worker Node 1 | 192.168.0.3 | Application Worker Node |
| Worker Node 2 | 192.168.0.4 | Application Worker Node |
| Worker Node 3 | 192.168.0.5 | Application Worker Node |

The master node is responsible for managing the Kubernetes cluster. Worker nodes are responsible for running application workloads such as Nginx, Apache, Node.js, MySQL, Redis, Prometheus, and Grafana.

---

## Architecture Flow

```text
                         Internet / External User
                                  |
                              Public IP
                                  |
                             NAT / Firewall
                                  |
                         Kubernetes Cluster
                                  |
              -----------------------------------------
              |                                       |
        Master Node                              Worker Nodes
       192.168.0.2                 192.168.0.3 / 192.168.0.4 / 192.168.0.5
              |                                       |
      Control Plane Components              Application Pods and Services
              |                                       |
 kube-apiserver, scheduler, etcd,            Nginx, Apache, Node.js,
 controller-manager                          MySQL, Redis, Grafana
```

---

## Components Used

| Component | Purpose |
|---|---|
| Ubuntu Server | Operating system for all VMs |
| kubeadm | Tool used to initialize and join Kubernetes nodes |
| kubelet | Kubernetes node agent running on every node |
| kubectl | Command-line tool used to manage the cluster |
| containerd | Container runtime used by Kubernetes |
| Calico | Container Network Interface used for pod networking |
| Nginx | Web application |
| Apache | Web application |
| Node.js | Backend API |
| MySQL | Database service |
| Redis | Cache service |
| Prometheus | Monitoring tool |
| Grafana | Visualization dashboard |
| Helm | Package manager used to install Prometheus and Grafana |

---

# Step-by-Step Setup

## 1. Configure Hostnames

Assign meaningful hostnames to each VM. This helps identify nodes clearly in Kubernetes commands.

### Master Node

```bash
hostnamectl set-hostname master
```

### Worker Node 1

```bash
hostnamectl set-hostname worker1
```

### Worker Node 2

```bash
hostnamectl set-hostname worker2
```

### Worker Node 3

```bash
hostnamectl set-hostname worker3
```

Verify hostname:

```bash
hostname
```

---

## 2. Basic Package Setup on All VMs

Run these commands on the master and all worker nodes.

```bash
apt update -y
apt install -y apt-transport-https ca-certificates curl gpg containerd
```

### Purpose

- `apt-transport-https` allows APT to use HTTPS repositories.
- `ca-certificates` verifies trusted SSL certificates.
- `curl` downloads Kubernetes repository keys and files.
- `gpg` verifies repository signing keys.
- `containerd` is the container runtime used by Kubernetes to run containers.

---

## 3. Disable Swap on All VMs

Kubernetes requires swap to be disabled because kubelet expects predictable memory management.

```bash
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
```

Verify swap:

```bash
free -h
```

Swap should show `0`.

---

## 4. Enable Required Kernel Modules

Kubernetes networking requires `overlay` and `br_netfilter` modules.

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
modprobe overlay
modprobe br_netfilter
```

### Purpose

- `overlay` is required for container filesystem layering.
- `br_netfilter` allows bridged network traffic to pass through iptables rules.

---

## 5. Configure Kernel Networking Parameters

```bash
cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
EOF
```

Apply changes:

```bash
sysctl --system
```

### Purpose

These settings allow Kubernetes pods and services to communicate properly across nodes.

---

## 6. Configure containerd on All VMs

```bash
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml
```

Enable systemd cgroup driver:

```bash
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

Restart and enable containerd:

```bash
systemctl restart containerd
systemctl enable containerd
```

Check status:

```bash
systemctl status containerd --no-pager
```

### Purpose

Kubernetes uses containerd to pull images and run containers inside pods.

---

## 7. Install Kubernetes Packages on All VMs

Create Kubernetes keyring directory:

```bash
mkdir -p -m 755 /etc/apt/keyrings
```

Add Kubernetes repository key:

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

Add Kubernetes repository:

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
tee /etc/apt/sources.list.d/kubernetes.list
```

Install Kubernetes components:

```bash
apt update -y
apt install -y kubelet kubeadm kubectl
```

Hold package versions:

```bash
apt-mark hold kubelet kubeadm kubectl
```

Enable kubelet:

```bash
systemctl enable kubelet
```

Check versions:

```bash
kubeadm version
kubelet --version
kubectl version --client
```

### Purpose

- `kubeadm` initializes the cluster and joins worker nodes.
- `kubelet` runs on every node and manages pods.
- `kubectl` is used to manage the Kubernetes cluster.

---

## 8. Initialize the Master Node

Run this command only on the master node.

```bash
kubeadm init --apiserver-advertise-address=192.168.0.2 --pod-network-cidr=192.168.0.0/16
```

### Purpose

This command initializes the Kubernetes control plane on the master node.

The control plane includes:

- kube-apiserver
- etcd
- kube-scheduler
- kube-controller-manager

---

## 9. Configure kubectl on Master Node

```bash
mkdir -p /root/.kube
cp -i /etc/kubernetes/admin.conf /root/.kube/config
chown root:root /root/.kube/config
```

### Purpose

This allows the root user on the master node to run `kubectl` commands.

---

## 10. Install Calico Network Plugin

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.2/manifests/calico.yaml
```

### Purpose

Calico provides pod-to-pod networking across all Kubernetes nodes.

Without a network plugin, worker nodes may join the cluster but pods will not communicate properly.

---

## 11. Join Worker Nodes to the Cluster

After `kubeadm init`, a join command is generated.

Run that command on every worker node.

Example:

```bash
kubeadm join 192.168.0.2:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

If the join command is lost, run this on the master node:

```bash
kubeadm token create --print-join-command
```

Then copy the output and run it on Worker Node 1, Worker Node 2, and Worker Node 3.

---

## 12. Verify Cluster Status

Run on the master node:

```bash
kubectl get nodes
```

```bash
kubectl get nodes -o wide
```

Expected output:

```text
master    Ready
worker1   Ready
worker2   Ready
worker3   Ready
```

Check system pods:

```bash
kubectl get pods -A
```

---

# Application Deployment

## 13. Deploy Nginx Web Application

```bash
kubectl apply -f nginx-website.yaml
```

Scale Nginx deployment:

```bash
kubectl scale deployment nginx-app --replicas=6
```

Check Nginx pods:

```bash
kubectl get pods -o wide
```

Check service:

```bash
kubectl get svc
```

### Purpose

Nginx demonstrates a scalable web application running with multiple replicas across worker nodes.

---

## 14. Deploy Apache Web Application

```bash
kubectl apply -f apache-website.yaml
```

Check Apache pods:

```bash
kubectl get pods -o wide
```

Check Apache service:

```bash
kubectl get svc
```

### Purpose

Apache demonstrates hosting another web application in the same Kubernetes cluster.

---

## 15. Deploy Node.js Backend API

```bash
kubectl apply -f nodejs-api.yaml
```

Check Node.js API pods:

```bash
kubectl get pods -o wide
```

Access Node.js API using NodePort:

```text
http://<Worker-Node-IP>:<NodePort>
```

### Purpose

Node.js demonstrates backend API deployment inside Kubernetes.

---

## 16. Deploy MySQL Database

```bash
kubectl apply -f mysql-clean.yaml
```

Check MySQL pod and service:

```bash
kubectl get pods -o wide
kubectl get svc
```

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
EXIT;
```

### Purpose

MySQL demonstrates database deployment and internal cluster communication using a ClusterIP service.

---

## 17. Deploy Redis Cache

```bash
kubectl apply -f redis.yaml
```

Check Redis pod:

```bash
kubectl get pods -o wide
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

### Purpose

Redis demonstrates in-memory caching inside the Kubernetes cluster.

---

# Monitoring Setup

## 18. Create Monitoring Namespace

```bash
kubectl create namespace monitoring
```

### Purpose

A separate namespace keeps monitoring components organized.

---

## 19. Install Helm

```bash
apt update -y
apt install -y snapd
snap install helm --classic
```

Check Helm:

```bash
helm version
```

### Purpose

Helm is a Kubernetes package manager. It is used here to install Prometheus and Grafana easily.

---

## 20. Add Prometheus Community Helm Repository

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 21. Install Prometheus and Grafana

```bash
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring
```

Check monitoring pods:

```bash
kubectl get pods -n monitoring
```

Expected pods include:

- Prometheus
- Grafana
- Alertmanager
- Node Exporter
- Kube State Metrics

---

## 22. Expose Grafana Using NodePort

```bash
kubectl patch svc monitoring-grafana -n monitoring -p '{"spec":{"type":"NodePort"}}'
```

Check Grafana service:

```bash
kubectl get svc -n monitoring
```

Access Grafana:

```text
http://<Worker-Node-IP>:<Grafana-NodePort>
```

Example:

```text
http://192.168.0.3:30451
```

---

## 23. Get Grafana Login Password

```bash
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

Grafana username:

```text
admin
```

Use the generated password to log in.

---

## 24. View Monitoring Dashboards

After logging into Grafana:

```text
Dashboards → Browse
```

Open dashboards such as:

```text
Kubernetes / Compute Resources / Cluster
Kubernetes / Compute Resources / Node
Kubernetes / Compute Resources / Pod
CoreDNS
Node Exporter / Nodes
```

---

# NAT and Networking Configuration

For all worker nodes to access the internet, SNAT should be configured for the full subnet:

```text
192.168.0.0/24 → Public IP
```

### Purpose

This allows all worker nodes to pull container images from Docker Hub and other registries.

DNAT is used only for inbound access such as:

- SSH
- Nginx NodePort
- Apache NodePort
- Node.js API NodePort
- Grafana NodePort

---

# Troubleshooting Notes

## APT Lock Issue

If APT is locked by unattended upgrades:

```bash
systemctl stop unattended-upgrades
systemctl stop apt-daily.service
systemctl stop apt-daily-upgrade.service
dpkg --configure -a
apt --fix-broken install -y
```

---

## DNS Issue During Image Pull

If pods show `ImagePullBackOff` due to DNS timeout:

```bash
rm -f /etc/resolv.conf

cat <<EOF > /etc/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

systemctl restart containerd
systemctl restart kubelet
```

Then delete the failed pod:

```bash
kubectl delete pod <pod-name>
```

Kubernetes will recreate it automatically.

---

# Useful Kubernetes Commands

```bash
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get pods -A
kubectl get svc
kubectl get svc -A
kubectl get deployments
kubectl get namespaces
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl scale deployment nginx-app --replicas=6
kubectl delete pod <pod-name>
```

---

# Final Verification

Run these commands to verify the complete project:

```bash
kubectl get nodes -o wide
kubectl get pods -o wide
kubectl get svc -A
kubectl get pods -n monitoring
```

Successful completion means:

- All nodes are in `Ready` state.
- Application pods are in `Running` state.
- Services are created successfully.
- Grafana dashboard is accessible.
- Prometheus is collecting metrics.

---

# Project Outcome

This setup successfully demonstrates:

- Multi-node Kubernetes cluster deployment
- Container orchestration using Kubernetes
- Application deployment using Deployments
- Load balancing using Services
- Scaling and self-healing
- Internal service communication
- Database deployment using MySQL
- Caching using Redis
- Monitoring using Prometheus
- Visualization using Grafana
- Practical DevOps and cloud infrastructure implementation
