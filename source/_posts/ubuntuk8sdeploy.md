---
title: 基于 Ubuntu + Containerd 部署 Kubernetes 集群
categories:
  - CNCF
tags:
  - Kubernetes
abbrlink: 28d6
date: 2022-06-07 22:17:15
---
### 一、环境准备
#### 1. multipass  虚拟机创建
```shell
# 生成密钥对
ssh-keygen -t rsa -b 4096 -f ~/k8s_rsa -C k8s

# 创建 Master 节点
multipass launch -c 2 -m 2G -d 20G -n master --cloud-init - << EOF
ssh_authorized_keys:
- $(cat ~/.ssh/k8s_rsa.pub)
EOF

# 创建 Node 节点
multipass launch -c 1 -m 2G -d 20G -n node1 --cloud-init - << EOF
ssh_authorized_keys:
- $(cat ~/.ssh/k8s_rsa.pub)
EOF

multipass launch -c 1 -m 2G -d 20G -n node2 --cloud-init - << EOF
ssh_authorized_keys:
- $(cat ~/.ssh/k8s_rsa.pub)
EOF
```
> --cloud-init ：导入本地生成的公钥文件到初始化系统中，可以使用密钥免密 SSH
<!--more-->


#### 2. 主机与网络规划
| **主机 IP** | **主机名** | **主机配置** | **节点角色** |
| --- | --- | --- | --- |
| 192.168.64.4 | master1 | 2C/2G | master 节点 |
| 192.168.64.5 | node1 | 1C/1G | node 节点 |
| 192.168.64.6 | node2 | 1C/1G | node 节点 |

| **子网 Subnet** | **CIDR 网段** |
| --- | --- |
| nodeSubnet | 192.168.64.0/24 |
| PodSubnet | 172.16.0.0/16 |
| ServiceSubnet | 10.10.0.0/16 |


#### 3.软件版本
| **软件** | **版本** |
| --- | --- |
| 操作系统 | Ubuntu 20.04.4 LTS |
| 内核版本 | 5.4.0-109-generic |
| containerd | 1.5.10-1 |
| kubernetes | v1.23.2 |
| kubeadm | v1.23.2 |
| kube-apiserver | v1.23.2 |
| kube-controller-manager | v1.23.2 |
| kube-scheduler | v1.23.2 |
| kubectl | v1.23.2 |
| kubelet | v1.23.2 |
| kube-proxy |  |
| etcd | v3.5.1 |
| CNI 插件（calico） | v3.18 |


### 二、集群配置（所有节点执行）
#### 1. 节点初始化

- 主机名与 host 解析
```shell
hostnamectl --static set-hostname master    # master 节点执行
hostnamectl --static set-hostname node1     # node1 节点执行
hostnamectl --static set-hostname node2     # node2 节点执行

sudo tee -a /etc/hosts << EOF
192.168.64.4 master
192.168.64.5 node1
192.168.64.6 node2
EOF
```

- 关闭防火墙与禁用 swap 分区
```shell
sudo ufw disable && sudo systemctl disable ufw

swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```
> k8s集群安装为什么需要关闭 swap 分区？ swap 必须关，否则 kubelet 起不来,进而导致 k8s 集群起不来； 且考虑 kublet 会用 swap 做数据交换的话，对性能影响比较大


- 同步时间与时区
```shell
sudo cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
sudo timedatectl set-timezone Asia/Shanghai

# 将当前的 UTC 时间写入硬件时钟 (硬件时间默认为UTC)
sudo timedatectl set-local-rtc 0
# 启用 NTP 时间同步：
sudo timedatectl set-ntp yes

# 校准时间服务器-时间同步(推荐使用 chronyc 进行平滑同步)
sudo apt-get install chrony -y
sudo chronyc tracking
# 手动校准-强制更新时间
# chronyc -a makestep
# 系统时钟同步硬件时钟
sudo hwclock -w

# 重启依赖于系统时间的服务
sudo systemctl restart rsyslog.service cron.service
```

- 内核模块加载与配置
```shell
# 1.安装 ipvs
sudo apt-get install ipset ipvsadm -y

# 2.加载内核模块
# 配置重启后永久加载模块
sudo tee /etc/modules-load.d/k8s.conf << EOF
# netfilter
br_netfilter
# containerd.
overlay
# ipvs
ip_vs
ip_vs_rr
ip_vs_wrr
ip_vs_sh
nf_conntrack
EOF
# 临时加载模块
mod_tmp=(br_netfilter overlay ip_vs ip_vs_rr ip_vs_wrr ip_vs_sh nf_conntrack)
for m in ${mod_tmp[@]};do sudo modprobe $m; done
lsmod | egrep "ip_vs|nf_conntrack_ipv4"

# 3.配置内核参数
# 设置 sysctl 必须参数，重启后永久生效
sudo tee /etc/sysctl.d/99-kubernetes-cri.conf << EOF
net.bridge.bridge-nf-call-ipv6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
# 临时应用 sysctl 参数而无需重新启动
sudo sysctl --system
```

- 配置免密登录（master 节点执行，非必须）
```shell
# master 节点执行
ssh-keygen
ssh-copy-id -i ~/.ssh/id_rsa.pub root@node1
ssh-copy-id -i ~/.ssh/id_rsa.pub root@node2
```

#### 2. 容器运行时安装
```shell
# 1.删除旧版本
sudo apt-get remove docker docker-engine docker.io containerd runc
 
# 2.更新 apt 程序包索引并安装程序包，以允许 apt 通过 HTTPS 使用存储库
sudo apt-get update
sudo apt-get install \
  apt-transport-https \
  ca-certificates \
  curl \
  gnupg \
  lsb-release -y

# 3.添加 Docker 的官方 GPG 密钥
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# 4.设置稳定存储库，添加 nightly 或 test 存储库
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
 $(lsb_release -cs) stable nightly" | sudo tee /etc/apt/sources.list.d/container.list
 
# 5.安装 containerd
#更新 apt 包索引，安装最新版本的 containerd 或进入下一步安装特定版本
sudo apt-get update
#查看 containerd.io 可用的版本
apt-cache madison containerd.io
#安装指定版本
sudo apt install containerd.io=1.5.10-1 -y

# 6.配置 containerd
containerd config default | sudo tee /etc/containerd/config.toml
#替换 pause 镜像源
sudo sed -i "s#k8s.gcr.io/pause#registry.cn-hangzhou.aliyuncs.com/google_containers/pause#g"  /etc/containerd/config.toml
# docker.io & gcr.io & k8s.gcr.io & quay.io 镜像加速
sudo tee ~/tmp.txt << EOF
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://taa4w07u.mirror.aliyuncs.com"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."gcr.io"]
          endpoint = ["https://gcr.mirrors.ustc.edu.cn"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."k8s.gcr.io"]
          endpoint = ["https://gcr.mirrors.ustc.edu.cn/google-containers/", "https://registry.aliyuncs.com/google-containers/"]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."quay.io"]
          endpoint = ["https://quay.mirrors.ustc.edu.cn"]
EOF
sudo sed -i '/registry.mirrors\]/r ./tmp.txt' /etc/containerd/config.toml
#使用 SystemdCgroup 驱动程序，节点资源紧张时更稳定
sudo sed -i 's# SystemdCgroup = false# SystemdCgroup = true#g' /etc/containerd/config.toml

# 7.启动 containerd 并验证
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd
#验证
sudo ctr version


```

### 三、构建集群
#### 1.组件安装（所有节点执行）
```shell
# 使用Alicloud加速镜像
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
sudo tee /etc/apt/sources.list.d/kubernetes.list << EOF
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF

# 更新 apt 包索引 & 查看并安装版本
sudo apt-get update
apt-cache madison kubeadm |head
sudo apt install kubeadm=1.23.2-00 kubelet=1.23.2-00 kubectl=1.23.2-00 -y
# 锁定版本
sudo apt-mark hold kubelet kubeadm kubectl
```

```shell
# 配置客户端工具 runtime 与镜像端点配置
sudo crictl config runtime-endpoint /run/containerd/containerd.sock
sudo tee /etc/crictl.yaml << EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# 重载 systemd 守护进程并将 kubelet 设置开机启动
sudo systemctl daemon-reload
sudo systemctl enable --now kubelet
# 查看 kubelet 状态异常，会每隔几秒重启，陷入等待 kubeadm 指令的死循环
systemctl status kubelet 

```

#### 2.初始化主节点
+ Master 节点执行
```shell
# 导出默认初始化配置
kubeadm config print init-defaults > kubeadm.yaml

# 根据本地环境修改初始配置内容
cat > kubeadm.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.64.4 # 修改 master 节点 IP
  bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock # 修改容器运行时为 containerd
  imagePullPolicy: IfNotPresent
  name: master # 修改 master 节点名称
  taints: # master 节点添加污点，不能调度应用
  - effect: "NoSchedule"
    key: "node-role.kubernetes.io/master"
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers # 修改镜像加速地址
kind: ClusterConfiguration
kubernetesVersion: 1.23.0
networking:
  dnsDomain: cluster.local
  podSubnet: 172.16.0.0/16  # 修改 Pod 子网
  serviceSubnet: 10.10.0.0/16 # 修改 Service CIDR 网段
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs # 修改kube-proxy 模式为ipvs，默认为iptables
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd # 配置 cgroup driver
EOF

# 查看初始化集群所需镜像与提前拉取
kubeadm config images list --config kubeadm.yaml
kubeadm config images pull --config kubeadm.yaml

# 初始化 master 节点
sudo kubeadm init --config=kubeadm.yaml

```

+ Node 节点执行
```shell
#初始化完成生成的命令：Node 节点执行，加入集群
sudo kubeadm join 192.168.64.4:6443 --token abcdef.0123456789abcdef \
	--discovery-token-ca-cert-hash sha256:6e25620c2478e38edfe335761b8dd37dbbe0dc8c1df9b41d539b148732d32718
#打印 join token 值
#kubeadm token create --print-join-command

#初始化完成生成的命令：用于 kubectl 命令
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 3.安装 CNI 网络插件（calico）
> calico 插件官方地址: [https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart](https://projectcalico.docs.tigera.io/getting-started/kubernetes/quickstart)

```shell
# calico 官方下载 calico 插件部署清单
wget https://docs.projectcalico.org/v3.18/manifests/calico.yaml
#wget https://docs.projectcalico.org/v3.22/manifests/calico.yaml

# 修改自定义配置
vim calico.yaml
- name: CALICO_IPV4POOL_CIDR
  value: "172.16.0.0/16"
  
# 验证等待 calico 插件 Pod 成功运行
watch kubectl get pod -n kube-system
NAME                                       READY   STATUS    RESTARTS     AGE
calico-kube-controllers-6cfb54c7bb-7xdld   1/1     Running   0            2m51s
calico-node-sjr6r                          1/1     Running   0            2m52s
calico-node-vsczr                          1/1     Running   0            2m51s


```

```shell
# 设置节点角色
kubectl label nodes master node-role.kubernetes.io/control-plane=
kubectl label nodes node1 node-role.kubernetes.io/work=
kubectl label nodes node2 node-role.kubernetes.io/work=
kubectl get nodes

# 自动补齐 kubectl 命令
sudo apt install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

# nerdctl 工具（替代 docker 命令）
#官方地址
# https://github.com/containerd/nerdctl
#下载安装
wget https://github.com/containerd/nerdctl/releases/download/v0.20.0/nerdctl-0.20.0-linux-amd64.tar.gz
tar Cxfz /usr/local/bin/ nerdctl-0.20.0-linux-amd64.tar.gz
#使用方式
sudo nerdctl -n k8s.io images 
sudo nerdctl -n k8s.io ps
sudo nerdctl -n k8s.io images     # 等同于 = sudo ctr -n k8s.io images ls
sudo nerdctl -n k8s.io pull nginx # 等同于 = sudo crictl pull nginx
```

```shell
# flannel 插件重置方式，非适用 calico
kubeadm reset
ifconfig cni0 down && ip link delete cni0
ifconfig flannel.1 down && ip link delete flannel.1
rm -rf /var/lib/cni/
```

#### 4.集群部署验证
```shell
# 部署 Nginx Deployment
kubectl create deployment nginx --image=nginx

# 暴露 Nginx 服务，类型为 NodePort
kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort 

# 访问验证
curl 10.10.225.108:80 -I      # 请求 Service 端口（port，集群内部）
curl 192.168.64.5:31052 -I    # 请求 Node 节点端口（nodePort，可集群外部访问）
curl 172.16.166.132:80 -I     # 请求 Pod 应用内部端口（targetPort，容器的启动端口）
```

#### 5.Kubernetes 组件
**控制平面组件**

- kube-apiserver：多实例伸缩，高可用且可均衡流量？
- etcd：高可用与备份策略？
- kube-scheduler  调度策略：Pod 资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间干扰和最后时限
- kube-controller-manager

**数据平面组件（所有节点）**

- kubelet
- kubeproxy
- 容器运行时（CR）：containerd（Kubernetes 后续版本不使用 docker）

**插件 Addons**

- 网络插件：calico、flannel

**可观测性：日志与监控**

- 日志：fluentd
- 监控：Prometheus

### 四、Kubernetes 仪表板（Dashboard）
#### 1.Kubernetes 原生仪表板
> 官方文档：[https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/](https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/)

```shell
# 1.部署 Dashboard 清单
#wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.1/aio/deploy/recommended.yaml
#kubectl apply -f recommended.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.6.0/aio/deploy/recommended.yaml

# 2.启用 Dashboard 访问
#查看资源是否正常启动
kubectl get pod,service -n kubernetes-dashboard
#修改服务暴露为 nodePort 方式
kubectl edit service kubernetes-dashboard -n kubernetes-dashboard
#===主要配置内容
  ports:
  - nodePort: 30333  # 新增
    port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: NodePort  # 修改
#### 

# 3.默认仪表板部署为最小 RBAC 权限集，需要操作资源时，需要创建 ClusterRole 角色。
# RBAC 参考：https://github.com/kubernetes/dashboard/blob/master/docs/user/access-control/README.md
#登录 Dashboard
kubectl describe secret -n kubernetes-dashboard $(kubectl get secret -n kubernetes-dashboard |grep kubernetes-dashboard-token |awk '{print $1}')
#浏览器访问（使用firefox） https://192.168.64.4:30333
#使用上面获取的 token 值登录（默认 token 只有 kubernetes-dashboard 空间权限）
```

#### 2.K9S 集群管理工具
官方文档：[https://k9scli.io/](https://k9scli.io/)

### 五、参考文档
1、[multipass 官网](https://multipass.run/)
2、[Kubernetes 官方文档](https://kubernetes.io/zh/docs/concepts/overview/components/#container-runtime)
3、[二进制方式安装 Kubernetes 集群](https://blog.weiyigeek.top/2022/5-7-654.html)

