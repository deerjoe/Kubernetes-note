# 2 使用kubespary在国内部署kubernetes

## 1. CentOS环境配置

> 以下操作需在每个机器上执行
>
>

### 1.1.关闭防火墙和Selinux

由于个人做实验，需要关闭防火墙，Selinux，若是生产环境建议另外配置防火墙和Selinux（实际上是我不会：）

```bash
systemctl disable firewalld #关闭防火墙自启动
systemctl stop firewalld  #关闭防火墙服务
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config #关闭Selinux
setenforce 0
```

### 1.2.关闭Swap分区

> 重要！Swap分区开启会导致kubelete启动失败，官方也建议关闭。

这一步建议在预装CentOS时取消Swap分区。后期使用swap -a off命令临时禁用。或修改/etc/fstab 注释掉swap，取消开机自动挂载

### 1.3.Reboot生效

## 2. 基础软件配置
### 2.1配置阿里Docker yum源

> 以下操作需在每个机器上执行

由于CentOS 为最小化安装，在每个节点上先安装yum-utils 

```bash
yum install -y git yum-utils 
```

### 2.2 添加阿里yum源

```bash
 yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

### 2.3配置Docker阿里云镜像源并安装docker

配置好阿里云加速器
​	
```bash
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://j4ckfrfn.mirror.aliyuncs.com"]
}
EOF

#官方脚本安装docker，使用阿里云镜像
curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

#安装较旧版本（比如Docker 17.03.2) 时需要指定完整的rpm包的包名，并且加上--setopt=obsoletes=0 参数
#yum install -y --setopt=obsoletes=0 \
#docker-ce-17.03.2.ce-1.el7.centos.x86_64 \
#docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch
```

## 3. ansible安装
### 3.1 ansible介绍
kubespray基于ansible编写，ansible是新出现的自动化运维工具，基于Python开发，集合了众多运维工具（puppet、cfengine、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能。
ansible是基于模块工作的，本身没有批量部署的能力。真正具有批量部署的是ansible所运行的模块，ansible只是提供一种框架。

### 3.1.2 安装ansible前的配置

### 3.1.3  ansible包安装
本教程装在node1节点上，托管节点python的版本需要大于2.5。ansible依赖netaddr Jinja2.8

> 选择一台主机作为ansible控制器，ansible可与k8s不同，但不能是Windows（不支持）

```bash
# 安装 epel
yum install -y epel-release 
# 安装 ansible 及 python git (必须先安装 epel 源再安装 ansible)
yum install -y  python-pip python34 python34-pip deltarpm ansible git
```

### 3.1.4 配置python国内源
修改 ~/.pip/pip.conf

```bash
mkdir ~/.pip
cat >> ~/.pip/pip.conf <<-'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
EOF
```



### 3.1.5 设置免密登录

在ansible-client执行 ssh-keygen -t rsa 生成密钥对

```bash
ssh-keygen -t rsa -P ''
```

节点ip同步密钥

```bash
IP=(10.45.10.180 10.45.10.181 10.45.10.182)
for x in ${IP[*]}; do ssh-copy-id -i ~/.ssh/id_rsa.pub $x; done
```

## 4. 配置kuberspray

### 4.1 下载kuberspray源码

```bash
git clone https://github.com/kubernetes-incubator/kubespray.git
```

也可以选择到kubespary的[releases](https://github.com/kubernetes-incubator/kubespray/releases "releases")下载对应的包文件

### 4.2 安装python依赖

```bash
cd kubespray
pip install -r requirements.txt
```

### 4.3 镜像替换

> 因gcr.io地址被GFW所屏蔽，安装kuberspray会失败，需要从docker仓库中转。网上已有大神使用自动脚本将gcr.io镜像同步到了docker仓库，见[github gcr.io_mirror](https://github.com/anjia0532/gcr.io_mirror "gcr.io_mirror")

在kuberspray源码源代码中搜索包含 gcr.io 镜像的文件

	cd kuberspray
	grep "gcr.io" ./ -nr


搜索结果：

```bash
[root@node1 kubespray]# grep "gcr.io" ./ -nr
匹配到二进制文件 ./.git/objects/pack/pack-96ce5d35f01f3f90ac3e4a5d6ffd20ac98a5d2db.pack
./roles/download/defaults/main.yml:73:hyperkube_image_repo: "gcr.io/google-containers/hyperkube-{{ image_arch }}"
./roles/download/defaults/main.yml:75:pod_infra_image_repo: "gcr.io/google_containers/pause-{{ image_arch }}"
./roles/download/defaults/main.yml:102:kubedns_image_repo: "gcr.io/google_containers/k8s-dns-kube-dns-{{ image_arch }}"
./roles/download/defaults/main.yml:107:dnsmasq_nanny_image_repo: "gcr.io/google_containers/k8s-dns-dnsmasq-nanny-{{ image_arch }}"
./roles/download/defaults/main.yml:109:dnsmasq_sidecar_image_repo: "gcr.io/google_containers/k8s-dns-sidecar-{{ image_arch }}"
./roles/download/defaults/main.yml:112:dnsmasqautoscaler_image_repo: "gcr.io/google_containers/cluster-proportional-autoscaler-{{ image_arch }}"
./roles/download/defaults/main.yml:115:kubednsautoscaler_image_repo: "gcr.io/google_containers/cluster-proportional-autoscaler-{{ image_arch }}"
./roles/download/defaults/main.yml:120:elasticsearch_image_repo: "k8s.gcr.io/elasticsearch"
./roles/download/defaults/main.yml:123:fluentd_image_repo: "k8s.gcr.io/fluentd-elasticsearch"
./roles/download/defaults/main.yml:131:tiller_image_repo: "gcr.io/kubernetes-helm/tiller"
./roles/download/defaults/main.yml:137:registry_proxy_image_repo: "gcr.io/google_containers/kube-registry-proxy"
./roles/download/defaults/main.yml:145:ingress_nginx_default_backend_image_repo: "gcr.io/google_containers/defaultbackend"
./roles/kubernetes-apps/ansible/defaults/main.yml:18:kubedns_image_repo: "gcr.io/google_containers/k8s-dns-kube-dns-{{ image_arch }}"
./roles/kubernetes-apps/ansible/defaults/main.yml:20:dnsmasq_nanny_image_repo: "gcr.io/google_containers/k8s-dns-dnsmasq-nanny-{{ image_arch }}"
./roles/kubernetes-apps/ansible/defaults/main.yml:22:dnsmasq_sidecar_image_repo: "gcr.io/google_containers/k8s-dns-sidecar-{{ image_arch }}"
./roles/kubernetes-apps/ansible/defaults/main.yml:24:kubednsautoscaler_image_repo: "gcr.io/google_containers/cluster-proportional-autoscaler-{{ image_arch }}"
./roles/kubernetes-apps/ansible/defaults/main.yml:53:dashboard_image_repo: gcr.io/google_containers/kubernetes-dashboard-{{ image_arch }}
./roles/kubernetes-apps/registry/README.md:209:        image: gcr.io/google_containers/kube-registry-proxy:0.4
```

并替换为anjia0532/google-containers。 文件基本上在./roles/download/defaults/main与
./roles/kubernetes-apps/ansible/defaults/main.yml内


> 替换规则：
> k8s.gcr.io/{image}/{tag} <==gcr.io/google-containers/{image}/{tag} <==anjia0532/google-containers.{image}/{tag}



### 4.4 配置文件内容

拷贝sample副本，使用脚本生成节点配置文件host.ini

	cp -rfp inventory/sample inventory/mycluster
	declare -a IPS=(10.0.0.88 10.0.0.89 10.0.0.90 10.0.0.91 10.0.0.92 )
	
	#生成节点配置文件，可根据实际需要修改配置的主机名、Master etcd node配置等信息
	CONFIG_FILE=inventory/mycluster/hosts.ini python3 contrib/inventory_builder/inventory.py ${IPS[@]}

安装k8s配置文件位于.kubespray/inventory/mycluster/group_vars/k8s-cluster.yml，其中内部包含了有k8s配置文件的位置和插件配置。这里罗列出需要修改或建议修改的地方



```bash
...
## Change this to use another Kubernetes version, e.g. a current beta release
kube_version: v1.11.2  #k8s安装版本

# Where the binaries will be downloaded.
# Note: ensure that you've enough disk space (about 1G)
local_release_dir: "/tmp/releases"
# Random shifts for retrying failed ops like pushing/downloading
retry_stagger: 5  #docker镜像下载重试次数
```


​	
```yaml
# Choose network plugin (cilium, calico, contiv, weave or flannel)
# Can also be set to 'cloud', which lets the cloud provider setup appropriate routing
kube_network_plugin: calico #k8s网络插件，网上大部分为flannel，生产环境建议用calico

...

# Kubernetes internal network for services, unused block of space.
kube_service_addresses: 10.233.0.0/18 #k8s services的网络地址

# internal network. When used, it will assign IP
# addresses from this range to individual pods.
# This network must be unused in your network infrastructure!
kube_pods_subnet: 10.233.64.0/18 #pod的网络地址

# internal network node size allocation (optional). This is the size allocated
# to each node on your network.  With these defaults you should have
# room for 4096 nodes with 254 pods per node.
kube_network_node_prefix: 24   #节点的子网掩码，注意：每个节点上pod的最大数量等于子网掩码允许的数量

#kube_apiserver监听地址
# The port the API Server will be listening on. 
kube_apiserver_ip: "{{ kube_service_addresses|ipaddr('net')|ipaddr(1)|ipaddr('address') }}"
kube_apiserver_port: 6443 # (https)
kube_apiserver_insecure_port: 8080 # (http)
# Set to 0 to disable insecure port - Requires RBAC in authorization_modes and kube_api_anonymous_auth: true
#kube_apiserver_insecure_port: 0 # (disabled)

# Kube-proxy proxyMode configuration.
# Can be ipvs, iptables
kube_proxy_mode: ipvs #使用高性能建议配置为ipvs

...

# Kubernetes dashboard  建议配置
# RBAC required. see docs/getting-started.md for access details.
dashboard_enabled: true

....

#ingress配置，建议配置
ingress_nginx_enabled: false
ingress_nginx_host_network: false
ingress_nginx_nodeselector:
  node-role.kubernetes.io/master: "true"
ingress_nginx_namespace: "ingress-nginx"
ingress_nginx_insecure_port: 80
ingress_nginx_secure_port: 443
ingress_nginx_configmap:
  map-hash-bucket-size: "128"
  ssl-protocols: "SSLv2"
#  ingress_nginx_configmap_tcp_services:
#    9000: "default/example-go:8080"
ingress_nginx_configmap_udp_services:
   53: "kube-system/coredns:53" #根据dns服务器修改，这里安装了coredns

# Cert manager 配置，若使用Let's encrypt建议配置
cert_manager_enabled: false
cert_manager_namespace: "cert-manager"
```

## 5. 使用kuberspray

### 5.1. 安装k8s集群

安装k8s，执行完后使用kubeclt get node 查看节点信息

	ansible-playbook -i inventory/mycluster/hosts.ini cluster.yml

### 5.2. 升级k8s集群



安全升级

	git fetch origin
	git checkout origin/master
	ansible-playbook upgrade-cluster.yml -b -i inventory/mycluster/hosts.ini -e kube_version=v1.6.0 #修改成需要升级的版本

### 5.3. 升级k8s部件

升级 docker:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=docker
升级 etcd:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=etcd
升级 vault:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=vault
升级 kubelet:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=node --skip-tags=k8s-gen-certs,k8s-gen-tokens
升级 Kubernetes master:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=master
升级网络插件:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=network
升级所有插件:

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=apps

升级单个插件： helm (helm_enabled is true):

	ansible-playbook -b -i inventory/mycluster/hosts.ini cluster.yml --tags=helm

###5.4. 扩展k8s集群、移除节点、删除集群

扩展集群

	ansible-playbook -b -i inventory/mycluster/hosts.ini scale.yml

移除节点

	ansible-playbook -b -i inventory/mycluster/hosts.ini remove-node.yml --limit node5 #节点名称
