# 前置操作：

使用别名代替kubectl，输命令时可以偷懒，将```alias kc=kubectl```写入.bashrc文件并使用```source .bashrc```使其生效。接下来就可以用kc来代替kubectl了

还是不够偷懒，执行下述操作，把自动补全kubectl命令也安排上。
1. 安装bash-completion。github可下载。下载后解压解压完执行下面的代码，使其生效。
```shell
source $PWD/bash_completion
```
2. 将下述代码写入全局变量文件/etc/profile(或者写入某个用户的.bashrc文件，使其对某个用户生效)，然后用source使其生效
```shell
source <(kubectl completion bash | sed s/kubectl/kc/g)
```
3. 要想使补全和命令代替同时生效，需要将下述代码写入/etc/profile并使其生效
```shell
source <(kubectl completion bash | sed s/kubectl/kc/g) 
```

# kubectl explain [type]
类似于linux man文档



