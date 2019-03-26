---
layout: post
title:  "kubernetes手动安装"
date:   2018-11-05 11:07:01 +0000
categories: Linux Docker
---

# kubernetes安装与配置

标签（空格分隔）： 参考

---

版本说明
```
kubernetes 1.10
docker docker-ce-18.03.1.ce
etcd 3.2.15
flannel 0.7.1
```
# kubernetes安装与配置

标签（空格分隔）： 参考

---

版本说明
```
kubernetes 1.12
docker docker-ce-18.03.1.ce
etcd 3.2.15
flannel 0.7.1
```

注意：本文以centos 7.3为例进行说明。各个说明尽量

## 一、环境准备

### 基础软件
```
yum install -y gcc  gcc-c++  wget  lrzsz telnet net-tools  epel*  vim   unzip  ntpdate  yum-utils device-mapper-persistent-data conntrack-tools  libseccomp libtool-ltdl
```
### 下载kubernetes（ALL）

提前下载好kubernetes，put kubernates to

```xshell
## put to /tmp/kub/kubernetes-server-linux-amd64.tar.gz
# 解压安装包，复制文件
tar -xzvf kubernetes-server-linux-amd64.tar.gz
cp kubernetes/server/bin/kube* /usr/bin/
#安装并赋予可执行权限，继续进行操作：
chmod a+x /usr/bin/kube*
```

### docker安装（节点）

```
echo ">>>>>>>>>>>>>>installdocker-ce strat<<<<<<<<<<<"
cd /usr/local
yum remove docker docker-common docker-selinux docker-engine
yum install -y yum-utils
yum install -y device-mapper-persistent-data
yum install -y lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
# yum list docker-ce --showduplicates | sort -r 查看现有docker-ce版本
yum install -y docker-ce-18.03.1.ce
curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s  http://7b8275a8.m.daocloud.io Copy
systemctl restart docker
docker ps
echo ">>>>>>>>>>>>>>installdocker-ce end<<<<<<<<<<<"
```

可以根据操作系统版本参考：https://docs.docker.com/install/#releases

### 拉取镜像（all）

```xshell
docker pull registry.docker-cn.com/coredns/coredns:0.9.10
docker pull coredns/coredns:1.2.4
```

### 安装CFSSL（master）

```
    wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
    wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

    chmod +x cfssl_linux-amd64
    mv cfssl_linux-amd64 /usr/local/bin/cfssl

    chmod +x cfssljson_linux-amd64
    mv cfssljson_linux-amd64 /usr/local/bin/cfssljson

    chmod +x cfssl-certinfo_linux-amd64
    mv cfssl-certinfo_linux-amd64 /usr/local/bin/cfssl-certinfo

    echo PATH=/usr/local/bin:$PATH >> ~/.bash_profile
```

### 安装Etcd(all)

```xshell
#查询版本是否合适，我这里是3.2.15版本
yum info etcd 
#安装
yum install -y etcd-3.2.15
```

### 环境准备

#### 2. 关闭firewalld

在本地的安装中，暂时关闭了防火墙。总体安装测试完成后，请按实际需求确认需要开放的端口后再开启防火墙。

```
systemctl stop firewalld
systemctl disable firewalld
```

#### 3. 关闭所有节点的Selinux

    setenforce 0
    sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config



#### 6.时间校对

```
    #安装ntp
    yum install -y ntp
    #校对时间
    ntpdate ntp1.aliyun.com
    hwclock -w
```
#### 7.linux底层的iptables 启动容器的所有节点每次开机都要执行      
```
    iptables --list
    iptables -P  FORWARD ACCEPT
```
#### 8.关闭swap分区
```
    #关闭全部
    swapoff -a
    ##注释掉vi /etc/fstab中带有swap的一行,则重启也不会开启   
    sed -i 's/^.*swap/#&/g' /etc/fstab
```
#### 9.更改主机名和hosts

设置统一环境变量

```
echo  master=192.168.102.111 >> ~/.bashrc
echo  n1=192.168.102.112 >> ~/.bashrc
echo  n2=192.168.102.113 >> ~/.bashrc

#个性的变量
echo  myname=node2 >> ~/.bashrc
echo  myip=192.168.102.113 >> ~/.bashrc

source ~/.bashrc

```


分别在三台不同的机器上：

```
    #更改hostname
    hostnamectl set-hostname $myname

    ##配置/etc/hosts文件
    echo $master master >> /etc/hosts
    echo $n1 node1  >> /etc/hosts
    echo $n2 node2 >> /etc/hosts

    hostname
    cat /etc/hosts
```

结构

    IP  节点  备注
    10.10.90.105  master  etcd复用此节点
    10.10.90.106  node1 etcd复用此节点
    10.10.90.107  node2 etcd复用此节点

#### 10.阿里服务器部署的改动

所有service文件start启动参数中去掉“\”

#### 11.多master时：

kube-controller-manager 查看当前的 leader
```
kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
```
kube-scheduler 查看当前的 leader
```
kubectl get endpoints kube-scheduler --namespace=kube-system  -o yaml
```
master节点的kubectl命令指定server：
```
alias kubectl="kubectl -s http://172.17.95.3:8080"
```
IP为启动haproxy的主机IP

#### 12.加入节点，原机器需要修改

(1) /etc/hosts修改
(2) 修改：kubernetes-csr.json  重新生成：kubernetes.pem kubernetes.csr  kubernetes-key.pem

### 二、安装预览

    1、创建 TLS 证书和秘钥 

    2、创建kubeconfig 文件 

    3、创建高可用etcd集群

    4、部署master节点 

    5、安装flannel网络插件 

    6、部署node节点 

    7、安装coredns插件 

    8、安装dashboard插件 

### 三、部署步骤

#### 1.创建TLS证书和秘钥(master)

1)创建CA

```xshell
mkdir /root/ssl
cd /root/ssl
cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```
根据config.json文件的格式创建如下的ca-config.json文件
过期时间设置成了 87600h

```xshell
cat > ca-config.json <<EOF
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
EOF
```

字段说明

    ca-config.json：可以定义多个 
    profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；
    signing：表示该证书可用于签名其它证书；生成的 ca.pem 证书中 CA=TRUE；
    server auth：表示client可以用该 CA 对server提供的证书进行验证；
    client auth：表示server可以用该CA对client提供的证书进行验证；

3)创建CA证书签名请求

创建 CA 证书签名请求

创建 ca-csr.json 文件：
```xshell
cat >ca-csr.json << EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
目前为止4个文件了。

4)生成CA证书私钥
```xshell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
ls ca*
```
输出
```
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```

目前为止7个文件了，ca开头的5个文件

5)创建kubernetes证书

hosts字段填写上所有你要用到的节点ip，创建 kubernetes 证书签名请求文件 kubernetes-csr.json：
```xshell
cat >kubernetes-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "127.0.0.1",
      "${master}",
      "${n1}",
      "${n2}",
      "10.254.0.1",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
6)生成kubernetes证书和私钥
```xshell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes
ls kubernetes*
```
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
截止到目前11个文件了，kuber开头的4个

以上2步可以合并成一个步骤，少生成1个kubernetes-csr.json文件，直接在命令行中输入参数代理了文件输入。
```xshell
echo '{"CN":"kubernetes","hosts":[""],"key":{"algo":"rsa","size":2048}}' | cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes -hostname="127.0.0.1,10.10.90.105,10.10.90.106,10.10.90.106,10.254.0.1,kubernetes,kubernetes.default" - | cfssljson -bare kubernetes
```
7)创建admin证书
```xshell
cat >admin-csr.json << EOF
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
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
EOF
```
8)生成admin证书和私钥
```xshell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
ls admin*
```

admin.csr  admin-csr.json  admin-key.pem  admin.pem
截止目前15个文件，admin开头的4个

9）创建kuber-proxy证书
```xshell
cat >kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
EOF
```
10)生成相关证书和私钥
```xshell
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes  kube-proxy-csr.json | cfssljson -bare kube-proxy
ls kube-proxy*
```

kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
截止到目前19个文件，kube-proxy开头的4个

11)校验证书，举例校验kubernetes.pem证书，2个方法都可以，看输出内容可json定义是否一致。
```xshell
openssl x509  -noout -text -in  kubernetes.pem
cfssl-certinfo -cert kubernetes.pem
```
12)分发证书
 
将生成的证书cp到指定目录备用，除了master，2个node节点也需要拷贝到这个这些文件，为了方便copy文件，建议2个node节点针对master做免密码登录

```xshell
mkdir -p /etc/kubernetes/ssl
cp *.pem /etc/kubernetes/ssl

ssh root@$n1 'mkdir -p /etc/kubernetes/ssl'
scp *.pem root@$n1:/etc/kubernetes/ssl

ssh root@$n2 'mkdir -p /etc/kubernetes/ssl'
scp *.pem root@$n2:/etc/kubernetes/ssl
```

从上面的顺序可以看出pem文件的创建都是以一个json文件为输入进行创建的，只需要把pem文件分别scp拷贝的所有节点的/etc/kubernetes/ssl文件夹即可。

## 二、证书kubeconfig文件创建(master执行)

###1、创建TLS bootstrapping Token，即token.csv文件
```xshell
export BOOTSTRAP_TOKEN=$(head -c 16 /dev/urandom | od -An -t x | tr -d ' ')
cat > token.csv <<EOF
${BOOTSTRAP_TOKEN},kubelet-bootstrap,10001,"system:kubelet-bootstrap"
EOF
```
注意事项：

  更新 token.csv 文件，分发到所有机器 (master 和 node）的 /etc/kubernetes/ 目录下，分发到node节点上非必需；

拷贝文件
```xshell
cp token.csv /etc/kubernetes

scp token.csv root@$n1:/etc/kubernetes
scp token.csv root@$n2:/etc/kubernetes

```

###2、创建kubelet（node节点）要用的bootstrap.kubeconfig文件（master）

注意：以下都是一条条执行的命令，不是复制这些代码到文件里。

多master时，KUBE_APISERVER为启动haproxy的主机IP。
```xshell
cd /etc/kubernetes

#变量
echo  KUBE_APISERVER="https://${master}:6443" >> ~/.bashrc
source ~/.bashrc

# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=bootstrap.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials kubelet-bootstrap \
  --token=${BOOTSTRAP_TOKEN} \
  --kubeconfig=bootstrap.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
###3、创建节点要用的kube-proxy.kubeconfig文件
```xshell
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-proxy.kubeconfig
# 设置客户端认证参数
kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
# 设置默认上下文
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
###4、分发文件

将两个 kubeconfig 文件分发到所有 Node 机器的 /etc/kubernetes/ 目录，本机就是生产的那些
```xshell
scp bootstrap.kubeconfig kube-proxy.kubeconfig root@$n1:/etc/kubernetes/
scp bootstrap.kubeconfig kube-proxy.kubeconfig root@$n2:/etc/kubernetes/
```
## 三、创建高可用etcd集群

这里的etcd集群复用我们测试的3个节点，3个node都要安装并启动，注意修改配置文件.
其实etcd的机器与目前的机器逻辑上是不相关的，即也可以直接将etcd放到其他的集群上。

###3、创建etcd的systemd unit文件（既centos7下的服务定义文件）

若使用yum安装，默认etcd命令将在/usr/bin目录下，注意修改下面的etcd.service文件中的启动命令地址为/usr/bin/etcd

文件位置：/usr/lib/systemd/system/etcd.service ，默认该文件存在，删除重建即可。

参照  kubernete安装、启动文件\k8s配置文件 内容


```
cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name etcd-host2 \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --peer-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --peer-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \
  --initial-advertise-peer-urls https://${myip}:2380 \
  --listen-peer-urls https://${myip}:2380 \
  --listen-client-urls https://${myip}:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://${myip}:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster etcd-host0=https://${master}:2380,etcd-host1=https://${n1}:2380,etcd-host2=https://${n2}:2380\
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```
配置注意事项：所有节点都必须配置此文件，并且注意下面4个注意事项。

1)IP地址除了initial-cluster 配置项是配置集群内3个地址的ip外，其他IP均为本机的IP。

2)配置下--name必须与--initial-cluster配置项里的的对应，比master的配置中的--name是etcd-host0，下面的IP对应的名称也是这个。

3)通过不同方式安装的软件Execstart配置项下的程序启动命令路径注意修改

4)WorkingDirectory工作目录需要实现创建，否则启动会报错。

###4、创建etcd环境变量文件

文件位置：/etc/etcd/etcd.conf，yum安装完之后该文件会存在，删除重建即可。
```
etcdname=infra2

mkdir -p /etc/etcd
cat > /etc/etcd/etcd.conf << EOF
# [member]
ETCD_NAME=${etcdname}
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_LISTEN_PEER_URLS="https://${myip}:2380"
ETCD_LISTEN_CLIENT_URLS="https://${myip}:2379"

#[cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://${myip}:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://${myip}:2379"
EOF

```

注意事项：

1)再次提醒ETCD_DATA_DIR一定要存在，其他的IP地址替换为本机的即可，maser及node节点都需要配置

2)ETCD_NAME按照etcd系统服务里面的配置一一对应，分别是infra1,infra2,infra3
　　
###5、设置开机启动及启动etcd
```xshell
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd
systemctl status etcd
```
###6、检测集群工作情况

在任意一个节点，master或者node都可以，执行以下命令
```xshell
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  cluster-health
```

如果输出类似如下如的情况，代表成功：

```
member 7f17bb5fcf278bf9 is healthy: got healthy result from https://192.168.1.167:2379
member c8e5d79e25edfe3f is healthy: got healthy result from https://192.168.1.159:2379
member d4d6147eed9aedcf is healthy: got healthy result from https://192.168.1.164:2379
```
结果最后一行为 cluster is healthy 时表示集群服务正常

注意事项：

1)建议所有节点都运行一次进行检测，我在maser检点进行检测的时候发现master本身的几点都链接上，报unhealthy错误，查看报错后发现是使用了代理上网设置

遇到的状况：为了在线软件设置的上网代理，需要关闭代理，取消配置参数，重启了服务器才检测成功。

2)防火墙务必关闭，否则会有链接不到etcd的问题。这里注意开放的几个端口，不一定非得关闭防火墙。

3)以后使用etcd查询数据都需要使用认证文件，查询格式：
```xshell
etcdctl \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  cluster-health
```
否则出错，例如：
```xshell
[root@kube_master ssl]# etcdctl cluster-health
failed to check the health of member 2f590aa6fa719c4b on https://10.10.90.105:2379: Get https://10.10.90.105:2379/health: x509: certificate signed by unknown authority
member 2f590aa6fa719c4b is unreachable: [https://10.10.90.105:2379] are all unreachable
failed to check the health of member 43ea47a48fb7ffce on https://10.10.90.106:2379: Get https://10.10.90.106:2379/health: x509: certificate signed by unknown authority
member 43ea47a48fb7ffce is unreachable: [https://10.10.90.106:2379] are all unreachable
failed to check the health of member d965bb336acbfc6c on https://10.10.90.107:2379: Get https://10.10.90.107:2379/health: x509: certificate signed by unknown authority
member d965bb336acbfc6c is unreachable: [https://10.10.90.107:2379] are all unreachable
cluster is unhealthy
```

## 四、Master节点安装

etcd集群为3台，分别复用这3台虚拟机。

作为k8s的核心，master节点主要包含三个组件，分别是：
```
三个组件：
kube-apiserver
kube-scheduler
kube-controller-manager
```

### 1、创建TLS证书

这些证书我们在第一篇文章中已经创建，共8个，这里核对一下数量是否正确，至于证书是否正确参考第一篇文章的注释实现。位置：105虚拟机master节点

```xshell
# ls /etc/kubernetes/ssl
admin-key.pem  admin.pem  ca-key.pem  ca.pem  kube-proxy-key.pem  kube-proxy.pem  kubernetes-key.pem  kubernetes.pem
```


### 3、制作apiserver的服务文件

/usr/lib/systemd/system/kube-apiserver.service内容：
```
cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Service
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=etcd.service
Wants=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/bin/kube-apiserver \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_ETCD_SERVERS \
        $KUBE_API_ADDRESS \
        $KUBE_API_PORT \
        $KUBELET_PORT \
        $KUBE_ALLOW_PRIV \
        $KUBE_SERVICE_ADDRESSES \
        $KUBE_ADMISSION_CONTROL \
        $KUBE_API_ARGS
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

EOF
```

制作/etc/kubernetes/config通用文件，的内容为：

```
cat > /etc/kubernetes/config << EOF
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://${master}:8080"
EOF
```

多master时：KUBE_MASTER="--master=http://10.10.90.105:7070"。IP为master本机IP。

kube-apiserver的配置文件/etc/kubernetes/apiserver内容为：
```

cat > /etc/kubernetes/apiserver << EOF
###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--advertise-address=${master} --bind-address=${master}  --insecure-bind-address=0.0.0.0"

# The port on the local server to listen on.
#KUBE_API_PORT="--port=8080"

# Port minions listen on
# KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=https://${master}:2379,https://${n1}:2379,https://${n2}:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction"

# Add your own!
KUBE_API_ARGS="--authorization-mode=RBAC,Node --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h" 
EOF

```
多master时（apiserver端口设置为5443 7070）：

KUBE_API_ARGS="--authorization-mode=RBAC,Node --runtime-config=rbac.authorization.k8s.io/v1beta1 --kubelet-https=true --enable-bootstrap-token-auth --token-auth-file=/etc/kubernetes/token.csv --secure-port=5443 --insecure-port=7070 --service-node-port-range=30000-32767 --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem --client-ca-file=/etc/kubernetes/ssl/ca.pem --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem --etcd-cafile=/etc/kubernetes/ssl/ca.pem --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem --enable-swagger-ui=true --apiserver-count=3 --audit-log-maxage=30 --audit-log-maxbackup=3 --audit-log-maxsize=100 --audit-log-path=/var/lib/audit.log --event-ttl=1h"


设置开机启动并启动apiserver组件：
```xshell
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```
ss -tanl  检查端口，6443和8080端口应该监听成功，代表apiserver安装成功。

### 4、配置和启动 kube-controller-manager

服务定义文件/usr/lib/systemd/system/kube-controller-manager.service内容为：

说明，某些文件可能已经存在，我们只要核对内容即可。
```
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/controller-manager
ExecStart=/usr/bin/kube-controller-manager \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_CONTROLLER_MANAGER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
相关配置文件配置文件/etc/kubernetes/controller-manager内容：
```
###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--address=0.0.0.0 --service-cluster-ip-range=10.254.0.0/16 --cluster-name=kubernetes --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem --root-ca-file=/etc/kubernetes/ssl/ca.pem --leader-elect=true"
```
多master时，增加“--use-service-account-credentials=true”配置。

设置开机启动并启动controller-manager
```xshell
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
```
###5、配置和启动 kube-scheduler

服务定义文件/usr/lib/systemd/system/kube-scheduler.service内容为：
```
[Unit]
Description=Kubernetes Scheduler Plugin
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/scheduler
User=kube
ExecStart=/usr/bin/kube-scheduler \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_SCHEDULER_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```
相关的配置文件/etc/kubernetes/scheduler内容为：
```
###
# kubernetes scheduler config

# default config should be adequate

# Add your own!
KUBE_SCHEDULER_ARGS="--leader-elect=true --address=0.0.0.0"
```

需要创建kube账户，否则scheduler启动不了
```xshell
useradd kube
```
设置开机启动并启动：
```xshell
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
```
注意事项：拷贝配置文件注意标点符号

###6、所有服务启动之后验证服务
首先ss -tanl查看端口：我的如下：

```
State       Recv-Q Send-Q   Local Address:Port                  Peer Address:Port              
LISTEN      0      128    192.168.102.111:6443                             *:*                  
LISTEN      0      128    192.168.102.111:2379                             *:*                  
LISTEN      0      128          127.0.0.1:2379                             *:*                  
LISTEN      0      128    192.168.102.111:2380                             *:*                  
LISTEN      0      128                  *:22                               *:*                  
LISTEN      0      128                 :::10251                           :::*                  
LISTEN      0      128                 :::10252                           :::*                  
LISTEN      0      128                 :::8080                            :::*                  
LISTEN      0      128                 :::10257                           :::*   
```
![Alt text](./img/2.png "optional title")

使用kubectl get命令获得组件信息：确保所有组件都是ok和healthy状态为true
```xshell
[root@c7test_master ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"} 
```

###7、多master时启动haproxy

1.haproxy下载

http://www.haproxy.org/#down

2.安装haproxy
查看内核版本
```
uname -r
```
解压haproxy，并安装
```
tar xf haproxy-1.7.2.tar.gz
cd haproxy-1.7.2
make TARGET=linux2628 PREFIX=/data/haproxy
make install PREFIX=/data/haproxy
```
安装成功后，查看版本
```
/data/haproxy/sbin/haproxy -v
```

复制haproxy文件到/usr/sbin下
因为下面的haproxy.init启动脚本默认会去/usr/sbin下找，当然你也可以修改，不过比较麻烦。
```
cp /data/haproxy/sbin/haproxy /usr/sbin/
```
复制haproxy脚本，到/etc/init.d下
```
cp ./examples/haproxy.init /etc/init.d/haproxy
chmod 755 /etc/init.d/haproxy
```
创建系统账号
```
useradd -r haproxy
```
创建配置文件
```
mkdir /etc/haproxy
vi /etc/haproxy/haproxy.cfg
```
文件内容(可到kubernetes_help/k8s配置文件内下载)
```
global
  #设置日志
  log 127.0.0.1 local3 info
  chroot /data/haproxy
  #用户与用户组
  user haproxy
  group haproxy
  #守护进程启动
  daemon
  #最大连接数
  maxconn 4000

defaults
  log global
  mode http
  #option httplog
  option dontlognull
  timeout connect 5000ms
  timeout client 50000ms
  timeout server 50000ms

listen stats
    bind :9090
    mode http
    balance
    stats uri /haproxy_stats
    stats auth admin:admin
    stats admin if TRUE

frontend api-https
   mode tcp
   bind :6443
   default_backend https-backend

frontend api-http
   mode tcp
   bind :8080
   default_backend http-backend

backend https-backend
    mode tcp
    server  api1  172.17.95.3:5443  check
    server  api2  172.17.95.31:5443  check
    server  api2  172.17.95.33:5443  check

backend http-backend
    mode tcp
    server  api1  172.17.95.3:7070  check
    server  api2  172.17.95.31:7070  check
    server  api2  172.17.95.33:7070  check
```
打开rsyslog配置
```
vi /etc/rsyslog.conf
```
去掉下面两行前面的#号
```
$ModLoad imudp
$UDPServerRun 514
```
并添加下面一行
``` 
local3.* /var/log/haproxy.log
```
重启rsyslog
```
systemctl restart rsyslog
```
设置开机启动并启动：
```xshell
systemctl daemon-reload
systemctl enable haproxy
```
启动haproxy
```
sudo service haproxy start
```

haproxy访问页面：http://启动主机IP:8090/haproxy_stats

至此，master节点安装完成，在创建配置文件的过程中一定要信息，如果发现报错，使用journalctl -xe -u 服务名称  查看相关报错以及查看/var/log/message查看更详细的报错情况，具体情况具体解决即可。

补充：

source <(kubectl completion bash)

执行以上命令可以执行kubectl命令的自动补全，因为kubectl太多子命令了。

## 五、安装flannel网络插件(node)

node节点需要安装flannel网络插件才能保证所有的pod在一个局域网内通信，直接使用yum安装即可，版本是0.7.1.

###1、安装flannel插件：

注意是2个node节点都需要安装，都需要修改service文件和配置文件。
```xshell
yum install  flannel -y
```
###2、修改service文件/usr/lib/systemd/system/flanneld.service其内容为：
```
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
#EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start \
  -etcd-endpoints=${FLANNEL_ETCD_ENDPOINTS} \
  -etcd-prefix=${FLANNEL_ETCD_PREFIX} \
  $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```
提示:service文件中所有的变量参数都是读取相应的配置文件里面的，所有要保证前后对应。

修改其配置文件/etc/sysconfig/flanneld 内容如下：
```
cat > /etc/sysconfig/flanneld << EOF
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://${master}:2379,https://${n1}:2379,https://${n2}:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
#FLANNEL_ETCD_PREFIX="/atomic.io/network"
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
#FLANNEL_OPTIONS=""
FLANNEL_OPTIONS="-etcd-cafile=/etc/kubernetes/ssl/ca.pem -etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem -etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem"
EOF
```
以上2步是2个node节点都需要做的。

###3、在etcd中常见网络配置信息

以下是2条命令，在任何node上创建都行，因为etcd是集群的。
```
etcdctl --endpoints=https://${master}:2379,https://${n1}:2379,https://${n2}:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mkdir /kube-centos/network

etcdctl --endpoints=https://${master}:2379,https://${n1}:2379,https://${n2}:2379 \
  --ca-file=/etc/kubernetes/ssl/ca.pem \
  --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  mk /kube-centos/network/config '{"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}'
```
###4、启动flannel服务
```xshell
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```
###5、核对相关配置信息：
```xshell
#先声明个endpoint变量，后边好调用
ETCD_ENDPOINTS='https://${master}:2379,https://${n1}:2379,https://${n2}:2379'


[root@c7test_node1 ~]# etcdctl --endpoints=${ETCD_ENDPOINTS} \
--ca-file=/etc/kubernetes/ssl/ca.pem \
--cert-file=/etc/kubernetes/ssl/kubernetes.pem \
--key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
ls /kube-centos/network/subnets
#输出
/kube-centos/network/subnets/172.30.87.0-24
/kube-centos/network/subnets/172.30.92.0-24

#说明，有几个node就有几个子网络，就有几条记录，我是2个node，分别安装了flannel插件


 [root@c7test_node1 ~]# etcdctl --endpoints=${ETCD_ENDPOINTS} \
 --ca-file=/etc/kubernetes/ssl/ca.pem \
 --cert-file=/etc/kubernetes/ssl/kubernetes.pem \
 --key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
 get /kube-centos/network/config

 #输出
 {"Network":"172.30.0.0/16","SubnetLen":24,"Backend":{"Type":"vxlan"}}

 #此处是查看主网络配置

etcdctl --endpoints=${ETCD_ENDPOINTS} --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/kubernetes.pem --key-file=/etc/kubernetes/ssl/kubernetes-key.pem get /kube-centos/network/subnets/172.30.92.0-24

 #输出

{"PublicIP":"10.10.90.106","BackendType":"vxlan","BackendData":{"VtepMAC":"26:af:ac:26:47:ad"}}

etcdctl --endpoints=${ETCD_ENDPOINTS} --ca-file=/etc/kubernetes/ssl/ca.pem --cert-file=/etc/kubernetes/ssl/kubernetes.pem --key-file=/etc/kubernetes/ssl/kubernetes-key.pem get /kube-centos/network/subnets/172.30.87.0-24

 #输出

 {"PublicIP":"10.10.90.107","BackendType":"vxlan","BackendData":{"VtepMAC":"82:6f:58:94:9f:59"}}
```
如有以上输出即代表flannel插件安装配置成功

## 六、node节点部署

### 1、问题记录

1)虚拟机时间同步不一致问题，导致etcd创建资源不成功

2)node节点无法自动创建kubelet.kubeconfig问题，这个是最严重的问题，原因是config文件没有拷贝到node的/etc/kubernetes文件夹内，因为kubelet启动调用kubelet配置文件的时候也会同时调用这个文件，具体见kubelt的servier文件配置方法，这个文件是自动生成的。如果没有自动生产，检查所有配置参数和报错，特别是config和kublet文件。

3)有关config文件并不是你从客户端拷贝过来的时候就直接可以用了，需要里面修改master地址，因为apiserver的配置启动参数绑定的地址中安全的访问地址是10.10.90.105:6443,不安全是127.0.0.1:8080，这里可以简单理解为6443是安全端口，不过只监听在master的10.10.90.105的ip上，所以要修改node中config配置文件的master地址为 10.10.90.90.105:6443，而如果你master节点同时也是node节点的话，我测试了这个形式，那么你的config文件只能用127.0.0.1:8080访问，使用6443也是不行的，也就是说本地和其他机器访问apiserver的方式不同时的，否则log中会狂报错无法连接api，这里注意一下，如果node复用了master节点需要重启scheduler和control服务。

4)Failed at step CHDIR spawning /usr/local/bin/kubelet: No such file or directory 是没有创建 /var/lib/kubelt文件夹

5)配置过程中一定要关闭防火墙，selinux，防止虚拟机重启了这些服务业自动重启。

6)1.8后面的kubelet配置文件不需要--api-servers参数，请注释掉！！

7)swap 分区请在/etc/fstab注释掉，并重启虚拟机和所有服务。

8)node节点涉及的docker服务文件的修改。

### 2、安装node准备工作

1、检查2个node节点配置文件和ssl证书是否齐全,这一步很重要。

![Alt text](./img/3.jpg "optional title")

kubelet,config,proxy文件后面内容介绍，其他文件要存在。

注意ssl里面有几个kubelet开头的文件 ，是通过过自动生成的文件。

2、配置docker的服务文件

因为需要docker联合flannel使用，所以需要修改docker的服务service文件

我们前面是flannel插件是通过yum方式安装的，修改方式如下：

修改docker的配置文件/usr/lib/systemd/system/docker.service，增加一条环境变量配置：
```
EnvironmentFile=-/run/flannel/docker
```
同时为start添加两个参数
1）--exec-opt native.cgroupdriver=systemd，这里的systemd和kubelet配置文件里面的--cgroup-drive相同
2）$DOCKER_NETWORK_OPTIONS

调整后的start
```
ExecStart=/usr/bin/dockerd --exec-opt native.cgroupdriver=systemd \
          $DOCKER_NETWORK_OPTIONS
```

修改配置参数后，重启docker服务

```xshell
systemctl daemon-reload
systemctl restart docker
```
###3、安装kubelet工具及配置

配置前我们需要现在master节点上执行如下操作，创建认证角色：

```xshell
cd /etc/kubernetes
kubectl create clusterrolebinding kubelet-bootstrap \
  --clusterrole=system:node-bootstrapper \
  --user=kubelet-bootstrap
```
created成功后，我们回到node节点操作：


至此一些必要的二进制命令文件获取完毕，下一部制作3个组件的服务程序和配置文件

我们已经获得了bin文件，开始配置相应的服务器文件

添加配置文件/etc/kubernetes/config:

更改KUBE_MASTER为master的ip

```
cat > /etc/kubernetes/config << EOF
###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=true"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=https://${master}:6443"
EOF
```
多master时：KUBE_MASTER="--master=https://192.168.1.167:6443"，IP为启动haproxy的主机IP

添加配置文件kubelet:

说明：里面的ip地址都为node节点的ip地址，其他节点相应就好，注意KUBELET_API_SERVER已经在1.8的时候不用了。注释掉。

```xshell
cd /etc/kubernetes
cat > kubelet << EOF
###
## kubernetes kubelet (minion) config
#
## The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=${myip}"
#
## The port for the info server to serve on
#KUBELET_PORT="--port=10250"
#
## You may leave this blank to use the actual hostname
##KUBELET_HOSTNAME="--hostname-override=192.168.1.159"
#
## pod infrastructure container
KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=pause-amd64:3.0"
#
## Add your own!
KUBELET_ARGS="--cgroup-driver=systemd --cluster-dns=10.254.0.2 --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig  --cert-dir=/etc/kubernetes/ssl --cluster-domain=cluster.local --hairpin-mode promiscuous-bridge --serialize-image-pulls=false"
EOF
```

```
KUBELET_POD_INFRA_CONTAINER是指定pod运行的基础镜像，必须存在，我这里直接指定的是一个本地的镜像，镜像的获取地址为：

docker pull kubernetes/pause
docker tag kubernetes/pause:latest pause-amd64:3.0

下载到本地后tag一下，方便使用，当然你也可以添加其他的公共pod基础镜像，在线地址也行，注意不要被墙就好。
```

添加kubelt的服务文件/usr/lib/systemd/system/kubelet.service

内容如下：
```
[Unit]
Description=Kubernetes Kubelet Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/kubelet
ExecStart=/usr/bin/kubelet \
            $KUBE_LOGTOSTDERR \
            $KUBE_LOG_LEVEL \
            $KUBELET_API_SERVER \
            $KUBELET_ADDRESS \
            $KUBELET_PORT \
            $KUBELET_HOSTNAME \
            $KUBE_ALLOW_PRIV \
            $KUBELET_POD_INFRA_CONTAINER \
            $KUBELET_ARGS
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
添加工作目录：不添加启动报错
```
mkdir /var/lib/kubelet
```
启动kubelt：
```
systemctl daemon-reload
systemctl enable kubelet
systemctl stop kubelet
systemctl start kubelet
systemctl status kubelet
```

###4、接受node请求

启动后，如果正确，会自动向master节点发送验证加入请求，我们在master节点操作：
```xshell
kubectl get csr 
```
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-_4hifguRKm2918NqqpbsQTcmZWIu9qMZ0Gl5FFZRbiA   23s   kubelet-bootstrap   Pending
node-csr-z0zIFZvW_643l2gOcZ1dY5ieNGw3TiW2BQJ0vF0H9M0   64s   kubelet-bootstrap   Pending


#此命令可以看到所有请求，所有为pending状态，则是需要批准的

kubectl certificate approve 节点name

#此命令可以通过请求
```
NAME                                                   AGE   REQUESTOR           CONDITION
node-csr-_4hifguRKm2918NqqpbsQTcmZWIu9qMZ0Gl5FFZRbiA   56s   kubelet-bootstrap   Approved,Issued
node-csr-z0zIFZvW_643l2gOcZ1dY5ieNGw3TiW2BQJ0vF0H9M0   97s   kubelet-bootstrap   Approved,Issued

```

显示为approved和issued状态。就正常了

命令扩展：

kubectl delete csr 节点名称 #删除单个节点的请求  

kubectl delete csr --all  #删除所有节点请求

kubectl  delete nodes  node名称  #删除加入的节点

kubectl  delete nodes --all   #删除所有节点

###5、配置kube-proxy服务

现安装个工具conntrack，跟踪并且记录连接状态的linux工具：
```xshell
yum install -y conntrack-tools
```
创建 kube-proxy 的service配置文件，路径/usr/lib/systemd/system/kube-proxy.service，内容：
```
cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
EnvironmentFile=-/etc/kubernetes/config
EnvironmentFile=-/etc/kubernetes/proxy
ExecStart=/usr/bin/kube-proxy \
        $KUBE_LOGTOSTDERR \
        $KUBE_LOG_LEVEL \
        $KUBE_MASTER \
        $KUBE_PROXY_ARGS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
添加配置文件/etc/kubernetes/proxy:内容为，需要更改为本机IP：
```
cat > /etc/kubernetes/proxy << EOF
# default config should be adequate

# Add your own!
#去掉了这一句 1107
#--hostname-override=10.10.90.106 
KUBE_PROXY_ARGS="--bind-address=${myip}--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig --cluster-cidr=10.254.0.0/16"
EOF
```

注意事项：

--hostname-override 参数值必须与 kubelet 的值一致，否则 kube-proxy 启动后会找不到该 Node，从而不会创建任何 iptables 规则；
kube-proxy 根据 --cluster-cidr 判断集群内部和外部流量，指定 --cluster-cidr 或 --masquerade-all 选项后 kube-proxy 才会对访问 Service IP 的请求做 SNAT；
--kubeconfig 指定的配置文件嵌入了 kube-apiserver 的地址、用户名、证书、秘钥等请求和认证信息；
预定义的 RoleBinding cluster-admin 将User system:kube-proxy 与 Role system:node-proxier 绑定，该 Role 授予了调用 kube-apiserver Proxy 相关 API 的权限；

启动proxy服务：
```xshell
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```
补充知识点：
```
有关node节点名称修改：

在master上通过kubectl get node 获得的列表中，Name显示的名称是通过  客户端kubelet和proxy配置文件中hostname-override配置参数定义的，修改这2个参数为你想要的名称，并且删除kubelet.kubeconfig(这个文件是master认证后客户端自动生成的，如果不删除会报node节点forbidden)文件，重新启动着2个服务，master端重新
kubectl certificate approve  name名称  就可以看到新名称。
```
 修改配置文件，不删除kubelet.kubeconfig文件会报错误：
```
kubelet_node_status.go:106] Unable to register node "node2" with API server: nodes "node2" is forbidden: node "10.10.90.107" cannot modify node "node2"
```

## 七、coredns安装


在/etc/kubernetes/yamlfile新增配置文件coredns.yaml，安装节点为master节点，node中pull镜像


###2、新增配置文件coredns.yaml配置文件内容：

注意配置文件中红色指定的image在本地仓库一定要存在，按照第一步下载下来接口，且名称要对应上
```
cat > coredns.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: coredns
  namespace: kube-system
  labels:
      kubernetes.io/cluster-service: "true"
      addonmanager.kubernetes.io/mode: Reconcile
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: Reconcile
  name: system:coredns
rules:
- apiGroups:
  - ""
  resources:
  - endpoints
  - services
  - pods
  - namespaces
  verbs:
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
    addonmanager.kubernetes.io/mode: EnsureExists
  name: system:coredns
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:coredns
subjects:
- kind: ServiceAccount
  name: coredns
  namespace: kube-system
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
  labels:
      addonmanager.kubernetes.io/mode: EnsureExists
data:
  Corefile: |
    .:53 {
        errors
        log stdout
        health
        kubernetes cluster.local 10.254.0.0/16
        prometheus
        proxy . /etc/resolv.conf
        cache 30
    }
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: coredns
  template:
    metadata:
      labels:
        k8s-app: coredns
    spec:
      serviceAccountName: coredns
      tolerations:
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
        - key: "CriticalAddonsOnly"
          operator: "Exists"
      containers:
      - name: coredns
        image: registry.docker-cn.com/coredns/coredns:0.9.10
        imagePullPolicy: IfNotPresent
        resources:
          limits:
            memory: 170Mi
          requests:
            cpu: 100m
            memory: 70Mi
        args: [ "-conf", "/etc/coredns/Corefile" ]
        volumeMounts:
        - name: config-volume
          mountPath: /etc/coredns
        ports:
        - containerPort: 53
          name: dns
          protocol: UDP
        - containerPort: 53
          name: dns-tcp
          protocol: TCP
        - containerPort: 9153
          name: metrics
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 60
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 5
      dnsPolicy: Default
      volumes:
        - name: config-volume
          configMap:
            name: coredns
            items:
            - key: Corefile
              path: Corefile
---
apiVersion: v1
kind: Service
metadata:
  name: coredns
  namespace: kube-system
  labels:
    k8s-app: coredns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "CoreDNS"
spec:
  selector:
    k8s-app: coredns
  clusterIP: 10.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
  - name: metrics
    port: 9153
    protocol: TCP
EOF
```
###3.启动dns前，修改kubernetes-csr.json文件，增加一行
```
"10.254.0.1",
```
相应的证书重新生成
###4、部署coredns
```xshell
kubectl create -f coredns.yaml
```
执行完成后开始添加服务及启动，可以通过kubectl  cluster-info查看

```
Kubernetes master is running at http://localhost:8080
CoreDNS is running at http://localhost:8080/api/v1/namespaces/kube-system/services/coredns:dns/proxy
```

![Alt text](./img/6.png "optional title")

## 八、kubernetes-dashboard安装

kubernets-dashboard顾名思义是操作面板安装，也就是可视化管理机器，同意我们用镜像结合配置文件部署。

###1、下载镜像：

```xshell
docker pull registry.docker-cn.com/kubernetesdashboarddev/kubernetes-dashboard-amd64:head
```
###2、新增部署配置文件

需要2个文件。

文件一dashboard.yaml：
```
cat > dashboard.yaml << EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    matchLabels:
      k8s-app: kubernetes-dashboard
  template:
    metadata:
      labels:
        k8s-app: kubernetes-dashboard
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      serviceAccountName: kubernetes-dashboard
      containers:
      - name: kubernetes-dashboard
        image: registry.docker-cn.com/kubernetesdashboarddev/kubernetes-dashboard-amd64:head
        resources:
          limits:
            cpu: 100m
            memory: 300Mi
          requests:
            cpu: 100m
            memory: 100Mi
        ports:
        - containerPort: 9090
        args:
        -  --apiserver-host=http://${master}:8080
        livenessProbe:
          httpGet:
            path: /
            port: 9090
          initialDelaySeconds: 30
          timeoutSeconds: 30
      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"
EOF        
```
apiserver-host为启动haproxy的主机IP

文件二：

dashboard-svc.yaml文件
```
cat > dashboard-svc.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 32017
EOF    
```
###3、分别执行2个部署文件
```xshell
kubectl create -f dashboard.yaml
kubectl create -f dashboard-svc.yaml
```
###4、查询服务
```xshell
kubectl get services kubernetes-dashboard -n kube-system
```
###5、如何访问？

定义了一个nodeport的方式，端口为32017，这样32017就监听在宿主机了，使用宿主机ip+32017,也就是10.10.90.106:32017访问了。

此时已经可以使用面板进行管理集群了，只不过面板目前少了不展示node节点的一定负载情况，比如cpu、内存等，可以按照另外一个heapster插件进行展示。

参考

1. http://www.cnblogs.com/netsa/category/1137187.html

