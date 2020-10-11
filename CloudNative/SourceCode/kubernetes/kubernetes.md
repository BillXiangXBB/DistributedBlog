# Kubernetes

Kubernetes是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。Kubernetes拥有一个庞大且快速增长的生态系统。Kubernetes的服务、支持和工具广泛可用。

Kubernetes的架构如下图所示：  
![Kubernetes架构](../../../images/kubernetes/00-KubernetesStructure.jpg)

通过上图可看出，Kubernetes主要包含以下几个重要的核心组件：
* apiserver：主节点上负责提供Kubernetes API服务的组件，提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
* controller manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
* scheduler：负责资源的调度，按照预定的调度策略将pod调度到相应的机器上；
* etcd：保存整个集群的状态；
* kubelet：负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；
* kube-proxy：负责为Service提供cluster内部的服务发现和负载均衡；
* Container runtime：负责镜像管理以及pod和容器的真正运行（CRI）。

除了核心组件外，还有一些推荐的Add-ons：
* kube-dns：负责为整个集群提供DNS服务
* Ingress Controller：为服务提供外网入口
* Heapster：提供资源监控
* Dashboard：提供GUI
* Federation：提供跨可用区的集群
* Fluentd-elasticsearch：提供集群日志采集、存储和查询

本文也将从这些组件入手，详细分析Kubernetes内部实现原理，并对核心源码进行解读。
