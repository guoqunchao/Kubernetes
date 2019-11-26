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

**5. 部署环境**
由于机器资源不足，下面的部署测试，只会以混布的方式部署一个1*haproxy，3*master，2*node，共5台机器的集群，实际上由于etcd选举要过半数，至少要3台master节点才能构成高可用，在生产环境，还是要根据实际情况，尽量选择风险低的拓扑结构。
