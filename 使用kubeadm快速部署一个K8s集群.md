> #### 作者：李振良
>
> 官方网站：http://www.ctnrs.com

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

```
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master节点的IP和端口 >
```

## 1、安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap分区

## 2、学习目标

1. 在所有节点上安装Docker和kubeadm
2. 部署Kubernetes Master
3. 部署容器网络插件
4. 部署 Kubernetes Node，将节点加入Kubernetes集群中
5. 部署Dashboard Web页面，可视化查看Kubernetes资源

## 3、准备环境

 ![kubernetes](https://blog-1252881505.cos.ap-beijing.myqcloud.com/k8s/single-master.jpg) 

| 角色       | IP           |
| ---------- | ------------ |
| k8s-master | 192.168.0.11 |
| k8s-node1  | 192.168.0.12 |
| k8s-node2  | 192.168.0.13 |

```shell
关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld

关闭selinux：
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config  # 永久
setenforce 0  # 临时

关闭swap：
swapoff -a  # 临时
vim /etc/fstab  # 永久

设置主机名：
hostnamectl set-hostname <hostname>

在master添加hosts：
cat >> /etc/hosts << EOF
192.168.0.11 k8s-m
192.168.0.12 k8s-n1
192.168.0.13 k8s-n2
EOF

将桥接的IPv4流量传递到iptables的链：
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system  # 生效

时间同步：
yum install ntpdate -y
ntpdate ntp.aliyun.com
```

## 4、所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 4.1、安装Docker

```shell
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo

yum -y install docker-ce

systemctl enable docker && systemctl start docker
```

```shell
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://oitxg1ek.mirror.aliyuncs.com"]
}
EOF
systemctl restart docker
docker info
```

### 4.2、添加阿里云YUM软件源

```shell
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 4.3、安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

```shell
yum install -y kubelet-1.20.0 kubeadm-1.20.0 kubectl-1.20.0
systemctl enable kubelet
```

## 5、部署Kubernetes Master

参考文档： https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm-init/#config-file 

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node 

在192.168.0.11（Master）执行。

```shell
kubeadm init \
  --apiserver-advertise-address=192.168.0.11 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.20.0 \
  --service-cidr=10.96.0.0/12 \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all
```

- --apiserver-advertise-address 集群通告地址
- --image-repository  由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址
- --kubernetes-version K8s版本，与上面安装的一致
- --service-cidr 集群内部虚拟网络，Pod统一访问入口
- --pod-network-cidr Pod网络，，与下面部署的CNI网络组件yaml中保持一致

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

或者使用配置文件引导：

```shell
vi kubeadm.conf

apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
imageRepository: registry.aliyuncs.com/google_containers 
networking:
  podSubnet: 10.244.0.0/16 
  serviceSubnet: 10.96.0.0/12 

kubeadm init --config kubeadm.conf ignore-preflight-errors=all  
```

使用kubectl工具：

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

```
命令补全：
```shell
yum install -y bash-completion
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

```shell
kubectl get nodes
NAME               STATUS     ROLES            AGE   VERSION
localhost.localdomain   NotReady   control-plane,master   20s   v1.20.0
```

## 6、加入Kubernetes Node

在192.168.0.12/13（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

```shell
kubeadm join 192.168.0.11:6443 --token t95iga.sslq87o0lncw10ou \
    --discovery-token-ca-cert-hash sha256:d0be719cca5b12b336698d74c77894a237cc30cebcda28b2f6df93a8dbad8a10
```

默认token有效期为24小时，当过期之后，该token就不可用了。这时就需要重新创建token，操作如下：

```shell
kubeadm token list # 列出token信息
kubeadm token create --print-join-command # 生成一个新的token --ttl=0 设置永不过期

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924

kubeadm join 192.168.31.61:6443 --token nuja6n.o3jrhsffiqs9swnu --discovery-token-ca-cert-hash sha256:63bca849e0e01691ae14eab449570284f0c3ddeea590f8da988c07fe2729e924
```

<https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-join/>

## 7、安装Pod网络插件（CNI）

 https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network 

注意：只需要部署下面其中一个，推荐Calico。

### 7.1、Calico

Calico是一个纯三层的数据中心网络方案，Calico支持广泛的平台，包括Kubernetes、OpenStack等。

Calico 在每一个计算节点利用 Linux Kernel 实现了一个高效的虚拟路由器（ vRouter） 来负责数据转发，而每个 vRouter 通过 BGP 协议负责把自己上运行的 workload 的路由信息向整个 Calico 网络内传播。

此外，Calico  项目还实现了 Kubernetes 网络策略，提供ACL功能。

 https://docs.projectcalico.org/getting-started/kubernetes/quickstart 

```shell
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

下载完后还需要修改里面配置项：

- 定义Pod网络（CALICO_IPV4POOL_CIDR），与前面pod CIDR配置一样
- 选择工作模式（CALICO_IPV4POOL_IPIP），支持**BGP（Never）**、**IPIP（Always）**、**CrossSubnet**（开启BGP并支持跨子网）

修改完后应用清单：

```shell
kubectl apply -f calico.yaml
kubectl get pods -n kube-system
```

### 7.2、Flannel

Flannel是CoreOS维护的一个网络组件，Flannel为每个Pod提供全局唯一的IP，Flannel使用ETCD来存储Pod子网与Node IP之间的关系。flanneld守护进程在每台主机上运行，并负责维护ETCD信息和路由数据包。

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
sed -i -r "s#quay.io/coreos/flannel:.*-amd64#lizhenliang/flannel:v0.11.0-amd64#g" kube-flannel.yml
$ kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

确保能够访问到quay.io这个registery。

如果Pod镜像下载失败，可以改成这个镜像地址：lizhenliang/flannel:v0.11.0-amd64

## 8、测试kubernetes集群

- 验证Pod工作
- 验证Pod网络通信
- 验证DNS解析

在Kubernetes集群中创建一个pod，验证是否正常运行：

```shell
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pods,svc
kubectl get pods -o wide
```

访问地址：http://NodeIP:Port  

## 9、部署 Dashboard

```shell
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.3/aio/deploy/recommended.yaml
```

默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部：

```shell
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type:NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```
访问地址：http://NodeIP:30001

创建service account并绑定默认cluster-admin管理员集群角色：

```shell
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```
使用输出的token登录Dashboard。

### 9.1、二进制 部署

注意你部署Dashboard的命名空间（之前部署默认是kube-system，新版是kubernetes-dashboard）

1、 删除默认的secret，用自签证书创建新的secret

```
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs \
--from-file=/opt/kubernetes/ssl/server-key.pem --from-file=/opt/kubernetes/ssl/server.pem -n kubernetes-dashboard
```

2、修改 dashboard.yaml 文件，在args下面增加证书两行

```
       args:
         # PLATFORM-SPECIFIC ARGS HERE
         - --auto-generate-certificates
         - --tls-key-file=server-key.pem
         - --tls-cert-file=server.pem
kubectl apply -f kubernetes-dashboard.yaml
```

### 9.2、kubeadm 部署

注意你部署Dashboard的命名空间（之前部署默认是kube-system，新版是kubernetes-dashboard）

1、删除默认的secret，用自签证书创建新的secret

```
kubectl delete secret kubernetes-dashboard-certs -n kubernetes-dashboard
kubectl create secret generic kubernetes-dashboard-certs \
--from-file=/etc/kubernetes/pki/apiserver.key --from-file=/etc/kubernetes/pki/apiserver.crt -n kubernetes-dashboard
```

2、修改 dashboard.yaml 文件，在args下面增加证书两行

```
       args:
         # PLATFORM-SPECIFIC ARGS HERE
         - --auto-generate-certificates
         - --tls-key-file=apiserver.key
         - --tls-cert-file=apiserver.crt
kubectl apply -f kubernetes-dashboard.yaml
```

## 10、切换容器引擎为Containerd

https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd

1、配置先决条件

```shell
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置必需的 sysctl 参数，这些参数在重新启动后仍然存在。
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

2、安装containerd

```shell
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
yum update -y && sudo yum install -y containerd.io
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
systemctl restart containerd
```

3、修改配置文件

```shell
vi /etc/containerd/config.toml
   [plugins."io.containerd.grpc.v1.cri"]
      sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.2"  
         ...
         [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
             SystemdCgroup = true
             ...
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
          endpoint = ["https://b9pmyelo.mirror.aliyuncs.com"]
          
systemctl restart containerd
```

4、配置kubelet使用containerd

```shell
vi /etc/sysconfig/kubelet 
KUBELET_EXTRA_ARGS=--container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --cgroup-driver=systemd

systemctl restart kubelet
```

5、验证

```shell
kubectl get node -o wide

k8s-node1  xxx  containerd://1.4.4
```

> 直播地址：https://ke.qq.com/course/266656

![](https://k8s-1252881505.cos.ap-beijing.myqcloud.com/wx.png)