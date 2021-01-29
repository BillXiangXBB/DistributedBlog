# etcd

## etcd简介
etcd是一个分布式的可靠的键值对存储系统，其通常被用来存储分布式系统的关键元数据等。目前，etcd广泛应用于各种云原生分布式项目中，如Kubernetes、locksmith、vulcand等。etcd作为了key-value存储系统，具有如下特点：
* 简便：定义明确，面向用户的API（使用gRPC）
* 安全：通过可选的客户端证书认证自动支持TLS
* 快捷：基准测试写入10000次/s
* 可靠：基于Raft算法实现

## 文章简介
本系列文章结合etcd源码从存储、网络、raft算法、WAL、租约等方面详细分析etcd的内部原理。
