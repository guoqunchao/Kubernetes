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
- 01.01.简介  
https://github.com/kubernetes/kubernetes/releases  
Kubernetes的主要服务程序都可以通过直接运行二进制文件加上启动参数完成运行。在Kubernetes的Master上需要部署etcd、kube-apiserver、kube-controller-manager、kube-scheduler服务进程，在工作Node上需要部署docker、kubelet和kube-proxy服务进程。  
将Kubernetes的二进制可执行文件复制到/usr/bin目录下，然后再/usr/lib/system/system目录下为各服务创建systemd服务配置文件，这样就完成了软件的安装。要使Kubernetes正常工作，需要详细配置各个服务的启动参数。  

 - 01.02.配置免密  
```shell script
ssh-keygen #连续回车即可   
ssh-copy-id root@k8s-master01  
ssh-copy-id root@k8s-master02  
ssh-copy-id root@k8s-master03  
ssh-copy-id root@k8s-node01  
ssh-copy-id root@k8s-node02  
```
- 01.03.安装依赖
```shell script
yum install -y epel-release
yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp # ipvs 依赖 ipset；
```

 - 01.04.清空防火墙
 ```shell script
iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat
iptables -P FORWARD ACCEPT
```

 - 01.05.关闭swap
 ```shell script
swapoff -a
sed -i '/ swap / s/^\(.*\)$/# \1/g' /etc/fstab
```

 - 01.06.关闭Selinux
 ```shell script
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```
