# **通过二进制包方式安装Kubernetes for v1.17.0 集群**
##### 01.系统初始化
##### 02.创建CA证书和密钥
##### 03.部署 kubectl 命令行工具
##### 04.部署 ETCD 集群
##### 05.部署 Flannel 网络
##### 06.部署 api-haproxy 高可用组件
##### 07.部署 master 节点之 kube-apiserver 组件
##### 08.部署 master 节点之 kube-controller-manager 组件
##### 09.部署 master 节点之 kube-scheduler 组件
##### 10.部署 worker 节点之 docker 组件
##### 11.部署 worker 节点之 kubelet 组件
##### 12.部署 worker 节点之 kube-proxy 组件
##### 13.验证集群功能 完毕！

#
##### 01.系统初始化
- 简介  
https://github.com/kubernetes/kubernetes/releases  
Kubernetes的主要服务程序都可以通过直接运行二进制文件加上启动参数完成运行。在Kubernetes的Master上需要部署etcd、kube-apiserver、kube-controller-manager、kube-scheduler服务进程，在工作Node上需要部署docker、kubelet和kube-proxy服务进程。  
将Kubernetes的二进制可执行文件复制到/usr/bin目录下，然后再/usr/lib/system/system目录下为各服务创建systemd服务配置文件，这样就完成了软件的安装。要使Kubernetes正常工作，需要详细配置各个服务的启动参数。  

 - 配置免密  
```shell script
ssh-keygen #连续回车即可   
ssh-copy-id root@k8s-master01  
ssh-copy-id root@k8s-master02  
ssh-copy-id root@k8s-master03  
ssh-copy-id root@k8s-node01  
ssh-copy-id root@k8s-node02  
```
- 安装依赖
```shell script
yum install -y epel-release
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp # ipvs 依赖 ipset；
```

- 清空防火墙
 ```shell script
iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
iptables -P FORWARD ACCEPT
```

- 关闭swap
 ```shell script
swapoff -a
sed -i '/ swap / s/^\(.*\)$/# \1/g' /etc/fstab
```

- 关闭Selinux
 ```shell script
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

- 设置系统参数
```shell script
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

cp kubernetes.conf /etc/sysctl.d/kubernetes.conf
```

- 安装Docker
```shell script
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine #卸载旧版docker
yum install -y yum-utils device-mapper-persistent-data lvm2  #安装依赖包
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo #添加docker源
yum install docker-ce docker-ce-cli containerd.io -y #install dokcer engine-community
systemctl start docker && systemctl enable docker #启动docker并开机自启动
yum list docker-ce --showduplicates | sort -r #查看安装特定版本【可选】
```

- 创建目录
```shell script
mkdir -p /opt/k8s/bin
mkdir -p /opt/k8s/cert
mkdir -p /opt/etcd/cert
mkdir -p /opt/lib/etcd
```

- 检查系统内核和模块是否适合运行docker
```shell script
curl https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh > check-config.sh
```

#
##### 02.创建CA证书和密钥
- 安装cfssl工具集
```shell script
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 /opt/k8s/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 /opt/k8s/bin/cfssljson
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /opt/k8s/bin/cfssl-certinfo
chmod +x /opt/k8s/bin/*
```

- 添加命令变量
```shell script
echo 'PATH=/opt/k8s/bin:$PATH'  >>/etc/profile.d/k8s.sh
```

- 创建根证书
CA 证书是集群所有节点共享的，只需要创建一个 CA 证书，后续创建的所有证书都由它签名。
CA 配置文件用于配置根证书的使用场景 (profile) 和具体参数 (usage，过期时间、服务端认证、客户端认证、加密等)，后续在签名其它证书时需要指定特定场景。

```shell script
cd /opt/k8s/cert
vim ca-config.json
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

- 创建证书签名请求文件
```shell script
vim ca-csr.json
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
            "O": "root",
            "OU": "4Paradigm"
        }
    ]
}
```

- 生成 CA 证书和私钥，将生成的 CA 证书、秘钥文件、配置文件拷贝到所有节点的/opt/k8s/cert 目录下
```shell script
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
#
##### 03.部署 kubectl 命令行工具
- 下载kubectl 二进制文件 
```shell script
参考官网：https://kubernetes.io/docs/setup/release/
wget https://dl.k8s.io/v1.17.0/kubernetes-client-linux-amd64.tar.gz
```

- 创建证书签名请求
```shell script
cd /opt/k8s/cert/
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
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "system:masters",
            "OU": "4Paradigm"
        }
    ]
}
EOF
```

- 生成证书和私钥
```shell script
cfssl gencert -ca=/opt/k8s/cert/ca.pem -ca-key=/opt/k8s/cert/ca-key.pem -config=/opt/k8s/cert/ca-config.json -profile=kubernetes admin-csr.json | cfssljson -bare admin
```

- 创建kubeconfig文件
```shell script
kubectl config set-cluster kubernetes --certificate-authority=/opt/k8s/cert/ca.pem --embed-certs=true --server=https://192.168.111.128:8443 --kubeconfig=/root/.kube/kubectl.kubeconfig  #设置集群参数
kubectl config set-credentials kube-admin --client-certificate=/opt/k8s/cert/admin.pem --client-key=/opt/k8s/cert/admin-key.pem --embed-certs=true --kubeconfig=/root/.kube/kubectl.kubeconfig  #设置客户端认证参数
kubectl config set-context kube-admin@kubernetes --cluster=kubernetes --user=kube-admin --kubeconfig=/root/.kube/kubectl.kubeconfig  #设置上下文参数
kubectl config use-context kube-admin@kubernetes --kubeconfig=/root/.kube/kubectl.kubeconfig  #设置默认上下文
```

#
##### 04.部署 ETCD 集群
- 简介
etcd 是基于 Raft 的分布式 key-value 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。kubernetes 使用 etcd 存储所有运行数据。  
本文档介绍部署一个三节点高可用 etcd 集群的步骤：  
    ① 下载和分发 etcd 二进制文件   
    ② 创建 etcd 集群各节点的 x509 证书，用于加密客户端(如 etcdctl) 与 etcd 集群、etcd 集群之间的数据流；   
    ③ 创建 etcd 的 systemd unit 文件，配置服务参数；   
    ④ 检查集群工作状态；   
    
- 下载etcd二进制文件
```shell script
wget https://github.com/etcd-io/etcd/releases/download/v3.3.18/etcd-v3.3.18-linux-amd64.tar.gz
tar xf etcd-v3.3.18-linux-amd64.tar.gz
```

- 创建etcd证书和私钥
```shell script
cd /opt/etcd/cert
cat > etcd-csr.json <<EOF
{
    "CN": "etcd",
    "hosts": [
        "127.0.0.1",
        "192.168.10.108",
        "192.168.10.109",
        "192.168.10.110"
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
            "OU": "4Paradigm"
        }
    ]
}
# 注：hosts 字段指定授权使用该证书的 etcd 节点 IP 或域名列表，这里将 etcd 集群的三个节点 IP 都列在其中；
```

- 生成证书和私钥
```shell script
cfssl gencert -ca=/opt/k8s/cert/ca.pem -ca-key=/opt/k8s/cert/ca-key.pem -config=/opt/k8s/cert/ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
[root@k8s-master01 cert]# ls
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem
```

- 创建etcd 的systemd unit 模板及etcd 配置文件
```shell script
cat > /opt/etcd/etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
User=root
Type=notify
WorkingDirectory=/opt/lib/etcd/
ExecStart=/opt/k8s/bin/etcd \
    --data-dir=/opt/lib/etcd \
    --name etcd01 \
    --cert-file=/opt/etcd/cert/etcd.pem \
    --key-file=/opt/etcd/cert/etcd-key.pem \
    --trusted-ca-file=/opt/k8s/cert/ca.pem \
    --peer-cert-file=/opt/etcd/cert/etcd.pem \
    --peer-key-file=/opt/etcd/cert/etcd-key.pem \
    --peer-trusted-ca-file=/opt/k8s/cert/ca.pem \
    --peer-client-cert-auth \
    --client-cert-auth \
    --listen-peer-urls=https://192.168.111.128:2380 \
    --initial-advertise-peer-urls=https://192.168.111.128:2380 \
    --listen-client-urls=https://192.168.111.128:2379,http://127.0.0.1:2379 \
    --advertise-client-urls=https://192.168.111.128:2379 \
    --initial-cluster-token=etcd-cluster-0 \
    --initial-cluster=etcd01=https://192.168.111.128:2380,etcd02=https://192.168.111.129:2380,etcd03=https://192.168.111.130:2380 \
    --initial-cluster-state=new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target

cp -a etcd.service.template /etc/systemd/system/etcd.service
systemctl start etcd && systemctl enable etcd
```

- 【坑】
```shell script
[root@k8s-master01 ssl]# etcdctl member list   
client: etcd cluster is unavailable or misconfigured; error #0: x509: certificate signed by unknown authority
; error #1: x509: certificate signed by unknown authority
; error #2: x509: certificate signed by unknown authority

etcdctl工具是一个可以对etcd数据进行管理的命令行工具，这个工具在两个不同的etcd版本下的行为方式也完全不同。
export ETCDCTL_API=2
export ETCDCTL_API=3
```


#
##### 05.部署 Flannel 网络
- 简介
kubernetes 要求集群内各节点(包括 master 节点)能通过 Pod 网段互联互通。flannel 使用 vxlan 技术为各节点创建一个可以互通的 Pod 网络，使用的端口为 UDP 8472，需要开放该端口（如公有云 AWS 等）。
flannel 第一次启动时，从 etcd 获取 Pod 网段信息，为本节点分配一个未使用的 /24段地址，然后创建 flannel.1 （也可能是其它名称，如 flannel1 等） 接口。
flannel 将分配的 Pod 网段信息写入 /run/flannel/docker 文件，docker 后续使用这个文件中的环境变量设置 docker0 网桥。
flannel只需要在Node节点安装,Master节点无需安装。

```shell script
mkdir /opt/flannel/cert && cd /opt/flannel/cert
cat > flanneld-csr.json <<EOF
{
    "CN": "flanneld",
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
            "OU": "4Paradigm"
        }
    ]
}
EOF


[root@kube-master cert]# cfssl gencert -ca=/opt/k8s/cert/ca.pem \
-ca-key=/opt/k8s/cert/ca-key.pem \
-config=/opt/k8s/cert/ca-config.json \
-profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld

[root@k8s-master01 cert]# ll flanneld*
-rw-r--r-- 1 root root 1001 Dec 23 16:33 flanneld.csr
-rw-r--r-- 1 root root  282 Dec 23 16:31 flanneld-csr.json
-rw------- 1 root root 1675 Dec 23 16:33 flanneld-key.pem
-rw-r--r-- 1 root root 1399 Dec 23 16:33 flanneld.pem
```

- 创建flanneld的systemd unit文件
```shell script
[root@k8s-master01 opt]# vim /etc/systemd/system/flanneld.service
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/opt/k8s/bin/flanneld \
-etcd-cafile=/opt/k8s/cert/ca.pem \
-etcd-certfile=/opt/flannel/cert/flanneld.pem \
-etcd-keyfile=/opt/flannel/cert/flanneld-key.pem \
-etcd-endpoints=https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379 \
-etcd-prefix=/coreos.com/network \
-iface=ens33
ExecStartPost=/opt/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```

```shell script
ETCDCTL_API=2 etcdctl \
--endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
--ca-file=/opt/k8s/cert/ca.pem \
--cert-file=/opt/flannel/cert/flanneld.pem \
--key-file=/opt/flannel/cert/flanneld-key.pem \
set /atomic.io/network/config '{"Network":"10.100.0.0/16","SubnetLen": 24, "Backend": {"Type": "vxlan"}}'


ETCDCTL_API=3 etcdctl \
--endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
--cacert=/opt/k8s/cert/ca.pem \
--cert=/opt/flannel/cert/flanneld.pem \
--key=/opt/flannel/cert/flanneld-key.pem \
mk /coreos.com/network/config '{"Network":"10.100.0.0/16","SubnetLen": 24, "Backend": {"Type": "vxlan","VNI": 1}}'
```

- 区别
```shell script
[root@k8s-master01 pkg]# ETCDCTL_API=2 etcdctl \
  --endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
  --ca-file=/opt/k8s/cert/ca.pem \
  --cert-file=/opt/flannel/cert/flanneld.pem \
  --key-file=/opt/flannel/cert/flanneld-key.pem \
  member list
1509ba27a4abd563: name=etcd02 peerURLs=https://192.168.111.129:2380 clientURLs=https://192.168.111.129:2379 isLeader=true
bb363834919755be: name=etcd03 peerURLs=https://192.168.111.130:2380 clientURLs=https://192.168.111.130:2379 isLeader=false
f38acfdc443da778: name=etcd01 peerURLs=https://192.168.111.128:2380 clientURLs=https://192.168.111.128:2379 isLeader=false


[root@k8s-master01 pkg]# ETCDCTL_API=3 etcdctl \
  --endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
  --cacert=/opt/k8s/cert/ca.pem \
  --cert=/opt/flannel/cert/flanneld.pem \
  --key=/opt/flannel/cert/flanneld-key.pem \
  member list
1509ba27a4abd563, started, etcd02, https://192.168.111.129:2380, https://192.168.111.129:2379
bb363834919755be, started, etcd03, https://192.168.111.130:2380, https://192.168.111.130:2379
f38acfdc443da778, started, etcd01, https://192.168.111.128:2380, https://192.168.111.128:2379
```

- 查看插入数据
```shell script
ETCDCTL_API=3 etcdctl \
--endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
--cacert=/opt/k8s/cert/ca.pem \
--cert=/opt/flannel/cert/flanneld.pem \
--key=/opt/flannel/cert/flanneld-key.pem \
get /coreos.com/network/config
```

- 获取所有key和value
```shell script
ETCDCTL_API=3 etcdctl \
--endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
--cacert=/opt/k8s/cert/ca.pem \
--cert=/opt/flannel/cert/flanneld.pem \
--key=/opt/flannel/cert/flanneld-key.pem \
--prefix --keys-only=false get /
```

- 查看已分配的Pod子网段列表
```shell script
ETCDCTL_API=2 etcdctl \
  --endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
  --ca-file=/opt/k8s/cert/ca.pem \
  --cert-file=/opt/flannel/cert/flanneld.pem \
  --key-file=/opt/flannel/cert/flanneld-key.pem \
  ls /coreos.com/network/subnets
/coreos.com/network/subnets/172.10.51.0-24
/coreos.com/network/subnets/172.10.52.0-24
```

- 查看某一 Pod 网段对应的节点 IP 和 flannel 接口地址
```shell script
ETCDCTL_API=2 etcdctl \
  --endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
  --ca-file=/opt/k8s/cert/ca.pem \
  --cert-file=/opt/flannel/cert/flanneld.pem \
  --key-file=/opt/flannel/cert/flanneld-key.pem \
get /atomic.io/network/subnets/10.30.22.0-24
```


#
##### 06.部署 api-haproxy 高可用组件
```shell script
yum install haproxy -y
cat << EOF > /etc/haproxy/haproxy.cfg
global
    log         127.0.0.1 local2
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

defaults
    mode                    tcp
    log                     global
    retries                 3
    timeout connect         10s
    timeout client          1m
    timeout server          1m

#frontend kube-apiserver
#    bind *:8443 # 指定前端端口
#    mode tcp
#    default_backend master
#
#backend master # 指定后端机器及端口，负载方式为轮询
#    balance roundrobin
#    server k8s-master01  192.168.111.128:6443 check maxconn 2000
#    server k8s-master02  192.168.111.129:6443 check maxconn 2000
#    server k8s-master03  192.168.111.130:6443 check maxconn 2000

listen kube-master
    bind 0.0.0.0:8443
    mode tcp
    option tcplog
    balance source
    server k8s-master01  192.168.111.128:6443 check maxconn 2000
    server k8s-master02  192.168.111.129:6443 check maxconn 2000
    server k8s-master03  192.168.111.130:6443 check maxconn 2000

listen admin_stats
    bind 0.0.0.0:10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:admin
    stats hide-version
    stats admin if TRUE
EOF

systemctl enable haproxy
systemctl start haproxy
```


##### 07.部署 master 节点之 kube-apiserver 组件
- 创建 kubernetes 证书和私钥
```shell script
[root@k8s-master01 cert]# cd /opt/k8s/cert/
[root@k8s-master01 cert]# vim kubernetes-csr.json
{
    "CN": "kubernetes",
    "hosts": [
        "127.0.0.1",
        "192.168.111.128",
        "192.168.111.129",
        "192.168.111.130",
        "192.168.10.10",
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
          "OU": "4Paradigm"
        }
    ]
}

```

- 生成证书和私钥
```shell script
[root@k8s-master01 cert]# cfssl gencert -ca=/opt/k8s/cert/ca.pem \
  -ca-key=/opt/k8s/cert/ca-key.pem \
  -config=/opt/k8s/cert/ca-config.json \
  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes

[root@k8s-master01 cert]# ls kubernetes*
kubernetes.csr  kubernetes-csr.json  kubernetes-key.pem  kubernetes.pem
```

- 创建加密配置文件
```shell script
# 产生一个用来加密Etcd 的 Key:
[root@k8s-master01 bin]# head -c 32 /dev/urandom | base64
YmMwxm94RIG1PJoGnZWLnmrtne2YXyi76luqo5RV0Pw=
# 注意：每台master节点需要用一样的 Key

[root@k8s-master01 k8s]# vim /opt/k8s/encryption-config.yaml
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: YmMwxm94RIG1PJoGnZWLnmrtne2YXyi76luqo5RV0Pw=
      - identity: {}
```

- 将生成的证书和私钥文件、加密配置文件拷贝到master 节点的/opt/k8s目录下


- 创建 kube-apiserver systemd unit 模板文件
```shell script
[root@k8s-master01 opt]# vim /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
ExecStart=/opt/k8s/bin/kube-apiserver \
--enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
--anonymous-auth=false \
--experimental-encryption-provider-config=/opt/k8s/encryption-config.yaml \
--advertise-address=192.168.111.128 \  ##nodeip##
--bind-address=192.168.111.128 \  ##nodeip##
--insecure-port=0 \
--authorization-mode=Node,RBAC \
--runtime-config=api/all=true \
--enable-bootstrap-token-auth \
--service-cluster-ip-range=10.101.0.0/16 \ 
--service-node-port-range=1-32767 \
--tls-cert-file=/opt/k8s/cert/kubernetes.pem \
--tls-private-key-file=/opt/k8s/cert/kubernetes-key.pem \
--client-ca-file=/opt/k8s/cert/ca.pem \
--kubelet-client-certificate=/opt/k8s/cert/kubernetes.pem \
--kubelet-client-key=/opt/k8s/cert/kubernetes-key.pem \
--service-account-key-file=/opt/k8s/cert/ca-key.pem \
--etcd-cafile=/opt/k8s/cert/ca.pem \
--etcd-certfile=/opt/k8s/cert/kubernetes.pem \
--etcd-keyfile=/opt/k8s/cert/kubernetes-key.pem \
--etcd-servers=https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379 \
--enable-swagger-ui=true \
--allow-privileged=true \
--apiserver-count=3 \
--audit-log-maxage=30 \
--audit-log-maxbackup=3 \
--audit-log-maxsize=100 \
--audit-log-path=/var/log/kube-apiserver-audit.log \
--event-ttl=1h \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/opt/log/kubernetes \
--v=2
Restart=on-failure
RestartSec=5
Type=notify
User=root
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

- 打印kube-apiserver写入etcd的数据
```shell script
ETCDCTL_API=3 etcdctl \
--endpoints="https://192.168.111.128:2379,https://192.168.111.129:2379,https://192.168.111.130:2379" \
--cacert=/opt/k8s/cert/ca.pem \
--cert=/opt/flannel/cert/flanneld.pem \
--key=/opt/flannel/cert/flanneld-key.pem \
--prefix --keys-only=false get /
```

- 检查集群信息
```shell script
[root@k8s-master01 opt]# kubectl cluster-info
Kubernetes master is running at https://192.168.111.128:8443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

[root@k8s-master01 opt]# kubectl get cs
NAME                 STATUS      MESSAGE                                                                                     ERROR
controller-manager   Unhealthy   Get http://127.0.0.1:10252/healthz: dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get http://127.0.0.1:10251/healthz: dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-2               Healthy     {"health":"true"}                                                                           
etcd-0               Healthy     {"health":"true"}                                                                           
etcd-1               Healthy     {"health":"true"}
```

- 检查 kube-apiserver 监听的端口
```shell script
[root@k8s-master01 opt]#  ss -nutlp |grep apiserver
tcp    LISTEN     0      128    192.168.111.128:6443                  *:*                   users:(("kube-apiserver",pid=18823,fd=6))
```

- 授予 kubernetes 证书访问 kubelet API 的权限
```shell script
# 在执行kubectl exec run logs 等命令时，apiserver会转发到kubelet。这里定义RBAC规则，授权apiserver调用kubeletAPI；
[root@k8s-master01 opt]# kubectl create clusterrolebinding kube-apiserver:kubelet-apis --clusterrole=system:kubelet-api-admin --user kubernetes
clusterrolebinding.rbac.authorization.k8s.io/kube-apiserver:kubelet-apis created
```

##### 08.部署 master 节点之 kube-controller-manager 组件
该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。
- 创建 kube-controller-manager 证书和私钥
```shell script
[root@k8s-master01 opt]# cd /opt/k8s/cert/
[root@k8s-master01 opt]# vim kube-controller-manager-csr.json
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
        "127.0.0.1",
        "192.168.111.128",
        "192.168.111.129",
        "192.168.111.130"
    ],
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "system:kube-controller-manager",
            "OU": "4Paradigm"
        }
    ]
}

# 注：
# hosts 列表包含所有 kube-controller-manager 节点 IP；
# CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予kube-controller-manager 工作所需的权限。

```

- 生成证书和私钥
```shell script
[root@k8s-master01 cert]# cfssl gencert -ca=/opt/k8s/cert/ca.pem \
  -ca-key=/opt/k8s/cert/ca-key.pem \
  -config=/opt/k8s/cert/ca-config.json \
  -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
2019/12/25 16:02:58 [INFO] generate received request
2019/12/25 16:02:58 [INFO] received CSR
2019/12/25 16:02:58 [INFO] generating key: rsa-2048
2019/12/25 16:02:58 [INFO] encoded CSR
2019/12/25 16:02:58 [INFO] signed certificate with serial number 39641631925030822703938202066387462489322509807
2019/12/25 16:02:58 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

[root@k8s-master01 cert]# ls kube-controller-manager*
kube-controller-manager.csr  kube-controller-manager-csr.json  kube-controller-manager-key.pem  kube-controller-manager.pem
```
- 创建 kube-controller-manager.kubeconfig 文件
```shell script
[root@k8s-master01 cert]# kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/k8s/cert/ca.pem \
  --embed-certs=true \
  --server=https://192.168.111.128:8443 \
  --kubeconfig=/root/.kube/kube-controller-manager.kubeconfig
Cluster "kubernetes" set.

[root@k8s-master01 cert]# kubectl config set-credentials system:kube-controller-manager \
> --client-certificate=/opt/k8s/cert/kube-controller-manager.pem \
> --client-key=/opt/k8s/cert/kube-controller-manager-key.pem \
> --embed-certs=true \
> --kubeconfig=/root/.kube/kube-controller-manager.kubeconfig
User "system:kube-controller-manager" set.

[root@k8s-master01 cert]# kubectl config set-context system:kube-controller-manager@kubernetes \
> --cluster=kubernetes \
> --user=system:kube-controller-manager \
> --kubeconfig=/root/.kube/kube-controller-manager.kubeconfig
Context "system:kube-controller-manager@kubernetes" created.

[root@k8s-master01 cert]# kubectl config use-context system:kube-controller-manager@kubernetes --kubeconfig=/root/.kube/kube-controller-manager.kubeconfig
Switched to context "system:kube-controller-manager@kubernetes".
```
- 验证kube-controller-manager.kubeconfig 文件
```shell script
[root@k8s-master01 cert]# ls /root/.kube/kube-controller-manager.kubeconfig
[root@k8s-master01 cert]# kubectl config view --kubeconfig=/root/.kube/kube-controller-manager.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.111.128:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-controller-manager
  name: system:kube-controller-manager@kubernetes
current-context: system:kube-controller-manager@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-controller-manager
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

```

- 创建和分发 kube-controller-manager systemd unit 文件
```shell script
# [root@k8s-master01 cert]# mkdir /opt/controller_manager
# [root@k8s-master01 cert]# cd /opt/controller_manager
[root@k8s-master01 controller_manager]# vim /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/k8s/bin/kube-controller-manager \
--port=0 \
--secure-port=10252 \
--bind-address=127.0.0.1 \
#--kubeconfig=/opt/k8s/kube-controller-manager.kubeconfig \
--kubeconfig=/root/.kube/kube-controller-manager.kubeconfig \
--service-cluster-ip-range=10.101.0.0/16 \
--cluster-name=kubernetes \
--cluster-signing-cert-file=/opt/k8s/cert/ca.pem \
--cluster-signing-key-file=/opt/k8s/cert/ca-key.pem \
--experimental-cluster-signing-duration=8760h \
--root-ca-file=/opt/k8s/cert/ca.pem \
--service-account-private-key-file=/opt/k8s/cert/ca-key.pem \
--leader-elect=true \
--feature-gates=RotateKubeletServerCertificate=true \
--controllers=*,bootstrapsigner,tokencleaner \
--horizontal-pod-autoscaler-use-rest-clients=true \
--horizontal-pod-autoscaler-sync-period=10s \
--tls-cert-file=/opt/k8s/cert/kube-controller-manager.pem \
--tls-private-key-file=/opt/k8s/cert/kube-controller-manager-key.pem \
--use-service-account-credentials=true \
--alsologtostderr=true \
--logtostderr=false \
--log-dir=/var/log/kubernetes \
--v=2
Restart=on
Restart=on-failure
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target

[root@k8s-master01 controller_manager]# systemctl start kube-controller-manager
[root@k8s-master01 controller_manager]# systemctl enable kube-controller-manager
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service to /etc/systemd/system/kube-controller-manager.service.


# 注：
# --port=0：关闭监听 http /metrics 的请求，同时 --address 参数无效，--bind-address 参数有效；
# --secure-port=10252、--bind-address=0.0.0.0: 在所有网络接口监听 10252 端口的 https /metrics 请求；
# --kubeconfig：指定 kubeconfig 文件路径，kube-controller-manager 使用它连接和验证 kube-apiserver；
# --cluster-signing-*-file：签名 TLS Bootstrap 创建的证书；
# --experimental-cluster-signing-duration：指定 TLS Bootstrap 证书的有效期；
# --root-ca-file：放置到容器 ServiceAccount 中的 CA 证书，用来对 kube-apiserver 的证书进行校验；
# --service-account-private-key-file：签名 ServiceAccount 中 Token 的私钥文件，必须和 kube-apiserver 的 --service-account-key-file 指定的公钥文件配对使用；
# --service-cluster-ip-range ：指定 Service Cluster IP 网段，必须和 kube-apiserver 中的同名参数一致；
# --leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；
# --feature-gates=RotateKubeletServerCertificate=true：开启 kublet server 证书的自动更新特性；
# --controllers=*,bootstrapsigner,tokencleaner：启用的控制器列表，tokencleaner 用于自动清理过期的 Bootstrap token；
# --horizontal-pod-autoscaler-*：custom metrics 相关参数，支持 autoscaling/v2alpha1；
# --tls-cert-file、--tls-private-key-file：使用 https 输出 metrics 时使用的 Server 证书和秘钥；
# --use-service-account-credentials=true:
# User=k8s：使用 k8s 账户运行；这里使用root
```

- 启动
```shell script
[root@k8s-master03 k8s]# systemctl start kube-controller-manager
[root@k8s-master03 k8s]# systemctl enable kube-controller-manager
Created symlink from /etc/systemd/system/multi-user.target.wants/kube-controller-manager.service to /etc/systemd/system/kube-controller-manager.service.
```

- 查看当前的 leader
```yaml
[root@k8s-master01 opt]# kubectl get endpoints kube-controller-manager --namespace=kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-master01_89863365-2bd1-4a8f-a454-db7f7f64594b","leaseDurationSeconds":15,"acquireTime":"2019-12-25T08:17:02Z","renewTime":"2019-12-25T08:29:25Z","leaderTransitions":0}'
  creationTimestamp: "2019-12-25T08:17:02Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "1931"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 285fcb7d-983b-4a7f-9c00-ad01ca9b4194

```
#

##### 09.部署 master 节点之 kube-scheduler 组件
#　该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。
- 创建 kube-scheduler 证书和私钥
```yaml
[root@k8s-master01 cert]# cd /opt/k8s/cert/
[root@k8s-master01 cert]# vim kube-scheduler-csr.json
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "192.168.111.128",
      "192.168.111.129",
      "192.168.111.130"
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
        "O": "system:kube-scheduler",
        "OU": "4Paradigm"
      }
    ]
}
```

- 生成证书和私钥
```shell script
[root@k8s-master01 cert]# cfssl gencert -ca=/opt/k8s/cert/ca.pem \
> -ca-key=/opt/k8s/cert/ca-key.pem \
> -config=/opt/k8s/cert/ca-config.json \
> -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
2019/12/25 16:36:17 [INFO] generate received request
2019/12/25 16:36:17 [INFO] received CSR
2019/12/25 16:36:17 [INFO] generating key: rsa-2048
2019/12/25 16:36:18 [INFO] encoded CSR
2019/12/25 16:36:18 [INFO] signed certificate with serial number 116837778336782174005556793697093783379336558872
2019/12/25 16:36:18 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
[root@k8s-master01 cert]# ls *schedu*
kube-scheduler.csr  kube-scheduler-csr.json  kube-scheduler-key.pem  kube-scheduler.pem
```

- 创建kube-scheduler.kubeconfig文件
```shell script
[root@k8s-master01 cert]# kubectl config set-cluster kubernetes \
> --certificate-authority=/opt/k8s/cert/ca.pem \
> --embed-certs=true \
> --server=https://192.168.111.128:8443 \
> --kubeconfig=/root/.kube/kube-scheduler.kubeconfig
Cluster "kubernetes" set.

[root@k8s-master01 cert]# kubectl config set-credentials system:kube-scheduler \
> --client-certificate=/opt/k8s/cert/kube-scheduler.pem \
> --client-key=/opt/k8s/cert/kube-scheduler-key.pem \
> --embed-certs=true \
> --kubeconfig=/root/.kube/kube-scheduler.kubeconfig
User "system:kube-scheduler" set.

[root@k8s-master01 cert]# kubectl config set-context system:kube-scheduler@kubernetes \
> --cluster=kubernetes \
> --user=system:kube-scheduler \
> --kubeconfig=/root/.kube/kube-scheduler.kubeconfig
Context "system:kube-scheduler@kubernetes" created.

[root@k8s-master01 cert]# kubectl config use-context system:kube-scheduler@kubernetes --kubeconfig=/root/.kube/kube-scheduler.kubeconfig
Switched to context "system:kube-scheduler@kubernetes".

[root@k8s-master01 cert]# ls /root/.kube/kube-scheduler.kubeconfig
/root/.kube/kube-scheduler.kubeconfig

[root@k8s-master01 cert]# kubectl config view --kubeconfig=/root/.kube/kube-scheduler.kubeconfig
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://192.168.111.128:8443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-scheduler
  name: system:kube-scheduler@kubernetes
current-context: system:kube-scheduler@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-scheduler
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED

```


- 创建kube-scheduler systemd unit 文件
```shell script
[root@k8s-master01 .kube]# vim /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=kube-apiserver.service
Requires=kube-apiserver.service

[Service]
ExecStart=/opt/k8s/bin/kube-scheduler \
  --address=127.0.0.1 \
  --kubeconfig=/root/.kube/kube-scheduler.kubeconfig \
  --leader-elect=true \
  --alsologtostderr=true \
  --logtostderr=false \
  --log-dir=/var/log/kubernetes \
  --v=2
Restart=on-failure
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
[root@k8s-master01 .kube]# systemctl start kube-scheduler
[root@k8s-master01 .kube]# systemctl enable kube-scheduler
```

- 查看当前的 leader
```shell script
[root@k8s-master01 .kube]# kubectl get endpoints kube-scheduler --namespace=kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"k8s-master01_35573124-e90f-4b33-b85c-2685c7b1d3af","leaseDurationSeconds":15,"acquireTime":"2019-12-25T08:52:06Z","renewTime":"2019-12-25T08:58:55Z","leaderTransitions":0}'
  creationTimestamp: "2019-12-25T08:52:06Z"
  name: kube-scheduler
  namespace: kube-system
  resourceVersion: "4615"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler
  uid: 596eba28-d8ed-4a14-bc5c-2b274dfd8d44
```

##### 10.部署 worker 节点之 docker 组件
```shell script
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine #卸载旧版docker
yum install -y yum-utils device-mapper-persistent-data lvm2  #安装依赖包
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo #添加docker源
yum install docker-ce docker-ce-cli containerd.io -y #install dokcer engine-community
yum list docker-ce --showduplicates | sort -r #查看安装特定版本【可选】
systemctl start docker && systemctl enable docker #启动docker并开机自启动

[root@k8s-node02 flannel]# vim /usr/lib/systemd/system/docker.service
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
BindsTo=containerd.service
After=network-online.target firewalld.service containerd.service
Wants=network-online.target
Requires=docker.socket

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
#ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
Environment="PATH=/opt/k8s/bin:/bin:/sbin:/usr/bin:/usr/sbin"
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --log-level=error $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

# Note that StartLimit* options were moved from "Service" to "Unit" in systemd 229.
# Both the old, and new location are accepted by systemd 229 and up, so using the old location
# to make them work for either version of systemd.
StartLimitBurst=3

# Note that StartLimitInterval was renamed to StartLimitIntervalSec in systemd 230.
# Both the old, and new name are accepted by systemd 230 and up, so using the old name to make
# this option work for either version of systemd.
StartLimitInterval=60s

# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity

# Comment TasksMax if your systemd version does not support it.
# Only systemd 226 and above support this option.
TasksMax=infinity

# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes

# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target


[root@k8s-node02 flannel]# cat docker subnet.env 
DOCKER_OPT_BIP="--bip=10.100.14.1/24"
DOCKER_OPT_IPMASQ="--ip-masq=true"
DOCKER_OPT_MTU="--mtu=1450"
DOCKER_NETWORK_OPTIONS=" --bip=10.100.14.1/24 --ip-masq=true --mtu=1450"
FLANNEL_NETWORK=10.100.0.0/16
FLANNEL_SUBNET=10.100.14.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
```




##### 11.部署 worker 节点之 kubelet 组件
##### 12.部署 worker 节点之 kube-proxy 组件
##### 13.验证集群功能 完毕！