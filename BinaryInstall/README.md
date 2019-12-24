# **通过二进制包方式安装Kubernetes for v1.17.0 集群**
##### 01.系统初始化
##### 02.创建CA证书和密钥
##### 03.部署 kubectl 命令行工具
##### 04.部署 ETCD 集群
##### 05.部署 Flannel 网络
##### 06.部署 master 节点之 kube-apiserver 组件
##### 07.部署 master 节点之 kube-controller-manager 组件
##### 08.部署 master 节点之 kube-scheduler 组件
##### 09.部署 worker 节点之 docker 组件
##### 10.部署 worker 节点之 kubelet 组件
##### 11.部署 worker 节点之 kube-proxy 组件
##### 12.验证集群功能 完毕！

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

