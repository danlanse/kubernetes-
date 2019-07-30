kubectl基本使用
===
[TOC]

### 1. kubectl 创建对象
``` json
$ kubectl create -f ./my-manifest.yaml     #创建资源
$ kubectl create -f ./my1.yaml -f ./my2.yaml    #使用多个文件创建资源
$ kubectl create -f ./dir    #使用目录下的所有清单文件（yaml）来创建资源
$ kubectl create -f https://git.io/vPieo    #使用url创建资源
$ kubectl run nginx --image=nginx    #启动一个nginx实例
$ kubectl explain pods    #获取pod和svc的文档
```

### 2. kubectl 显示和查找资源
``` json
以下命令查找资源时可能查不到的原因是需要指定namespace，通过 -n <namespace>指定即可，或者all

$ kubectl get pods --all-namespaces    #列出所有namespace中的pod，也可以是services、deployment等
$ kubectl get pods -o wide    #列出pod并显示详细信息
$ kubectl get deployment my-dep    #列出指定daployment
$ kubectl get pods --include-uninitialized    #列出该namespace中的所有pod，包括未初始化的

```

### 2. kubectl 显示和查找资源
``` json
使用详细输出来描述命令
$ kubectl describe nodes <my-node IP or name>    #查看node节点信息
$ kubectl describe pods <my-pod>    #查看pod详细信息
$ kubectl get services --sort-by=.metadata.name --all-namespaces    #l列出所有service并按名称排序
```
``` json
根据重启次数排序列出pod

$ kubectl get pods --sort-by='.status.containerStatuses[0].restartCount' --all-namespaces
```
``` json
获取所有具有app=cassandra的pod中的version标签

$ kubectl get pods --selector=app=cassandra rc -o jsonpath='{.items[*].metadata.labels.version}'
```

``` json
获取所有节点的ExternalIP

$ kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternlIP")].address}'
```

### 3. kubectl编辑资源
``` json
$ kubectl -n codeus edit svc/c    #编辑codeus命名空间下名称为c的service
```

### 4. kubectl Scale 资源
``` json
扩展pod下容器数量
$ kubectl scale --replicas=3 rs/foo    #扩展名称为foo的资源到3个，是否使用rs取决于yaml中的编写
```
``` json
例如yaml中kind: Deployment ，则应通过下面方法扩展
$ kubectl scale --replicas=3 deployment/foo
```

``` json
或者直接通过创建资源的yaml文件扩展
$ kubectl scale --replicas=3 -f foo.yaml
```

``` json
根据判断条件扩展
例如条件是：如果mysql的数量是2，则扩展到3
$ kubectl scale --current-replicas=2 --replicas=3 deployment/mysql
```

``` json
同时扩展多个资源
$ kubectl scale --replicas=5 rc/foo rc/bar rc/baz
```

### 5. kubectl 删除资源
``` json
$ kubectl delete deployment <name>     #删除指定deployment，此方法还可以删除service等
$ kubectl delete -f xxx.yaml    #通过创建此pod的yaml文件删除pod
```

### 6. kubectl 与运行中的pod交互
``` json
$ kubectl -n <namespaces> logs my-podname    #查看pod日志， -f 持续查看
$ kubectl port-forward my-podname 5000:6000    #转发pod中的6000端口到本地的5000端口
$ kubectl exec my-podname -- ls /    #在已存在的容器中执行命令
```
