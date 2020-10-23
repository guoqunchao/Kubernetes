
## Kubernetes使用kubeadm部署高可用集群
**1. Kubeadm原理简述**  
Kubeadm工具的出发点很简单，就是尽可能简单的部署一个生产可用的Kubernetes集群。实际也确实很简单，只需要两条命令：
```shell
# 创建一个 Master 节点
$ kubeadm init

# 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master 节点的 IP 和端口 >
```
**2. 执行 kubeadm init时：**  
 - 自动化的集群机器合规检查。
 - 自动化生成集群运行所需的各类证书及各类配置，并将Master节点信息保存在名为cluster-info的ConfigMap中。
 - 通过static Pod方式，运行API server, controller manager 、scheduler及etcd组件。
 - 生成Token以便其他节点加入集群。

**3. 执行 kubeadm join时：**  
 - 节点通过token访问kube-apiserver，获取cluster-info中信息，主要是apiserver的授权信息（节点信任集群）。
 - 通过授权信息，kubelet可执行TLS bootstrapping，与apiserver真正建立互信任关系（集群信任节点）。

**简单来说，kubeadm做的事就是把大部分组件都容器化，通过StaticPod方式运行，并自动化了大部分的集群配置及认证等工作，简单几步即可搭建一个可用Kubernetes的集群。**  

这里有个问题，为什么不把kubelet组件也容器化呢，是因为，kubelet在配置容器网络、管理容器数据卷时，都需要直接操作宿主机，而如果现在 kubelet 本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦。比如，容器内要做NFS的挂载，需要kubelet先在宿主机执行mount挂载NFS。如果kubelet运行在容器中问题来了，如果kubectl运行在容器中，要操作宿主机的Mount Namespace是非常复杂的。所以，kubeadm选择把kubelet运行直接运行在宿主机中，使用容器部署其他Kubernetes组件。所以，Kubeadm部署要安装的组件有Kubeadm、kubelet、kubectl三个。

上面说的是kubeadm部署方式的一般步骤，kubeadm部署是可以自由定制的，包括要容器化哪些组件，所用的镜像，是否用外部etcd，是否使用用户证书认证等以及集群的配置等等，都是可以灵活定制的，这也是kubeadm能够快速部署一个高可用的集群的基础。详细的说明可以参考官方Reference。但是，kubeadm最重要的作用还是解决集群部署问题，而不是集群配置管理的问题，官方也建议把Kubeadm作为一个基础工具，在其上层再去量身定制适合自己的集群的管理工具（例如minikube）。

**4. Kubeadm部署一个高可用集群**  
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191126112453987.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3F1bmNoYW9fQmxvZw==,size_16,color_FFFFFF,t_70)

另外一种方式是，使用独立的Etcd集群，不与Master节点混布：![在这里插入图片描述](https://img-blog.csdnimg.cn/20191126112531509.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3F1bmNoYW9fQmxvZw==,size_16,color_FFFFFF,t_70)

两种方式的相同之处在于都提供了控制平面的冗余，实现了集群高可以用，区别在于：

Etcd混布方式：  

 - 所需机器资源少
 - 部署简单，利于管理
 - 容易进行横向扩展
 - 风险大，一台宿主机挂了，master和etcd就都少了一套，集群冗余度受到的影响比较大。

Etcd独立部署方式：  

 - 所需机器资源多（按照Etcd集群的奇数原则，这种拓扑的集群关控制平面最少就要6台宿主机了）。
 - 部署相对复杂，要独立管理etcd集群和和master集群。
 - 解耦了控制平面和Etcd，集群风险小健壮性强，单独挂了一台master或etcd对集群的影响很小。

**5. 部署准备**    
由于机器资源不足，下面的部署测试，只会以混布的方式部署一个1*haproxy，3*master，2*node，共5台机器的集群，实际上由于etcd选举要过半数，至少要3台master节点才能构成高可用，在生产环境，还是要根据实际情况，尽量选择风险低的拓扑结构。

 - 机器：
```shell
192.168.111.133 k8s-api-proxy 
192.168.111.128 k8s-master01
192.168.111.129 k8s-master02
192.168.111.130 k8s-master03
192.168.111.131 k8s-node01
192.168.111.132 k8s-node02
```

 - 系统版本：
```shell
[root@k8s-master01 ~]# cat /etc/redhat-release 
CentOS Linux release 7.7.1908 (Core)
[root@k8s-master01 ~]# uname -r
3.10.0-1062.4.3.el7.x86_64
```
 - 集群版本
```shell
[root@k8s-master02 ~]# docker version
Client: Docker Engine - Community
 Version:           19.03.5
[root@k8s-master02 ~]# rpm -qa kubelet*
kubelet-1.16.3-0.x86_64
[root@k8s-master02 ~]# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"16", GitVersion:"v1.16.3", GitCommit:"b3cbbae08ec52a7fc73d334838e18d17e8512749", GitTreeState:"clean", BuildDate:"2019-11-13T11:20:25Z", GoVersion:"go1.12.12", Compiler:"gc", Platform:"linux/amd64"}
[root@k8s-master02 kubernetes]# docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-apiserver            v1.16.3             df60c7526a3d        12 days ago         217MB
k8s.gcr.io/kube-proxy                v1.16.3             9b65a0f78b09        12 days ago         86.1MB
k8s.gcr.io/kube-controller-manager   v1.16.3             bb16442bcd94        12 days ago         163MB
k8s.gcr.io/kube-scheduler            v1.16.3             98fecf43a54f        12 days ago         87.3MB
k8s.gcr.io/etcd                      3.3.15-0            b2756210eeab        2 months ago        247MB
k8s.gcr.io/coredns                   1.6.2               bf261d157914        3 months ago        44.1MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        23 months ago       742kB
```
**6. 配置主机**    

 - 关闭Selinux
```shell
$ setenforce  0
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```
 - 关闭防火墙
```shell
$ systemctl stop firewalld
$ systemctl disable firewalld
```
 - 关闭Swap
```shell
$ swapoff -a
$ sed -i s#`egrep "swap" /etc/fstab |awk '{print $1}'`#"\#`egrep "swap" /etc/fstab |awk '{print $1}'`"# /etc/fstab
```
 - 修改内核参数
```shell
# 修改下面内核参数，否则请求数据经过iptables的路由可能有问题
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

 - 添加hosts
```shell
192.168.111.133 k8s-api-proxy
192.168.111.128 k8s-master01
192.168.111.129 k8s-master02
192.168.111.130 k8s-master03
192.168.111.131 k8s-node01
192.168.111.132 k8s-node02
```
**7. 安装docker**  
除了k8s-api-proxy之外所有节点执行；
```shell
yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine #卸载旧版docker
yum install -y yum-utils device-mapper-persistent-data lvm2  #安装依赖包
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo #添加docker源
yum install docker-ce docker-ce-cli containerd.io -y #install dokcer engine-community
yum list docker-ce --showduplicates | sort -r #查看安装特定版本【可选】
systemctl start docker && systemctl enable docker #启动docker并开机自启动
```

**8. 安装kubectl**
除了k8s-api-proxy之外所有节点执行；
```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl #下载命令
chmod +x ./kubectl #使kubectl二进制可执行
mv ./kubectl /usr/local/bin/kubectl #将二进制文件移动到PATH中
```

**9. 安装kubeadm和kubelet** 
```shell
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
yum install -y kubelet kubeadm
systemctl enable kubelet && systemctl start kubelet
```

**10. 安装api-haproxy**  
```shell
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

frontend kube-apiserver
    bind *:6443 # 指定前端端口
    mode tcp
    default_backend master

backend master # 指定后端机器及端口，负载方式为轮询
    balance roundrobin
    server k8s-master01  192.168.111.128:6443 check maxconn 2000
    server k8s-master02  192.168.111.129:6443 check maxconn 2000
    server k8s-master03  192.168.111.130:6443 check maxconn 2000
EOF

systemctl enable haproxy
systemctl start haproxy

```
**11. 先拉取镜像**  
由于需要下载gcr官方源，无法直接下载；这里做中转。
```shell
docker pull inkwhite/pause:3.1
docker pull inkwhite/coredns:1.6.2
docker pull inkwhite/etcd:3.3.15-0
docker pull inkwhite/kube-scheduler:v1.16.3
docker pull inkwhite/kube-controller-manager:v1.16.3
docker pull inkwhite/kube-apiserver:v1.16.3
docker pull inkwhite/kube-proxy:v1.16.3

docker tag inkwhite/pause:3.1 k8s.gcr.io/pause:3.1
docker tag inkwhite/coredns:1.6.2 k8s.gcr.io/coredns:1.6.2
docker tag inkwhite/etcd:3.3.15-0 k8s.gcr.io/etcd:3.3.15-0
docker tag inkwhite/kube-scheduler:v1.16.3 k8s.gcr.io/kube-scheduler:v1.16.3
docker tag inkwhite/kube-controller-manager:v1.16.3 k8s.gcr.io/kube-controller-manager:v1.16.3
docker tag inkwhite/kube-apiserver:v1.16.3 k8s.gcr.io/kube-apiserver:v1.16.3
docker tag inkwhite/kube-proxy:v1.16.3 k8s.gcr.io/kube-proxy:v1.16.3
```
**12. master节点初始化**  
这里的--control-plane-endpoint 填写api-proxy主机的IP；执行完保留输出信息；
```shell
kubeadm init --control-plane-endpoint "192.168.111.133:6443" --pod-network-cidr=10.100.0.0/16 --service-cidr=10.101.0.0/16 --upload-certs

# Your Kubernetes control-plane has initialized successfully!
#
# To start using your cluster, you need to run the following as a regular user:
#
#   mkdir -p $HOME/.kube
#   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
#   sudo chown $(id -u):$(id -g) $HOME/.kube/config
#
# You should now deploy a pod network to the cluster.
# Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
#   https://kubernetes.io/docs/concepts/cluster-administration/addons/
#
# You can now join any number of the control-plane node running the following command on each as root:
#
#   kubeadm join 192.168.111.133:6443 --token 2ioekx.gtygik5gvqdym370 \
#     --discovery-token-ca-cert-hash sha256:a38c6defced0e9901f35dd69e36419feb9598d5b7d8f8890418375ee3088e130 \
#     --control-plane --certificate-key 8f5cfa993eacc368511941f0ac32239c9c50891e5c8e3f163fc8f26cd3e0af8b
#
# Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
# As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
# "kubeadm init phase upload-certs --upload-certs" to reload certs afterward.
#
# Then you can join any number of worker nodes by running the following on each as root:
#
# kubeadm join 192.168.111.133:6443 --token 2ioekx.gtygik5gvqdym370 \
#     --discovery-token-ca-cert-hash sha256:a38c6defced0e9901f35dd69e36419feb9598d5b7d8f8890418375ee3088e130
```

**13. 添加新的master节点**  
```shell
kubeadm join 192.168.111.133:6443 --token 2ioekx.gtygik5gvqdym370 \
  --discovery-token-ca-cert-hash sha256:a38c6defced0e9901f35dd69e36419feb9598d5b7d8f8890418375ee3088e130 \
  --control-plane --certificate-key 8f5cfa993eacc368511941f0ac32239c9c50891e5c8e3f163fc8f26cd3e0af8b
```

**14. 添加新的node节点**
```shell
kubeadm join 192.168.111.133:6443 --token 2ioekx.gtygik5gvqdym370 \
    --discovery-token-ca-cert-hash sha256:a38c6defced0e9901f35dd69e36419feb9598d5b7d8f8890418375ee3088e130
```
**15. 部署Flannel网络**
```shell
# 注意： - 为了使Flannel正常工作，执行kubeadm init命令时需要增加--pod-network-cidr=10.244.0.0/16参数。-Flannel适用于amd64，arm，arm64和ppc64le上工作，但使用除amd64平台得其他平台，你必须手动下载并替换amd64。
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
**16. 查看所有节点状态**  
```shell
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    master   20h   v1.16.3
k8s-master02   Ready    master   20h   v1.16.3
k8s-master03   Ready    master   19h   v1.16.3
k8s-node01     Ready    <none>   37m   v1.16.3
k8s-node02     Ready    <none>   31m   v1.16.3
```
**17. 查看系统组件**  
```shell
[root@k8s-master01 ~]# kubectl get pod -nkube-system -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP                NODE           NOMINATED NODE   READINESS GATES
coredns-5644d7b6d9-g228d               1/1     Running   0          20h   10.100.1.2        k8s-master02   <none>           <none>
coredns-5644d7b6d9-pnv5f               1/1     Running   0          20h   10.100.1.3        k8s-master02   <none>           <none>
etcd-k8s-master01                      1/1     Running   0          20h   192.168.111.128   k8s-master01   <none>           <none>
etcd-k8s-master02                      1/1     Running   0          20h   192.168.111.129   k8s-master02   <none>           <none>
etcd-k8s-master03                      1/1     Running   0          19h   192.168.111.130   k8s-master03   <none>           <none>
kube-apiserver-k8s-master01            1/1     Running   0          20h   192.168.111.128   k8s-master01   <none>           <none>
kube-apiserver-k8s-master02            1/1     Running   0          20h   192.168.111.129   k8s-master02   <none>           <none>
kube-apiserver-k8s-master03            1/1     Running   0          19h   192.168.111.130   k8s-master03   <none>           <none>
kube-controller-manager-k8s-master01   1/1     Running   0          20h   192.168.111.128   k8s-master01   <none>           <none>
kube-controller-manager-k8s-master02   1/1     Running   0          20h   192.168.111.129   k8s-master02   <none>           <none>
kube-controller-manager-k8s-master03   1/1     Running   0          19h   192.168.111.130   k8s-master03   <none>           <none>
kube-flannel-ds-amd64-7xd8v            1/1     Running   0          28m   192.168.111.132   k8s-node02     <none>           <none>
kube-flannel-ds-amd64-8hk4m            1/1     Running   0          28m   192.168.111.130   k8s-master03   <none>           <none>
kube-flannel-ds-amd64-djddv            1/1     Running   0          28m   192.168.111.128   k8s-master01   <none>           <none>
kube-flannel-ds-amd64-hpcmw            1/1     Running   0          28m   192.168.111.129   k8s-master02   <none>           <none>
kube-flannel-ds-amd64-r6zwd            1/1     Running   0          28m   192.168.111.131   k8s-node01     <none>           <none>
kube-proxy-662qm                       1/1     Running   0          33m   192.168.111.132   k8s-node02     <none>           <none>
kube-proxy-gqxzb                       1/1     Running   0          19h   192.168.111.130   k8s-master03   <none>           <none>
kube-proxy-pf55v                       1/1     Running   0          20h   192.168.111.129   k8s-master02   <none>           <none>
kube-proxy-xtnw8                       1/1     Running   0          38m   192.168.111.131   k8s-node01     <none>           <none>
kube-proxy-zd8wn                       1/1     Running   0          20h   192.168.111.128   k8s-master01   <none>           <none>
kube-scheduler-k8s-master01            1/1     Running   0          20h   192.168.111.128   k8s-master01   <none>           <none>
kube-scheduler-k8s-master02            1/1     Running   0          20h   192.168.111.129   k8s-master02   <none>           <none>
kube-scheduler-k8s-master03            1/1     Running   0          19h   192.168.111.130   k8s-master03   <none>           <none>
```

**18. 部署仪表盘WebUI**  
```shell
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml
```

**19. 创建以token方式登录dashborad的用户**   
```shell
kubectl create serviceaccount dashboard-admin -n kube-system #创建用于登录dashborad的serviceaccount账号
ubectl create clusterrolebinding dashboard-cluster-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin   #创建一个clusterrolebingding，将名称为cluster-admin的clusterrole绑定到我们刚刚从的serviceaccount上，名称空间和sa使用:作为间隔
kubectl get secret -n kube-system #创建完成后系统会自动创建一个secret，名称以serviceaccount名称开头
kubectl describe secret dashboard-admin-token-pbsj9 -n kube-system  #使用describe查看该secret的详细信息，主要是token一段
```


**20. 解决flannel下Kubernetes pod及容器无法跨主机互通问题**    
```shell
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -F
iptables -L -n
```



参考文档:  
[http://docs.kubernetes.org.cn/459.html](http://docs.kubernetes.org.cn/459.html)  
[https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm/](https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm/)  
[https://segmentfault.com/a/1190000018741112](https://segmentfault.com/a/1190000018741112)  



#### 允许master节点部署pod
```shell
kubectl taint nodes --all node-role.kubernetes.io/master-
kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule #如果不允许调度:
污点可选参数:
      NoSchedule: 一定不能被调度
      PreferNoSchedule: 尽量不要调度
      NoExecute: 不仅不会调度, 还会驱逐Node上已有的Pod
```
