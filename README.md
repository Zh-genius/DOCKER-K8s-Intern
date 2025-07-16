# 在 Kubernetes 中使用 Helm 部署 MySQL 完整指南

## 目录
- [环境准备](#环境准备)
- [Kubernetes 集群部署](#kubernetes-集群部署)
- [Helm 安装与配置](#helm-安装与配置)
- [使用 Helm Chart 部署 MySQL](#使用-helm-chart-部署-mysql)
- [验证部署与连接测试](#验证部署与连接测试)
- [Helm 基本操作指南](#helm-基本操作指南)


## 环境准备

### 硬件要求
- 虚拟机：2 台（1 台 master + 1 台 node，或单节点兼作 master 和 node）
- 操作系统：CentOS 7.9（64 位，最小化安装）
- 资源配置：每台至少 2 核 CPU、4GB 内存、20GB 磁盘
- 网络设置：VMware 采用「NAT 模式」，需配置静态 IP 确保联网

### 软件依赖
- Docker（容器运行时）
- kubeadm、kubelet、kubectl（K8s 核心组件）
- Helm（包管理工具）
- 网络插件（flannel）


## Kubernetes 集群部署

### 1. 下载 CentOS 镜像
```bash
# 镜像下载地址
https://vault.centos.org/7.9.2009/isos/x86_64/  # 选择 CentOS-7-x86_64-Minimal-2009.iso
```

### 2. 创建虚拟机（VMware）
1. 新建虚拟机 → 典型配置 → 选择下载的 ISO 镜像
2. 操作系统选择「Linux → CentOS 7 64 位」
3. 虚拟机名称：`k8s-master`（node 节点命名为 `k8s-node1`）
4. 磁盘大小：20GB，勾选「将磁盘存储为单个文件」
5. 自定义硬件：CPU≥2 核，内存≥4GB，网络适配器选择「NAT 模式」
6. 启动虚拟机并安装 CentOS（设置 root 密码，例如 `123456`）

### 3. 配置静态 IP（所有节点）
```bash
# 查看网卡名称（通常为 ens33）
ip addr

# 编辑网络配置文件
vi /etc/sysconfig/network-scripts/ifcfg-ens33
```

修改配置（按实际网络调整）：
```ini
BOOTPROTO=static          # 静态IP
ONBOOT=yes                # 开机启动
IPADDR=192.168.159.130    # master节点IP（示例）
NETMASK=255.255.255.0     # 子网掩码
GATEWAY=192.168.159.2     # 网关（在VMware虚拟网络编辑器中查看）
DNS1=114.114.114.114      # DNS服务器
```

```bash
# 重启网络
systemctl restart network

# 验证联网
ping baidu.com  # 能通则正常
```

### 4. 基础环境配置（所有节点）

#### （1）关闭防火墙、SELinux、Swap
```bash
# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 关闭SELinux（临时+永久）
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 关闭Swap（临时+永久）
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab  # 注释swap行
```

#### （2）配置主机名和 hosts
```bash
# master节点设置主机名
hostnamectl set-hostname k8s-master

# node节点设置主机名（在node节点执行）
hostnamectl set-hostname k8s-node1

# 编辑hosts文件（所有节点都添加）
cat >> /etc/hosts << EOF
192.168.159.130 k8s-master
192.168.159.131 k8s-node1  # 替换为node节点实际IP
EOF
```

#### （3）配置时间同步
```bash
# 安装chrony
yum install -y chrony

# 启动并设置开机自启
systemctl start chronyd
systemctl enable chronyd

# 同步时间
chronyc sources
```

#### （4）配置内核参数（K8s 要求）
```bash
# 添加内核模块
cat >> /etc/modules-load.d/k8s.conf << EOF
overlay
br_netfilter
EOF

# 加载模块
modprobe overlay
modprobe br_netfilter

# 配置内核参数
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 生效配置
sysctl --system
```

### 5. 安装 Docker（所有节点）
```bash
# 安装依赖
yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加Docker源（阿里云）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装Docker（指定版本，避免兼容性问题）
yum install -y docker-ce-19.03.15 docker-ce-cli-19.03.15 containerd.io

# 启动Docker并设置开机自启
systemctl start docker
systemctl enable docker

# 配置Docker镜像加速器（阿里云，提升拉取速度）
mkdir -p /etc/docker
cat >> /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF

# 重启Docker
systemctl daemon-reload
systemctl restart docker
```

### 6. 部署 K8s 集群

#### （1）安装 kubeadm、kubelet、kubectl（所有节点）
```bash
# 添加K8s源（阿里云）
cat >> /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装组件（指定版本，与Docker兼容）
yum install -y kubeadm-1.23.6 kubelet-1.23.6 kubectl-1.23.6

# 启动kubelet并设置开机自启
systemctl start kubelet
systemctl enable kubelet
```

#### （2）初始化 Master 节点（仅在 master 执行）
```bash
# 初始化集群（指定阿里云镜像仓库，避免墙内拉取失败）
kubeadm init \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.23.6 \
  --pod-network-cidr=10.244.0.0/16  # flannel网络需要的网段

# 配置kubectl（普通用户权限）
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 验证master状态（此时节点为NotReady，因未安装网络插件）
kubectl get nodes
```

#### （3）安装网络插件（flannel，仅在 master 执行）
```bash
# 下载flannel配置文件（国内可访问地址）
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/v0.17.0/Documentation/kube-flannel.yml

# 等待3-5分钟，验证节点状态（变为Ready）
kubectl get nodes
```

#### （4）添加 Node 节点到集群（仅在 node 执行）
```bash
# 在master节点生成join命令（如果忘记初始命令）
kubeadm token create --print-join-command

# 示例输出（替换为实际生成的命令）
kubeadm join 192.168.159.130:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# 在master节点验证集群节点
kubectl get nodes  # 所有节点状态为Ready则成功
```


## Helm 安装与配置

### 1. 下载并安装 Helm
```bash
# 下载Helm 3.8.0（适合K8s 1.23）
wget https://get.helm.sh/helm-v3.8.0-linux-amd64.tar.gz

# 解压并安装
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm

# 验证安装
helm version  # 输出版本信息则成功
```

### 2. Helm 仓库管理
```bash
# 添加bitnami仓库（包含MySQL等常用应用）
helm repo add bitnami https://charts.bitnami.com/bitnami

# 查看已添加的仓库
helm repo list

# 更新仓库索引（获取最新Chart）
helm repo update

# 移除仓库（如需）
helm repo remove bitnami
```


## 使用 Helm Chart 部署 MySQL

### 1. 搜索并获取 MySQL Chart 信息
```bash
# 搜索MySQL相关Chart
helm search repo mysql

# 下载Chart到本地查看详情（可选）
helm pull bitnami/mysql --untar
ls mysql/  # 查看Chart目录结构（templates/ values.yaml等）
```

### 2. 自定义 MySQL 配置
创建 `mysql-values.yaml` 文件，覆盖默认配置：
```yaml
# mysql-values.yaml
# root用户密码（生产环境使用强密码）
auth:
  rootPassword: "MyRoot123!"

# 主节点配置
primary:
  # 持久化存储设置
  persistence:
    size: "10Gi"  # 存储大小

# 服务暴露配置
service:
  type: NodePort  # 暴露为NodePort，允许外部访问
  nodePort: 30006  # 节点端口（范围30000-32767）
```

### 3. 部署 MySQL
```bash
# 创建命名空间（隔离资源）
kubectl create namespace mysql-demo

# 使用自定义配置部署MySQL
helm install my-mysql bitnami/mysql \
  --namespace mysql-demo \
  -f mysql-values.yaml
```

### 4. 查看部署状态
```bash
# 查看Helm部署的Release
helm list -n mysql-demo

# 查看MySQL Pod状态
kubectl get pods -n mysql-demo -l app.kubernetes.io/name=mysql

# 查看MySQL Service（获取访问端口）
kubectl get svc my-mysql -n mysql-demo

# 查看持久化存储（PVC）
kubectl get pvc -n mysql-demo
```


## 验证部署与连接测试

### 1. 连接 MySQL 容器
```bash
# 进入MySQL Pod
kubectl exec -it -n mysql-demo my-mysql-0 -- bash

# 登录MySQL（使用配置文件中的root密码）
mysql -u root -p
# 输入密码：MyRoot123!
```

### 2. 外部访问测试（可选）
```bash
# 在master节点安装mysql客户端
yum install -y mysql

# 获取Service的ClusterIP（或使用节点IP+NodePort）
MYSQL_IP=$(kubectl get svc my-mysql -n mysql-demo -o jsonpath='{.spec.clusterIP}')

# 连接MySQL
mysql -h $MYSQL_IP -P 3306 -u root -p
# 输入密码MyRoot123!，成功连接则部署正常
```


## Helm 基本操作指南

| 操作                 | 命令示例                                  | 说明                     |
|----------------------|-------------------------------------------|--------------------------|
| 安装应用             | `helm install <release名> <仓库/chart名>`  | 基于Chart部署应用        |
| 查看部署             | `helm list -n <命名空间>`                 | 查看指定命名空间的Release |
| 更新配置             | `helm upgrade <release名> <仓库/chart名> -f <values.yaml>` | 升级应用配置             |
| 回滚版本             | `helm rollback <release名> <版本号>`      | 回滚到指定版本           |
| 卸载应用             | `helm uninstall <release名> -n <命名空间>` | 移除应用及相关资源       |
| 查看Chart历史版本    | `helm search repo <chart名> --versions`   | 查看Chart的所有版本      |


## 总结
通过 Helm Chart 部署 MySQL 的核心优势：
- **简化部署**：无需手动编写大量 YAML 文件，基于预定义模板快速部署
- **版本管理**：支持升级、回滚，轻松管理应用生命周期
- **配置灵活**：通过 `values.yaml` 自定义参数，适应不同环境需求

此流程适用于大多数主流应用的部署，是 Kubernetes 生态中标准化、高效的应用管理方式。
