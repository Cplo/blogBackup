---
title: 拓展Kubernetes集群规模到2500节点
date: 2018-07-29 23:22:18
categories:
- Kubernetes
tags:
- kubernetes
- 翻译
---
原文链接：https://blog.openai.com/scaling-kubernetes-to-2500-nodes/

我们已经用Kubernetes做深度学习研究两年了。尽管我们最大规模的负载是管理裸云虚拟机，但Kubernetes提供了更快的迭代周期、合理的可拓展性使得其成为实验的理想选择。现在我们管理几个Kubernetes集群（一些在云上一些在物理机上），最大的集群已经2500个节点。这个集群运行在Azure的D15v2和NC24型号的虚拟机中。  
在这种规模下，许多系统组件遇到了问题，如etcd、k8s masters、docker image pull、网络、kubedns甚至机器的arp缓存。这里分享下我们遇到的问题和解决办法。

## etcd
在我们节点规模达到500后，kubectl命令操作会出现超时的现象，通过增加k8s master节点（kube-apiserver运行在虚拟机中）暂时解决了该问题，但治标不治本。

这使得我们强烈怀疑etcd集群，它是k8s master的中央存储。通过监控发现，尽管每台机器使用有着5000的iops性能的P30 SSD，但我们看到etcd副本的写入时延仍达到几百毫秒。

![image](https://note.youdao.com/yws/public/resource/59eab83614af4d1cefd08c4d3a9f9ddc/xmlnote/WEBRESOURCE9764bed05d7cfc52b2938526d06c2010/5878)

这个时延正在阻碍我们的集群的正常运转。

使用fio对性能进行基准测试，我们看到etcd只能使用大约10％的可用IOPS，因为写入延迟为2ms，而etcd执行顺序I/O，使其具有延迟限制。


然后，我们将每个节点的etcd目录移动到**本地临时磁盘**，该磁盘是直接连接到实例的SSD而不是网络连接的。切换到本地磁盘会使写入延迟达到200us，而etcd变得健康！

我们的集群运行良好，直到集群规模到大约1,000个节点，此时我们再次看到来自etcd的高提交延迟。这一次，我们注意到kube-apiservers从etcd读取的速度超过500MB / s。我们设置Prometheus来监控apiservers，并设置--audit-log-path和--audit-log-maxbackup标志以启用更多的apiserver日志记录。发现许多慢查询和对list api for Events的过度调用。


根本原因是Fluentd和Datadog监控过程的默认设置是从集群中的每个节点查询apiservers（例如，此问题现已修复）。我们只是简单地改变了这些流程，使其轮询降低频率，发现apiservers的负载再次变得稳定。

![image](https://note.youdao.com/yws/public/resource/59eab83614af4d1cefd08c4d3a9f9ddc/xmlnote/WEBRESOURCE289b4d64aa7de72b2475a4f351b9e4e4/5888)

etcd出口从500MB/s+下降到几乎为0（上图中的负数表示出口）

另一个有用的调整是将Kubernetes事件存储在一个单独的etcd集群中，因此事件创建中的峰值不会影响主要etcd实例的性能。为此，我们只需将--etcd-servers-overrides标志设置为以下内容：--etcd-servers-overrides=/events#https://0.example.com:2381;https://1.example.com:2381;https://2.example.com:2381

另一个是1000个节点会发生的故障是达到了etcd的硬存储限制（默认为2GB），这导致它停止接受写入。这引发了级联故障：我们所有的Kube节点都未通过运行状况检查，我们的自动定标器因此决定终止所有工作节点。我们使用--quota-backend-bytes标志增加了最大etcd大小，并且autoscaler现在有一个健全性检查，如果它将终止超过50％的集群节点，则不采取行动。

## Kube masters
我们在同一台机器上配置kube-apiserver，kube-controller-manager和kube-scheduler进程。为了获得高可用性，我们总是至少有2个主服务器，并将--apiserver-count标志设置为我们正在运行的服务器数量（否则Prometheus监控可能会在实例之间混淆）。

我们主要使用Kubernetes作为批处理调度系统，并依靠我们的autoscaler来动态扩展我们的集群 - 这使我们可以显着降低空闲节点的成本，同时在快速迭代的同时仍然提供低延迟。默认的kube-scheduler策略是在节点之间均匀分布负载，但我们希望相反，以便终止未使用的节点，并且还可以快速调度大型pod。所以我们切换到以下策略：
```
{
"kind" : "Policy",
"apiVersion" : "v1",
"predicates" : [
  {"name" : "GeneralPredicates"},
  {"name" : "MatchInterPodAffinity"},
  {"name" : "NoDiskConflict"},
  {"name" : "NoVolumeZoneConflict"},
  {"name" : "PodToleratesNodeTaints"}
  ],
"priorities" : [
  {"name" : "MostRequestedPriority", "weight" : 1},
  {"name" : "InterPodAffinityPriority", "weight" : 2}
  ]
}
```

我们广泛使用KubeDNS进行服务发现，但在推出新的调度策略后不久，它开始出现可靠性问题。我们发现故障只发生在KubeDNS的某些pod上。使用新的调度策略，一些机器最终运行了10多个KubeDNS副本，创建了热点，并且我们已经超过了允许的从每个Azure VM进行外部域查找的~200QPS。

我们通过向KubeDNS pod添加反关联性规则来解决此问题：
```
affinity:
 podAntiAffinity:
   requiredDuringSchedulingIgnoredDuringExecution:
   - weight: 100
     labelSelector:
       matchExpressions:
       - key: k8s-app
         operator: In
         values:
         - kube-dns
     topologyKey: kubernetes.io/hostname

```

## Docker image pulls
我们的Dota项目始于Kubernetes，随着它的扩展，我们注意到新鲜的Kubernetes节点通常有待放置待定很长时间。游戏图像大约为17GB，并且通常需要30分钟才能启动一个新的群集节点，因此我们理解为什么Dota容器将pending一段时间 - 但对于其他容器也是如此。挖掘，我们发现kubelet有一个--serialize-image-pulls标志，默认为true，这意味着Dota image pull阻止了所有其他image。更改为false需要将Docker切换为overlay2而不是AUFS。为了进一步加快拉动速度，我们还将Docker根目录移到了有SSD的实例，就像我们为etcd机器所做的那样。

即使在优化拉速后，我们也看到pod无法启动时出现错误信息：rpc error：code = 2 desc = net / http：request cancelled。由于缺乏进度，kubelet和Docker日志还包含指示image拉取已被取消的消息。我们跟踪根部到大image需要太长时间来拉取，或者我们有大量积压的image来拉动。为了解决这个问题，我们将kubelet的--image-pull-progress-deadline标志设置为30分钟，并将Docker守护程序的max-concurrent-downloads选项设置为10.（第二个选项不会加速提取大image，但是允许image队列并行拉动。）


我们上次的Docker pull问题是由于Google Container Registry。默认情况下，kubelet从gcr.io（由-pod-infra-container-image标志控制）中提取pause image，该image在启动任何新容器时使用。如果由于任何原因（如超出配额）而导致拉动失败，则该节点将无法启动任何容器。因为我们的节点通过NAT到达gcr.io而不是拥有自己的公共IP，所以我们很可能会达到IP配额限制。为了解决这个问题，我们只需使用docker image save -o /opt/preloaded_docker_images.tar和docker image load -i /opt/preloaded_docker_images.tar，在我们的Kubernetes工作者的机器映像中预装Docker镜像。为了提高性能，我们对常见的OpenAI内部image（如Dota image）的白名单也这样做。

## Networking
随着我们的实验变得越来越大，它们也变得越来越复杂，分布式系统严重依赖网络进行操作。当我们第一次开始运行分布式实验时，很明显我们的网络配置不好。直接在机器之间我们获得了10-15Gbit / s的吞吐量，但我们使用flannel的Kube pod最大输出为~2Gbit / s。 Machine Zone公布的基准测试显示了类似的数字，这意味着问题可能不仅仅是错误的配置，而是我们环境固有的东西。 （相比之下，flannel不会在我们的物理机器上增加这种开销。）

要解决此问题，用户可以添加两个不同的设置来为其pod禁用Flannel：hostNetwork：true和dnsPolicy：ClusterFirstWithHostNet。 （尽管在执行此操作之前，请阅读Kubernetes文档中的警告。）

## ARP Cache
尽管我们进行了DNS调整，但我们仍然看到了DNS解析的间歇性问题。有一天，一位工程师报告说，他们的Redis服务器的nc -v需要30秒才能打印出已建立的连接。我们将问题跟踪到内核的ARP堆栈。 Redis pod主机的初步调查显示网络存在严重错误：任何端口上的通信都挂了几秒钟，并且没有DNS名称可以通过本地dnsmasq守护进程解析，而dig只是打印一条神秘的失败消息：socket.c ：1915：internal_send：127.0.0.1＃53：Invalid argument。 dmesg日志提供了更多信息：邻居表溢出！这意味着ARP缓存空间不足。 ARP用于将诸如IPv4地址的网络地址映射到物理地址，例如MAC地址。幸运的是，通过在/etc/sysctl.conf中设置一些选项很容易解决这个问题：
```
net.ipv4.neigh.default.gc_thresh1 = 80000
net.ipv4.neigh.default.gc_thresh2 = 90000
net.ipv4.neigh.default.gc_thresh3 = 100000
```
在HPC群集中调整此设置很常见，并且在Kubernetes群集中尤其相关，因为每个pod都有自己的IP地址，这会消耗ARP缓存中的空间。

我们的Kubernetes集群现已无事故发生约3个月，我们计划在2018年扩展到更大的集群。我们最近升级到1.8.4版本，很高兴看到它现在正式支持5000节点。如果您对构建大型计算集群感兴趣，我们正在招聘！
