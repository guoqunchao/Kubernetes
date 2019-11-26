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

<font size=10>你在哪里 </font>


<font size=3>这里有个问题，为什么不把kubelet组件也容器化呢，是因为，kubelet在配置容器网络、管理容器数据卷时，都需要直接操作宿主机，而如果现在 kubelet 本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦。比如，容器内要做NFS的挂载，需要kubelet先在宿主机执行mount挂载NFS。如果kubelet运行在容器中问题来了，如果kubectl运行在容器中，要操作宿主机的Mount Namespace是非常复杂的。所以，kubeadm选择把kubelet运行直接运行在宿主机中，使用容器部署其他Kubernetes组件。所以，Kubeadm部署要安装的组件有Kubeadm、kubelet、kubectl三个。
上面说的是kubeadm部署方式的一般步骤，kubeadm部署是可以自由定制的，包括要容器化哪些组件，所用的镜像，是否用外部etcd，是否使用用户证书认证等以及集群的配置等等，都是可以灵活定制的，这也是kubeadm能够快速部署一个高可用的集群的基础。详细的说明可以参考官方Reference。但是，kubeadm最重要的作用还是解决集群部署问题，而不是集群配置管理的问题，官方也建议把Kubeadm作为一个基础工具，在其上层再去量身定制适合自己的集群的管理工具；</font>






















## 创建一个表格
一个简单的表格是这么创建的：
项目     | Value
-------- | -----
电脑  | $1600
手机  | $12
导管  | $1

