# Complete Guide to Deploying MySQL with Helm in Kubernetes

## Table of Contents
- [Environment Preparation](#environment-preparation)
- [Kubernetes Cluster Deployment](#kubernetes-cluster-deployment)
- [Helm Installation and Configuration](#helm-installation-and-configuration)
- [Deploy MySQL Using Helm Chart](#deploy-mysql-using-helm-chart)
- [Verify Deployment and Connection Testing](#verify-deployment-and-connection-testing)
- [Basic Helm Operation Guide](#basic-helm-operation-guide)


## Environment Preparation

### Hardware Requirements
- Virtual Machines: 2 (1 master + 1 node, or a single node serving as both master and node)
- Operating System: CentOS 7.9 (64-bit, minimal installation)
- Resource Configuration: Each machine requires at least 2 CPU cores, 4GB memory, and 20GB disk space
- Network Settings: VMware uses "NAT Mode"; configure a static IP to ensure network connectivity

### Software Dependencies
- Docker (container runtime)
- kubeadm, kubelet, kubectl (K8s core components)
- Helm (package management tool)
- Network Plugin (flannel)


## Kubernetes Cluster Deployment

### 1. Download CentOS Image
```bash
# Image download address
https://vault.centos.org/7.9.2009/isos/x86_64/  # Select CentOS-7-x86_64-Minimal-2009.iso
```

### 2. Create Virtual Machines (VMware)
1. Create a new virtual machine → Typical configuration → Select the downloaded ISO image
2. Select operating system as "Linux → CentOS 7 64-bit"
3. Virtual machine name: `k8s-master` (name the node machine `k8s-node1`)
4. Disk size: 20GB, check "Store virtual disk as a single file"
5. Customize hardware: CPU ≥ 2 cores, memory ≥ 4GB, select "NAT Mode" for network adapter
6. Start the virtual machine and install CentOS (set root password, e.g., `123456`)

### 3. Configure Static IP (All Nodes)
```bash
# Check network card name (usually ens33)
ip addr

# Edit network configuration file
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

Modify the configuration (adjust according to actual network):
```ini
BOOTPROTO=static          # Static IP
ONBOOT=yes                # Enable on boot
IPADDR=192.168.159.130    # Master node IP (example)
NETMASK=255.255.255.0     # Subnet mask
GATEWAY=192.168.159.2     # Gateway (check in VMware Virtual Network Editor)
DNS1=114.114.114.114      # DNS server
```

```bash
# Restart network service
systemctl restart network

# Verify network connectivity
ping baidu.com  # Normal if ping succeeds
```

### 4. Basic Environment Configuration (All Nodes)

#### (1) Disable Firewall, SELinux, and Swap
```bash
# Disable firewall
systemctl stop firewalld
systemctl disable firewalld

# Disable SELinux (temporary + permanent)
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# Disable Swap (temporary + permanent)
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab  # Comment out the swap line
```

#### (2) Configure Hostname and Hosts File
```bash
# Set hostname for master node
hostnamectl set-hostname k8s-master

# Set hostname for node (execute on node machine)
hostnamectl set-hostname k8s-node1

# Edit hosts file (add to all nodes)
cat >> /etc/hosts << EOF
192.168.159.130 k8s-master
192.168.159.131 k8s-node1  # Replace with actual node IP
EOF
```

#### (3) Configure Time Synchronization
```bash
# Install chrony
yum install -y chrony

# Start and enable chrony on boot
systemctl start chronyd
systemctl enable chronyd

# Synchronize time
chronyc sources
```

#### (4) Configure Kernel Parameters (Required by K8s)
```bash
# Add kernel modules
cat >> /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

# Load modules
modprobe overlay
modprobe br_netfilter

# Configure kernel parameters
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply configuration
sysctl --system
```

### 5. Install Docker (All Nodes)
```bash
# Install dependencies
yum install -y yum-utils device-mapper-persistent-data lvm2

# Add Docker repository (Alibaba Cloud)
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# Install Docker (specify version to avoid compatibility issues)
yum install -y docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io

# Start Docker and enable on boot
systemctl start docker
systemctl enable docker

# Configure Docker image accelerator (Alibaba Cloud, to improve pull speed)
mkdir -p /etc/docker
cat >> /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF

# Restart Docker
systemctl daemon-reload
systemctl restart docker
```

# Complete Guide to Deploying MySQL with Helm in Kubernetes (Continued)

### 6. Deploy Kubernetes Cluster

#### (1) Install kubeadm, kubelet, and kubectl (All Nodes)
```bash
# Add Kubernetes repository (Alibaba Cloud)
cat >> /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# Install components (specify version for Docker compatibility)
yum install -y kubeadm-1.23.6 kubelet-1.23.6 kubectl-1.23.6

# Start kubelet and enable it on boot
systemctl start kubelet
systemctl enable kubelet
```

#### (2) Initialize Master Node (Execute Only on Master)
```bash
# Initialize cluster (use Alibaba Cloud image repository to avoid pull failures in China)
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.6 \
  --pod-network-cidr=10.244.0.0/16  # Network segment required for flannel

# Configure kubectl (for regular user permissions)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Verify master status (node will be "NotReady" until network plugin is installed)
kubectl get nodes
```

#### (3) Install Network Plugin (flannel, Execute Only on Master)
```bash
# Download flannel configuration file (domestically accessible URL)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.17.0/Documentation/kube-flannel.yml

# Wait 3-5 minutes and verify node status (should change to "Ready")
kubectl get nodes
```

#### (4) Add Node to Cluster (Execute Only on Node)
```bash
# Generate join command on master node (if initial command is forgotten)
kubeadm token create --print-join-command

# Example output (replace with actual generated command)
kubeadm join 192.168.159.130:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# Verify cluster nodes on master node
kubectl get nodes  # Cluster deployment succeeds if all nodes show "Ready"
```


## Helm Installation and Configuration

### 1. Download and Install Helm
```bash
# Download Helm 3.8.0 (compatible with Kubernetes 1.23)
wget https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz

# Extract and install
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm

# Verify installation
helm version  # Installation succeeds if version information is displayed
```

### 2. Helm Repository Management
```bash
# Add Bitnami repository (contains common applications like MySQL)
helm repo add bitnami https://charts.bitnami.com/bitnami

# List added repositories
helm repo list

# Update repository index (to get latest Charts)
helm repo update

# Remove repository (if needed)
helm repo remove bitnami
```


## Deploy MySQL Using Helm Chart

### 1. Search and Get MySQL Chart Information
```bash
# Search for MySQL-related Charts
helm search repo mysql

# Download Chart to local for detailed inspection (optional)
helm pull bitnami/mysql --untar
ls mysql/  # View Chart directory structure (templates/, values.yaml, etc.)
```

### 2. Customize MySQL Configuration
Create a `mysql-values.yaml` file to override default configurations:
```yaml
# mysql-values.yaml
# Root user password (use a strong password in production)
auth:
  rootPassword: "MyRoot123!"

# Primary node configuration
primary:
  # Persistent storage settings
  persistence:
    size: "10Gi"  # Storage capacity

# Service exposure configuration
service:
  type: NodePort  # Expose as NodePort for external access
  nodePort: 30006  # Node port (range: 30000-32767)
```

### 3. Deploy MySQL
```bash
# Create namespace (for resource isolation)
kubectl create namespace mysql-demo

# Deploy MySQL with custom configuration
helm install my-mysql bitnami/mysql \
  --namespace mysql-demo \
  -f mysql-values.yaml
```

### 4. Check Deployment Status
```bash
# List Helm-deployed Releases
helm list -n mysql-demo

# Check MySQL Pod status
kubectl get pods -n mysql-demo -l app.kubernetes.io/name=mysql

# Check MySQL Service (to get access port)
kubectl get svc my-mysql -n mysql-demo

# Check persistent storage (PVC)
kubectl get pvc -n mysql-demo
```


## Verify Deployment and Connection Testing

### 1. Connect to MySQL Container
```bash
# Enter MySQL Pod
kubectl exec -it -n mysql-demo my-mysql-0 -- bash

# Log in to MySQL (use root password from configuration file)
mysql -u root -p
# Enter password: MyRoot123!
```

### 2. External Access Test (Optional)
```bash
# Install MySQL client on master node
yum install -y mysql

# Get Service's ClusterIP (or use node IP + NodePort)
MYSQL_IP=$(kubectl get svc my-mysql -n mysql-demo -o jsonpath='{.spec.clusterIP}')

# Connect to MySQL
mysql -h $MYSQL_IP -P 3306 -u root -p
# Enter password "MyRoot123!"; deployment is normal if connection succeeds
```


## Basic Helm Operation Guide

| Operation | Command Example | Description |
|-----------|-----------------|-------------|
| Install Application | `helm install <release-name> <repo/chart-name>` | Deploy application based on Chart |
| List Deployments | `helm list -n <namespace>` | View Releases in specified namespace |
| Update Configuration | `helm upgrade <release-name> <repo/chart-name> -f <values.yaml>` | Upgrade application configuration |
| Rollback Version | `helm rollback <release-name> <version-number>` | Roll back to specified version |
| Uninstall Application | `helm uninstall <release-name> -n <namespace>` | Remove application and related resources |
| View Chart History Versions | `helm search repo <chart-name> --versions` | View all versions of a Chart |


## Summary
Core advantages of deploying MySQL via Helm Chart:
- **Simplified Deployment**: No need to manually write large amounts of YAML files; deploy quickly using predefined templates.
- **Version Management**: Supports upgrades and rollbacks for easy application lifecycle management.
- **Flexible Configuration**: Customize parameters via `values.yaml` to adapt to different environment requirements.

This workflow is applicable to the deployment of most mainstream applications in Kubernetes environments. Adjust configurations (such as resource limits, storage classes, and network policies) according to actual production needs to ensure stable operation of applications.部署，是 Kubernetes 生态中标准化、高效的应用管理方式。
