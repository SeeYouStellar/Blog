# 前置操作：

使用别名代替kubectl，输命令时可以偷懒，将```alias kc=kubectl```写入.bashrc文件并使用```source .bashrc```使其生效。接下来就可以用kc来代替kubectl了

还是不够偷懒，执行下述操作，把自动补全kubectl命令也安排上。
1. 安装bash-completion。github可下载。下载后解压解压完执行下面的代码，使其生效。$BASH_COMPLETION_PATH是bash-completion解压后的路径。要想使以后每次打开登录后都能生效，需要将下述代码写入/etc/profile或.bashrc，并使其生效。
```shell
source $BASH_COMPLETION_PATH/bash_completion
```
2. 将下述代码写入全局变量文件/etc/profile(或者写入某个用户的.bashrc文件，使其对某个用户生效)，然后用source使其生效
```shell
source <(kubectl completion bash)
```
1. 要想使补全和命令代替同时生效，需要将2的代码修改如下，并使其生效
```shell
source <(kubectl completion bash | sed s/kubectl/kc/g) 
```

## kubectl explain [type]
类似于linux man文档

## kubectl apply 和 kubectl create

相同点：都可以用来创建新的对象

不同点：create只能创建一个对象一次，重复创建会报错；而apply将配置应用于对象，可以对同一个对象进行多次配置应用。

## 强制删除资源

```shell
kubectl delete pod [pod name] --force --grace-period=0 -n [namespace]
```

## 进入pod

```shell
kc exec -it -n namespace podname -- /bin/sh
```

```shell
docker exec -it container-name /bin/sh
```

注意这里容易犯错的是，使用kc在master节点创建了pod后，pod内的容器不一定在master节点运行docker ps看到，因为docker没有集群，master节点的docker无法获得运行在其他节点的pod的容器信息。所以需要去运行该pod的节点上运行docker ps。