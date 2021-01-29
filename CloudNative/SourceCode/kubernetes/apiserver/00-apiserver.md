# kube-apiserver

## kubeapiserver简介
Kubernetes采用了轮辐式（Hub And Spoke）API模式。所以kube-apiserver是Kubernetes控制面的核心组件之一，主要提供了以下的两个功能：
* 提供集群管理的REST API接口，包括认证鉴权、数据校验以及集群状态变更等接口；
* 提供其他模块之间的数据交互和通信枢纽，其他模块通过API Server查询和修改数据，只有API Server才直接操作etcd。

*Tips：轮辐模型（Hub And Spoke）*  
Hub意为轮毂，Spoke表示轮辐。轮辐模型是一种简化网络路由的中心化体系，广泛应用与航空、货运、快递以及网络技术领域。其核心思想就是，无论去Spoke上哪个点，都必须经过中心Hub，从而简化路由，便于管理。但此时中心Hub就会成为整个系统的瓶颈。
