# 组件

![Alt text](image/image5.png)

etcd 保存了整个集群的状态；
kube-apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制；
kube-controller-manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
kube-scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上；
kubelet 负责维持容器的生命周期，同时也负责 Volume（CVI）和网络（CNI）的管理；
Container runtime 负责镜像管理以及 Pod 和容器的真正运行（CRI），默认的容器运行时为 Docker；
kube-proxy 负责为 Service 提供 cluster 内部的服务发现和负载均衡；

## 组件高可用

控制平面的组件可以不在同一台服务器上，并且可以有多个组件实例，但是只有一个实例处于运行状态