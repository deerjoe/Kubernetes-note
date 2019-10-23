# 使用二进制搭建Kubernetes

## 0. 组件版本及配置策略

### 0.1 组件版本

- Kubernetes v1.16.2
- Docker 待定
- Etcd v3.3.10
- Flanneld 待定
- 插件：
  - CoreDNS
  - Dashboard
  - Metrics-Serve



### 0.2 主要配置策略

- kube-apiserver：

  - 使用 keepalived 和 haproxy 实现 3 节点高可用；
    关闭非安全端口 8080 和匿名访问；
    在安全端口 6443 接收 https 请求；
    严格的认证和授权策略 (x509、token、RBAC)；
    开启 bootstrap token 认证，支持 kubelet TLS bootstrapping；
    使用 https 访问 kubelet、etcd，加密通信；
  - 
- kube-controller-manager：、

  - 3 节点高可用；
  - 关闭非安全端口，在安全端口 10252 接收 https 请求；
  - 使用 kubeconfig 访问 apiserver 的安全端口；
  - 自动 approve kubelet 证书签名请求 (CSR)，证书过期后自动轮转；
  - 各 controller 使用自己的 ServiceAccount 访问 apiserver；
- kube-scheduler：
  - 3 节点高可用；
  - 使用 kubeconfig 访问 apiserver 的安全端口；

- kubelet：

  - 使用 kubeadm 动态创建 bootstrap token，而不是在 apiserver 中静态配置；
  - 使用 TLS bootstrap 机制自动生成 client 和 server 证书，过期后自动轮转；
  - 在 KubeletConfiguration 类型的 JSON 文件配置主要参数；
  - 关闭只读端口，在安全端口 10250 接收 https 请求，对请求进行认证和授权，拒绝匿名访问和非授权访问；
  - 使用 kubeconfig 访问 apiserver 的安全端口；
- kube-proxy：

  - 使用 kubeconfig 访问 apiserver 的安全端口；
  - 在 KubeProxyConfiguration 类型的 JSON 文件配置主要参数；
  - 使用 ipvs 代理模式；

## 1. 系统规划及其配置

### 1.1 集群机器

操作系统：Ubuntu 18.04，etcd集群、master节点、worker节点均使用以下三节点

- kube-master：10.0.0.31
- kube-node1:  10.0.0.32
- kube-node2: 10.0.0.33

### 1.2 主机配置

#### 1.2.1 设置主机名及hostname对应关系

分别对三台主机设置永久主机名：

```bash
sudo hostnamectl set-hostname kube-master

sudo hostnamectl set-hostname kube-node1

sudo hostnamectl set-hostname kube-node2
```



对所有主机修改host文件，添加主机名对应关系

```bash
sudo nano /etc/hosts

10.0.0.31 kube-master

10.0.0.32 kube-node1

10.0.0.33 kube-node2

```
#### 1.2.2 添加账户并配置sudo

在Master主机上生成SSH秘钥及无密码登录其他节点，本教程使用用户为phadmin，以下使用phadmin的用户名时请注意替换

```bash
sudo useradd -m phadmin
sudo sh -c 'echo along |passwd phadmin --stdin' #为phadmin账户设置密码
sudo sh -c " echo \"phadmin ALL=(ALL) NOPASSWD: ALL\" >> /etc/sudoers"

```



### 1.2.3 关闭防火墙及交换分区

1. 关闭三台主机防火墙

   ```bash
   sudo systemctl stop firewalld && sudo systemctl disable firewalld
   ```

   

2. 关闭三台主机交换分区

   ```bash
   sudo swapoff -a
   ```

   注释fstab中swap的信息

   ```bash
   sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
   ```

#### 1.2.4 安装及设置docker

   

1. 安装docker

```bash
 #使用官方脚本安装
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun
 #将用户加入docker组
sudo usermod -aG docker <用户名>
```
2. 增加docker加速器镜像

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://j4ckfrfn.mirror.aliyuncs.com"]
}
EOF
```



#### 1.2.5 安装依赖及修改内核参数  

1. 安装依赖

```bash
sudo apt-get install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```

2. 加载内核模块

```
    sudo modprobe br_netfilter
    sudo modprobe ip_vs
```

​    


3. 设置内核参数

    

```bash
cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
    
sudo cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
#启用内核参数
sudo sysctl -p /etc/sysctl.d/kubernetes.conf
#确认Cgroup是否mount
sudo mount -t cgroup -o cpu,cpuacct none /sys/fs/cgroup/cpu,cpuacct
    
```

4. 设置系统时区

```bash
#调整系统 TimeZone
sudo timedatectl set-timezone Asia/Shanghai
#将当前的 UTC 时间写入硬件时钟
sudo timedatectl set-local-rtc 0
#重启依赖于系统时间的服务
sudo systemctl restart rsyslog
sudo systemctl restart crond
#同步系统时间
sudo apt install ntpdate
sudo ntpdate cn.pool.ntp.org
```
### 1.3 文件夹创建

```bash
sudo mkdir -p /etc/kubernetes
sudo mkdir -p /etc/kubernetes/ssl
sudo mkdir -p /etc/kubernetes/manifests
sudo chown -R phadmin /etc/kubernetes/
```



## 2. 创建CA证书和秘钥

### 2.1 安装cfssl工具集

```bash
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo
mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssl-certinfo /usr/local/bin/cfssljson

#添加相关文件执行权限
sudo chmod +x /usr/local/bin/cfssl /usr/local/bin/cfssljson /usr/local/bin/cfssl-certinfo 
```



### 2.2 创建CA证书

#### 2.2.1 创建证书配置

```bash
cd  /etc/kubernetes/ssl
nano ca-config.json
{
    "signing": {
        "default": {
            "expiry": "87600h" 
        },
        "profiles": {
            "kubernetes": {
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ],
                "expiry": "87600h"
            }
        }
    }
}
```

注：

① signing ：表示该证书可用于签名其它证书，生成的 ca.pem 证书中CA=TRUE ；

② server auth ：表示 client 可以用该该证书对 server 提供的证书进行验证；

③ client auth ：表示 server 可以用该该证书对 client 提供的证书进行验证；

####  2.2.2 创建CA证书请求

```bash
cd  /etc/kubernetes/ssl
nano ca-csr.json
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "HuiZhou",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
```

注：

① CN： Common Name ，kube-apiserver 从证书中提取该字段作为请求的用户名(User Name)，浏览器使用该字段验证网站是否合法；

② O： Organization ，kube-apiserver 从证书中提取该字段作为请求用户所属的组(Group)；

③ kube-apiserver 将提取的 User、Group 作为 RBAC 授权的用户标识；



#### 2.2.3 生成证书文件并分发

```bash
#生成证书
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
#分发证书
NODE_IPS=("10.0.0.31" "10.0.0.32" "10.0.0.33")
for node_ip in ${NODE_IPS[@]};do
    echo ">>> ${node_ip}"
    scp /etc/kubernetes/ssl/ca*.pem /etc/kubernetes/ssl/ca-config.json phadmin@${node_ip}:/etc/kubernetes/ssl/
done
```

## 3. 部署etcd集群

　　etcd 是基于 Raft 的分布式 key-value 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。kubernetes 使用 etcd 存储所有运行数据。

本文档介绍部署一个三节点高可用 etcd 集群的步骤：

① 下载和分发 etcd 二进制文件

② 创建 etcd 集群各节点的 x509 证书，用于加密客户端(如 etcdctl) 与 etcd 集群、etcd 集群之间的数据流；

③ 创建 etcd 的 systemd unit 文件，配置服务参数；

④ 检查集群工作状态；

### 3.1 下载etcd二进制文件

``` bash
sudo wget http://mirror.kaiyuanshe.cn/kubernetes/etcd/etcd-v3.3.10-linux-amd64.tar.gz
tar -zxvf etcd-v3.3.10-linux-amd64.tar.gz
cd etcd-v3.3.10-linux-amd64
sudo mv etcd /usr/local/bin/etcd
sudo mv etcdctl /usr/local/bin/etcdctl
sudo chmod +x /usr/local/bin/etcd
sudo chmod +x /usr/local/bin/etcdctl
```

### 3.2  创建etcd证书和私钥

#### 3.2.1 创建证书签名请求 

```bash
mkdir -p /etc/kubernetes/ssl/etcd
cd /etc/kubernetes/ssl/etcd
cat >etcd-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "10.0.0.31",
        "10.0.0.32",
        "10.0.0.33"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "HuiZhou",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```



#### 3.2.2 生成证书和私钥

```bash
cd /etc/kubernetes/ssl/etcd
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
              -ca-key=/etc/kubernetes/ssl/ca-key.pem \
              -config=/etc/kubernetes/ssl/ca-config.json \
              -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```



#### 3.2.3 分发生成的证书及私钥到etcd节点

```bash
NODE_IPS=("10.0.0.31" "10.0.0.32" "10.0.0.33")
for node_ip in ${NODE_IPS[@]};do
        echo ">>> ${node_ip}"
        scp /usr/local/bin/etcd* phadmin@${node_ip}:/tmp/
        ssh -t phadmin@${node_ip} "sudo chmod +x /tmp/etcd* && sudo mv /tmp/etcd /usr/local/bin/etcd && sudo mv /tmp/etcdctl /usr/local/bin/etcdctl "
        ssh -t phadmin@${node_ip} "mkdir -p /etc/kubernetes/ssl/etcd && chown -R phadmin /etc/kubernetes/ssl/"
        scp /etc/kubernetes/ssl/etcd/etcd*.pem phadmin@${node_ip}:/etc/kubernetes/ssl/etcd/
done
```



### 3.3 创建etcd systemd unit模板及配置文件

#### 3.3.1 创建etcd systemd unit 模板

创建systemd unit模板，以便分发时修改对应参数

```bash
cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
User=root
Type=notify
PermissionsStartOnly=true
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
    --data-dir=/var/lib/etcd/ \
    --name ##NODE_NAME## \
    --cert-file=/etc/kubernetes/ssl/etcd/etcd.pem \
    --key-file=/etc/kubernetes/ssl/etcd/etcd-key.pem \
    --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --peer-cert-file=/etc/kubernetes/ssl/etcd/etcd.pem \
    --peer-key-file=/etc/kubernetes/ssl/etcd/etcd-key.pem \
    --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --listen-peer-urls=https://##NODE_IP##:2380 \
    --initial-advertise-peer-urls=https://##NODE_IP##:2380 \
    --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379\
    --advertise-client-urls=https://##NODE_IP##:2379 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=etcd0=https://10.0.0.31:2380,etcd1=https://10.0.0.32:2380,etcd2=https://10.0.0.33:2380 \
    --initial-cluster-state=new
Restart=on-failure
RestartSec=15s
TimeoutStartSec=30s
[Install]
WantedBy=multi-user.target
EOF
```



- WorkingDirectory 、 --data-dir ：指定工作目录和数据目录为/var/lib/etcd ，需在启动服务前创建这个目录；
- --name ：指定节点名称，当 --initial-cluster-state 值为 new 时， --name 的参数值必须位于 --initial-cluster 列表中；
- --cert-file 、 --key-file ：etcd server 与 client 通信时使用的证书和私钥；
- --trusted-ca-file ：签名 client 证书的 CA 证书，用于验证 client 证书；
- --peer-cert-file 、 --peer-key-file ：etcd 与 peer 通信使用的证书和私钥；
- --peer-trusted-ca-file ：签名 peer 证书的 CA 证书，用于验证 peer 证书；



#### 3.3.2 为etcd节点创建和分发systemd unit文件

```bash
NODE_NAMES=("etcd0" "etcd1" "etcd2")
NODE_IPS=("10.0.0.31" "10.0.0.32" "10.0.0.33")
#替换模板文件中的变量，为各节点创建 systemd unit 文件
for (( i=0; i < 3; i++ ));do
        sed -e "s/##NODE_NAME##/${NODE_NAMES[i]}/g" -e "s/##NODE_IP##/${NODE_IPS[i]}/g" etcd.service.template > etcd-${NODE_IPS[i]}.service
done
#分发生成的 systemd unit 和etcd的配置文件：
for node_ip in ${NODE_IPS[@]};do
        echo ">>> ${node_ip}"
        ssh phadmin@${node_ip} "sudo mkdir -p /var/lib/etcd && sudo chown -R phadmin /var/lib/etcd"
        scp etcd-${node_ip}.service phadmin@${node_ip}:/tmp/etcd.service
        ssh phadmin@${node_ip} "sudo mv /tmp/etcd.service /etc/systemd/system/etcd.service "
done

```

### 3.3.3 启动并验证etcd服务

```bash
NODE_IPS=("10.0.0.31" "10.0.0.32" "10.0.0.33")
#启动 etcd 服务
for node_ip in ${NODE_IPS[@]};do
        echo ">>> ${node_ip}"
        ssh phadmin@${node_ip} "sudo systemctl daemon-reload && sudo systemctl enable etcd && sudo systemctl start etcd"
done
#检查启动结果,确保状态为 active (running)
for node_ip in ${NODE_IPS[@]};do
        echo ">>> ${node_ip}"
        ssh phadmin@${node_ip} "sudo systemctl status etcd|grep Active"
done
#验证服务状态,输出均为healthy 时表示集群服务正常
for node_ip in ${NODE_IPS[@]};do
        echo ">>> ${node_ip}"
        ETCDCTL_API=3 /usr/local/bin/etcdctl \
--endpoints=https://${node_ip}:2379 \
--cacert=/etc/kubernetes/ssl/ca.pem \
--cert=/etc/kubernetes/ssl/etcd/etcd.pem \
--key=/etc/kubernetes/ssl/etcd/etcd-key.pem endpoint health
done 
```



## 4. 部署master节点

① kubernetes master 节点运行如下组件：

- kube-apiserver
- kube-scheduler
- kube-controller-manager

② kube-scheduler 和 kube-controller-manager 可以以集群模式运行，通过 leader 选举产生一个工作进程，其它进程处于阻塞模式。

③ 对于 kube-apiserver，可以运行多个实例（本文档是 3 实例），但对其它组件需要提供统一的访问地址，该地址需要高可用。本文档使用 keepalived 和 haproxy 实现 kube-apiserver VIP 高可用和负载均衡。

④ 因为对master做了keepalived高可用，所以3台服务器都有可能会升成master服务器（主master宕机，会有从升级为主）；因此所有的master操作，在3个服务器上都要进行。

### 4.1 更新微软kubernetes镜像源



```
VER=v1.16.2
wget https://dl.k8s.io/$VER/kubernetes-server-linux-amd64.tar.gz

```



### 4.2 部署高可用组件













































## 3. 安装并配置Kubelet

### 3.1 下载kubelet 

- 由于k8s.io被墙，可以采用国内的镜像下载，这里采用微软镜像download，下同。

  ```bash
  VER=v1.16.2
  sudo wget https://mirror.azure.cn/kubernetes/kubectl/$VER/bin/linux/amd64/kubectl -O /usr/local/bin/kubectl
  sudo chmod +x /usr/local/bin/kubectl
  ```

### 3.2 创建admin证书和私钥

- kubectl 与 apiserver https 安全端口通信，apiserver 对提供的证书进行认证和授权。

- kubectl 作为集群的管理工具，需要被授予最高权限。这里创建具有最高权限的admin 证书。

  


#### 3.2.1 创建证书签名请求

  ```bash
cd /etc/kubernetes/ssl/
cat > admin-csr.json <<EOF
{
    "CN": "admin",
    "hosts": [],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "GuangDong",
            "L": "HuiZhou",
            "O": "system:masters",
            "OU": "System"
        }
    ]
}
EOF

  ```

注：

① O 为 system:masters ，kube-apiserver 收到该证书后将请求的 Group 设置为system:masters；

② 预定义的 ClusterRoleBinding cluster-admin 将 Group system:masters 与Role cluster-admin 绑定，该 Role 授予所有 API的权限；

③ 该证书只会被 kubectl 当做 client 证书使用，所以 hosts 字段为空；

#### 3.2.2 生成证书和私钥

```bash
cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
              -ca-key=/etc/kubernetes/ssl/ca-key.pem \
              -config=/etc/kubernetes/ssl/ca-config.json \
              -profile=kubernetes admin-csr.json \
              | cfssljson -bare admin
```



### 3.3 kubectl客户端配置

#### 3.3.1 创建kubeconfig文件

kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如apiserver 地址、CA 证书和自身使用的证书；

① 设置集群参数，（--server=${KUBE_APISERVER} ，指定IP和端口；如果没有haproxy代理，就用实际服务的IP和端口；如：https://10.0.0.31:6443）

```bash
export KUBE_APISERVER="https://10.0.0.31:6443"
kubectl config set-cluster kubernetes \
--certificate-authority=/etc/kubernetes/ssl/ca.pem \
--embed-certs=true \
--server=${KUBE_APISERVER} 
```

② 设置客户端认证参数

```bash
kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
```

③ 设置上下文参数

```bash
kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin
```

④ 设置默认上下文
```bash
kubectl config use-context kubernetes
```
- --certificate-authority ：验证 kube-apiserver 证书的根证书；
- --client-certificate 、 --client-key ：刚生成的 admin 证书和私钥，连接kube-apiserver 时使用；
- --embed-certs=true ：将 ca.pem 和 admin.pem 证书内容嵌入到生成的kubectl.kubeconfig 文件中(不加时，写入的是证书文件路径)；
- 生成的 kubeconfig 被保存到 `~/.kube/config` 文件；配置文件描述了集群、用户和上下文

 