# ReplicaSet

保证集群的pod副本数量

以标签匹配为基础的pod副本管理对象

根据标签名、标签值、多个相同标签名的不同标签值（作为一个组）来匹配
## rs配置文件

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
   name: kubia
spec:
   replicas: 5
   selector:
     matchExpressions:
       - key: app
         operator: In
         values:
           - kubia
   template:
     metadata:
       labels:
           app: kubia
     spec:
       containers:
       - name: kubia
         image: luksa/kubia
         ports:
         - containerPort: 8080
           protocol: TCP
```
## rs更新

改动replicas：副本的数量会随之改动，效果和终端输入下述语句一样
```shell
kubectl scale rs kubia --replicas=3 
```
改动sepc.template：之前属于rs管辖的pod会被删除，创建新的pod

删除rs，默认会将所有rs控制的pod也删除，添加``` --cascade= false```可以保留pod

# DaemonSet

DaemonSet 确保所有符合条件的节点都运行该 Pod 的一个副本 当有节点加入集群时， 也会为他们新增一个 Pod 。 当有节点从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

DaemonSet 的一些典型用法：

- 在每个节点上运行集群守护进程
- 在每个节点上运行日志收集守护进程
- 在每个节点上运行监控守护进程

具有和rs一样的标签匹配功能

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
   name: kubia
spec:
   selector:
     matchExpressions:
       - key: app
         operator: In
         values:
           - kubia
   template:
     metadata:
       labels:
           app: kubia
     spec:
       containers:
       - name: kubia
         image: luksa/kubia
         ports:
         - containerPort: 8080
           protocol: TCP
```

spec.selector为Pod选择算符，必须与.spec.template.metadata.labels属性相匹配

spec.template.spec.nodeSelector筛选可以运行template里的pod的节点，然后这些节点上只能运行一个该pod

## ds更新

某个节点被添加到集群，那么该节点立即添加一个符合要求的pod

某个节点的标签改动使其不符合ds的spec.template.spec.nodeSelector，那么该节点上的这个pod会被删除。

ds的spec.template.spec.nodeSelector改动，则会重新匹配新的标签上的节点并创建pod，同时删除不匹配节点上的pod

删除ds，默认会将所有ds控制的pod也删除，添加``` --cascade=orphan```可以保留pod