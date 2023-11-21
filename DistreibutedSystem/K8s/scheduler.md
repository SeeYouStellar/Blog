# 使用污点和容忍度对pod进行调度

Taints：污点，节点的字段

Toleration：容忍度，pod的字段

定义格式：key=value,operator:effect

如果没有value可以不写"="和value

**本质是一个匹配问题，pod无法调度到有其无法容忍的污点的节点上。这里的匹配指的是key/effect都要匹配。并且当operator为exsits时，只需要key相同就行（此时也不能指定value）；当operator为equal时，需要value也匹配。**

特殊情况：

- key为空，operator为exist，则表示啥都容忍
- effect为空，则表示任意operator都容忍

effect类型：
- NoExecute:这会影响已在节点上运行的 Pod，具体影响如下：
    - 如果 Pod 不能容忍这类污点，会马上被驱逐。
    - 如果 Pod 能够容忍这类污点，但是在容忍度定义中没有指定 tolerationSeconds， 则 Pod 还会一直在这个节点上运行。
    - 如果 Pod 能够容忍这类污点，而且指定了 tolerationSeconds， 则 Pod 还能在这个节点上继续运行这个指定的时间长度。 这段时间过去后，节点生命周期控制器从节点驱除这些 Pod。
- NoSchedule：除非具有匹配的容忍度规约，否则新的 Pod 不会被调度到带有污点的节点上。 当前正在节点上运行的 Pod 不会被驱逐。
- PreferNoSchedule：是"偏好"或"软性"的 NoSchedule。 控制平面将尝试避免将不能容忍污点的 Pod 调度到的节点上，但不能保证完全避免。

当节点与pod匹配完后还有taint：
- 如果未被忽略的污点中存在至少一个 effect 值为 NoSchedule 的污点， 则 Kubernetes 不会将 Pod 调度到该节点。
- 如果未被忽略的污点中不存在 effect 值为 NoSchedule 的污点， 但是存在至少一个 effect 值为 PreferNoSchedule 的污点， 则 Kubernetes 会 尝试 不将 Pod 调度到该节点。
- 如果未被忽略的污点中存在至少一个 effect 值为 NoExecute 的污点， 则 Kubernetes 不会将 Pod 调度到该节点（如果 Pod 还未在节点上运行）， 并且会将 Pod 从该节点驱逐（如果 Pod 已经在节点上运行）。

# 节点亲缘性

表达性更好的方式

requiredDuringSchedulingIgnoredDuringExecution：调度时必须满足某个规则；在执行中的若节点规则改变，pod继续执行

preferredDuringSchedulingIgnoredDuringExecution：调度时尽量满足某个规则；在执行中的若节点规则改变，pod继续执行

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - antarctica-east1
            - antarctica-west1
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1 # 权重
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
      - weight: 5
        preference:
          matchExpressions:
          - key: 
            operator: 
            values:
            - 
```

operator：
- In
- Exists
- Gt
- Lt
- NotIn 节点反亲和性
- DoesExist 节点反亲和性

nodeSelectorTerms字段可以有多个matchExpressions字段，只要满足一个就可以被调度

nodeSelectorTerms.matchExpressions字段若有多个表达式，则必须都满足才能被调度

