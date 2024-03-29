# Pod基础及进阶


## 为什么需要Pod

在日常运维中，我们经常遇到需要几个服务配合完成一件事的情况，比如PHP项目，我们需要nginx解析静态文件，需要PHP解析动态文件，所以通常需要将Nginx和PHP运行在同一台机器上，这个需求怎么在Kubernetes里实现呢？

我们知道容器的本质是进程，docker官方也建议一个容器最好只干一件事情，我们将Nginx和PHP跑在一个容器里不是一种好的设计模式，那我们该如何实现这种服务间亲密关系拓扑呢？Kubernetes的Pod正是帮我们实现这种服务间亲密关系拓扑的。那他是怎么实现的呢？

## Pod设计原理

首先Pod只是一个逻辑概念，也就是说，Kubernetes真正处理的，还是宿主机操作系统上容器的Namespace和Cgroups，而并不存在一个所谓的Pod的边界或者隔离环境。Pod其实就是一组共享了某些资源的容器。具体的说：Pod里的所有容器，共享的是同一个Network Namespace，并且可以声明共享同一个Volume。

举例说明，一个有A、B两个容器的Pod，就等同于一个容器（容器 A）共享另外一个容器（容器 B）的网络和Volume。

比如：

    $ docker run --net=B --volumes-from=B --name=A image-A

但是，你有没有考虑过，如果真这样做的话，容器B就必须比容器A先启动，这样一个Pod里的多个容器就不是对等关系，而是拓扑关系了。

所以，在Kubernetes项目里，Pod的实现需要使用一个中间容器，这个容器叫作Infra容器。在这个Pod中，Infra容器永远都是第一个被创建的容器，而其他用户定义的应用容器，与Infra容器关联在一起，共享Infra容的网络空间和Volume，如下图所示：

![Pod](https://github.com/findsec-cn/k201/raw/master/imgs/2/pod_infra.jpg)

这个Pod里有三个用户容器A、B和C，还有一个Infra容器。很容易理解，在Kubernetes项目里，Infra容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有100~200KB左右。

当Infra容器创建完网络空间和Volume，应用容器就可以加入到Infra容器的Network Namespace当中了。这也就意味着，对于Pod里的容器A、容器B和容器C来说：

- 它们可以直接使用localhost进行通信；
- 一个Pod只分配IP地址，也就是Infra容器对应的IP地址；
- Pod的生命周期只跟Infra容器一致，而与应用容器无关；
- 相同Pod的容器可以共享Volume。

我们可以将Pod类比为虚拟机，Pod里面运行的容器就是虚拟机里运行的进程。有Pod的这个概念，Kubernetes就能够将Pod当做一个整体进行调度了，在计算调度资源时，会考虑到Pod中的所有容器，只有满足Pod中所有容器资源时才会调度到此节点上。

## 什么样的容器应该放到同一个Pod中

Pod是为了更好的解决服务间亲密关系而设计的，只有服务间具备这种亲密关系，才应该被放到一个Pod里，比如Nginx和PHP需要共享代码，Nginx需要和PHP进行本地通信（通过localhost访问PHP），所有把他们放到相同的Pod里是合理的；但是，向PHP也会访问MySQL服务，那它们是否也需要放到同一个Pod呢？通常我们不会将PHP和MySQL放到同一个Pod，原因是PHP和MySQL使用的资源通常是不同的，MySQL需要更快的硬盘、更多的内存和CPU，PHP和MySQL通常会放到不同的宿主机上，而PHP通常通过网络连接MySQL服务，所有并不会把MySQL和PHP放到同一个Pod中。

在明白了Pod的设计原理之后，我们来看下Pod这个API对象。

## Pod对象

Pod是Kubernetes调度的最小单位，Kubernetes中的其他编排对象如：Deployment、StatefulSet、DaemonSet、Job等最终调度的都是Pod对象，Pod在Kubernetes世界里是一等公民，所以它被设计到核心API中。

首先我们从整体上来看下Pod这个API对象：

![Pod Object](https://github.com/findsec-cn/k201/raw/master/imgs/2/pod_api_object.jpg)

- **apiVersion** api版本，我们可以通过`kubectl api-versions`查看个api组的版本，不同组API版本是不一样的，Pod属于核心API，版本为V1；
- **kind** 编排对象，此处是pod，以后还会学习Deployment、StatefulSet、DaemonSet、Service、Ingress等编排对象；
- **metadata** 元数据，用来定义编排对象的名字（name），所在命名空间（namespace）、标签（labels）、注解（annotations）等，在Kubernetes世界里，一个API对象通过标签（label selector）来选择另一个API对象，所以为对象定义标签是很重要的；metadata中的注解（annotations）是给Kubernetes看的，比如告诉Kubernetes开启某些功能；
- **spec** 描述，用来描述这个API对象的期望状态，比如如何调度、如何配置网络、如何配置存储、如何配置安全上下文、如何启动容器等等；
- **status** 状态，这个字段是只读的，用来展示这个API对象的实际状态，status有包括conditions详细状态和containerStatuses，后面会详细介绍。

## Pod示例

### 启动Nginx服务

示例（1.pod-nginx.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

- name：启动容器的名字
- image：启动容器使用的镜像和标签
- imagePullPolicy：拉取镜像的规则，Always：始终拉取；Never：从不拉取，需要镜像已存在宿主机上；IfNotPresent：如果不存在拉取

部署配置清单：

    $ kubectl apply -f 1.pod-nginx.yaml

查看部署情况：

    $ kubectl get pods -w

可以如下查看Pod详情：

    $ kubectl describe pod example

通过代理访问Pod容器：

    $ kubectl port-forward example 8080:80

打开浏览器访问：127.0.0.1:8080

### Init容器

在nginx容器启动以前，有时我们需要先做些准备工作，比如下载nginx使用的代码文件。Init容器与普通的容器非常像，但是Init容器总是运行到成功完成为止。另外，每个Init容器都必须在下一个Init容器启动之前成功完成。

我们说Init容器主要是来做初始化工作的，那么他有哪些应用场景呢？

- 等待其他服务进入就绪状态：这个可以用来解决服务之间的依赖问题，比如我们有一个Web服务，该服务又依赖于另外一个数据库服务，但是在我们启动这个Web服务的时候我们并不能保证依赖的这个数据库服务就已经启动起来了，所以可能会出现一段时间内Web服务连接数据库异常。要解决这个问题的话我们就可以在Web服务的Pod中使用一个InitContainer，在这个初始化容器中去检查数据库是否已经准备好了，如果数据库服务准备好了，初始化容器就结束退出，然后我们的主容器Web服务开始启动，这个时候去连接数据库就不会有问题了。
- 做初始化配置：比如集群里检测所有已经存在的成员节点，为主容器准备好集群的配置信息，这样主容器起来后就能用这个配置信息加入集群。

示例（2.pod-init-containers.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent
        volumeMounts:
        - name: workdir
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: install
        image: busybox
        command:
        - wget
        - "-O"
        - "/work-dir/index.html"
        - http://kubernetes.io
        volumeMounts:
        - name: workdir
          mountPath: "/work-dir"
      volumes:
      - name: workdir
        emptyDir: {}

我们在这里定义了一个volumes，它是一个共享卷，我们可以将它挂载到initContainers，也可以挂载到nginx容器。我们这里使用的是emptyDir{}，Kubernetes会帮我们在宿主机上创建一个临时目录，生命周期等同于Pod的生命周期，当这个Pod被销毁的时候会同时删除这个临时目录。
初始化容器帮我们下载一个index.html文件，当初始化容器执行完后，index.html文件会被存储到临时目录里，当Nginx容器启动后会将这个目录挂载到/usr/share/nginx/html目录下，这样Nginx就能够访问到这个index.html文件了。

部署配置清单：

    $ kubectl apply -f 2.pod-init-containers.yaml --force

查看部署情况：

    $ kubectl get pods -w

通过代理访问Pod容器：

    $ kubectl port-forward example 8080:80

打开浏览器访问：127.0.0.1:8080

### Lifecycle钩子

Container Lifecycle Hooks作用是在容器状态发生变化时触发一系列“钩子”。我们来看一个例子：

示例（3.pod-lifecycle.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
          preStop:
            exec:
              command: ["/usr/sbin/nginx","-s","quit"]

先说postStart指的是在容器启动后立刻执行一个指定的操作。需要明确的是postStart定义的操作虽然是在Docker容器ENTRYPOINT执行之后，但它并不严格保证顺序。也就是说在postStart启动时，ENTRYPOINT有可能还没有结束。
当然，如果postStart执行超时或者错误，Kubernetes会在该Pod的Events中报出该容器启动失败的错误信息，同时导致Pod也处于失败的状态。
而类似的，preStop发生的时机则是容器被杀死之前。preStop操作的执行是同步的。所以，它会阻塞当前的容器杀死流程，直到这个钩子定义操作完成之后，才允许容器被杀死，这跟postStart不一样。
所以，在这个例子中，我们在容器成功启动之后，在 /usr/share/message里写入了一句“欢迎信息”。而在这个容器被删除之前，我们则先调用了 nginx的退出指令，从而实现了容器的“优雅退出”。


部署配置清单：

    $ kubectl apply -f 3.pod-lifecycle.yaml --force

查看部署情况：

    $ kubectl get pods -w

验证postStart命令已执行：

    $ kubectl exec -ti example -c nginx bash
    root@example:/# cat /usr/share/message
    Hello from the postStart handler

### 资源限制

Kubernetes作为一个容器调度平台，它需要统计整体平台的资源使用情况，合理地将资源分配给容器使用，并且要保证容器生命周期内有足够的资源来运行。 同时，如果资源发放是独占的，即资源已发放给了个容器，同样的资源不会发放给另外一个容器，对于空闲的容器来说占用着没有使用的资源比如CPU是非常浪费的，Kubernetes需要考虑如何在优先度和公平性的前提下提高资源的利用率。为了实现资源被有效调度和分配同时提高资源的利用率，Kubernetes采用Request和Limit两种限制类型来对资源进行分配。

- **Request**: 容器使用的最小资源需求，作为容器调度时资源分配的判断依赖。只有当节点上可分配资源量>=容器资源请求数时才允许将容器调度到该节点。但Request参数不限制容器的最大可使用资源。
- **Limit**: 容器可以资源的最大值，容器使用的资源是不会超过limits的限制的，设置为0表示使用资源无上限。

Request能够保证Pod有足够的资源来运行，而Limit则是防止某个Pod无限制地使用资源，导致其他Pod崩溃。建议在设置资源限制时，CPU: 可以根据业务的特点动态的调整Request和Limit的比例关系，因为CPU是竞争资源，可以适当超配。Memory: Limit最好和Request设置相同的值，因为内存是不能超配的，要保证分配到此节点的Pod有充足的资源。

示例（4.pod-resources.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        resources:
          limits:
            cpu: 500m
            memory: 0.5Gi
          requests:
            cpu: 0
            memory: 0.5Gi

部署配置清单：

    $ kubectl apply -f 4.pod-resources.yaml --force

查看部署情况：

    $ kubectl get pods -w

可以如下查看Pod详情：

    $ kubectl describe pod example

### 健康检查

接下来，我们再来看Pod的另一个重要的配置：容器健康检查和恢复机制。在Kubernetes中，你可以为Pod里的容器定义一个健康检查“探针”（Probe）。这样，kubelet就会根据这个Probe的返回值决定这个容器的状态，而不是直接以容器是否运行作为依据。在生成环境中，健康检查是保证服务正常运行的重要手段。

示例（5.pod-liveness.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      labels:
        test: liveness
      name: liveness-exec
    spec:
      containers:
      - name: liveness
        image: busybox
        args:
        - /bin/sh
        - -c
        - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
        livenessProbe:
          exec:
            command:
            - cat
            - /tmp/healthy
          initialDelaySeconds: 5
          periodSeconds: 5

当Pod启动之后做的第一件事情就是在/tmp目录下创建了一个healthy文件，容器启动5s后（initialDelaySeconds: 5）开始执行检查操作，每5s执行一次，如果/tmp/healthy存在，则cat命令正常退出，说明容器健康，30s后/tmp/healthy被删除，cat命令异常退出，说明容器不健康，这时容器会被重启。

部署配置清单：

    $ kubectl apply -f 5.pod-liveness.yaml

查看部署情况：

    $ kubectl get pods -w

等待一段时间，发现容器一直会被重启

    $ kubectl get pods -w
    NAME            READY   STATUS    RESTARTS   AGE
    example         1/1     Running   0          4m23s
    liveness-exec   1/1     Running   0          23s
    liveness-exec   1/1   Running   1     93s
    liveness-exec   1/1   Running   2     2m50s

当一个Pod不断重启的时候，我们可以通过查看Pod的详情信息里面的事件日志和状态来排查问题。

    $ kubectl describe pod liveness-exec
    Events:
      Type     Reason     Age                   From                             Message
      ----     ------     ----                  ----                             -------
      Normal   Scheduled  5m15s                 default-scheduler                Successfully assigned default/liveness-exec to k8s-worker-0
      Normal   Pulled     2m26s (x3 over 5m6s)  kubelet, k8s-worker-0  Successfully pulled image "busybox"
      Normal   Created    2m26s (x3 over 5m6s)  kubelet, k8s-worker-0  Created container liveness
      Normal   Started    2m26s (x3 over 5m5s)  kubelet, k8s-worker-0  Started container liveness
      Warning  Unhealthy  102s (x9 over 4m32s)  kubelet, k8s-worker-0  Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
      Normal   Killing    102s (x3 over 4m22s)  kubelet, k8s-worker-0  Container liveness failed liveness probe, will be restarted
      Normal   Pulling    72s (x4 over 5m13s)   kubelet, k8s-worker-0  Pulling image "busybox"

从上面的Events日志里可以看到liveness探针失败了（Liveness probe failed），容器要被重启了。这个探针是和重启策略（restartPolicy）配合使用的。这个功能就是Kubernetes里的Pod恢复机制，也叫restartPolicy。pod.spec.restartPolicy的默认值是Always，即：任何时候这个容器发生了异常，它一定会被重新创建。但一定要强调的是，Pod的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去。事实上，一旦一个Pod与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个Pod也不会主动迁移到其他节点上去。当节点出现问题时如果你想让Pod出现在其他的可用节点上，就必须使用Deployment这样的“控制器”来管理Pod，哪怕你只需要一个Pod副本。后面我们会详细介绍Deployment对象。

我们可以通过设置restartPolicy，改变Pod的恢复策略。除了Always，它还有OnFailure和Never两种情况：

- **Always**：在任何情况下，只要容器不是运行状态，就自动重启容器（干掉旧的Pod，新建一个新Pod）；
- **OnFailure**: 只在容器异常退出时才自动重启容器；
- **Never**: 从来不重启容器。

在实际使用时，我们需要根据应用运行的特性，合理设置这三种恢复策略。比如，一个Pod，它只计算1+1=2，计算完成输出结果后退出，变成Succeeded状态。这时，你如果再用restartPolicy=Always强制重启这个Pod的容器，就没有任何意义了。这时候你可以把他设置成OnFailure，比如在计算1+1=2的过程中出错，这是重启它，如果正确的得出了结果则不重启。也可以设置成Never，无论失败与否都不重启。

除了在容器中执行命令外，livenessProbe也可以定义为发起HTTP或者TCP请，定义格式如下：

    ...
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: X-Custom-Header
          value: Awesome
        initialDelaySeconds: 3
        periodSeconds: 3
    ...
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20

你的应用可以暴露一个健康检查的URL（比如 /healthz），或者直接让健康检查去检测应用的监听端口。这两种配置方法，在Web服务类的应用中非常常用。

在Kubernetes的Pod中，还有一个叫readinessProbe的字段。虽然它的用法与livenessProbe类似，但作用却是不一样的。readinessProbe检查结果的成功与否，决定了这个Pod是不是能被加入Service对外提供服务，而并不影响Pod的生命周期。

示例（6.pod-readiness.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        livenessProbe:
          tcpSocket:
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 10
          successThreshold: 1
          failureThreshold: 2
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 2

### Pod的生命周期

在熟悉了Pod以及它的Container部分的主要字段之后，我再来看下Pod对象在Kubernetes中的生命周期。

Pod生命周期的变化，主要体现在Pod API对象的Status部分，这是它除了Metadata和Spec之外的第三个重要字段。其中，pod.status.phase，就是 Pod的当前状态，它有如下几种可能的情况：

- **Pending** 这个状态意味着配置已经提交给了Kubernetes，但是这个Pod里有些容器因为某种原因而不能被顺利创建。比如集群没有资源了，Pod未调度成功。
- **Running** 这个状态下Pod已经调度成功。它包含的容器都已经创建成功，并且至少有一个容器处于运行状态。
- **Succeeded** 这个状态意味着Pod里的所有容器都正常运行完毕，并且已经正常退出了。这种情况在运行一次性任务时最为常见。
- **Failed** 这个状态下Pod里至少有一个容器以不正常的状态（非0返回码）退出。这个状态的出现，意味着你需要排除出错原因，比如查看Pod的 Events和和容器的日志。
- **Unknown** 这是一个异常状态，意味着Pod的状态不能持续地被kubelet汇报给kube-apiserver，这很有可能是主从节点（Master 和 Kubelet）间的通信出现了问题。

Pod的Status状态，还可以再细分出一组Conditions。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及Unschedulable。它们主要用于描述造成当前Status的具体原因是什么。比如，Pod当前的Status是Pending，对应的Condition是Unschedulable，这就意味着它的调度出现了问题。而其中，Ready这个细分状态非常值得我们关注：它意味着Pod不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。

### 卷（Volumes）

有时候我们需要挂载宿主机的某个目录，比如一个共享的日志目录，应用容器会将日志写到这个宿主机目录下，这样便于在宿主机下统一收集日志。我们可以通过volumes实现这个需求。

示例（7.pod-volume.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      containers:
      - name: nginx
        image: nginx:1.15.12
        volumeMounts:
        - name: logs
          mountPath: /var/log/nginx
      volumes:
      - name: logs
        hostPath:
          path: /data1/logs

本示例会将宿主机的/data1/logs挂载到nginx容器的/data/logs目录下，然后nginx在容器/data/logs目录下写日志，日志实际会希尔宿主机的/data1/logs目录下，然后我们就可在宿主机上启动一个日志收集程序来收集日志了。如果你不关心宿主机挂载的目录，只是在容器间共享存储卷来交换数据，也可以使用emptyDir: {}，这个在代码共享那个示例里已经讲过了。

### Projected Volumes

除了hostPath、emptyDir还有一种Volume叫作Projected Volume（投射数据卷）。在Kubernetes中，有几种特殊的Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器间的数据共享。这些特殊Volume的作用是为容器提供预先定义好的数据。所以，从容器的角度来看，这些 Volume里的信息就是仿佛是被Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume的含义。

到目前为止，Kubernetes支持的Projected Volume一共有四种：

- **Secret**
- **ConfigMap**
- **Downward API**
- **ServiceAccountToken**

Secret最典型的使用场景，莫过于存放用户名密码等敏感信息，比如下面这个例子：

示例（8.pod-projected-secret.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: projected-volume
    spec:
      containers:
      - name: secret-volume
        image: busybox
        args:
        - sleep
        - "86400"
        volumeMounts:
        - name: cred
          mountPath: "/projected-volume"
          readOnly: true
      volumes:
      - name: cred
        projected:
          sources:
          - secret:
              name: user
          - secret:
              name: pass

在这个Pod中，它声明挂载的Volume，并不是常见的emptyDir或者hostPath类型，而是projected类型。而这个Volume的数据来源Secret对象，分别对应的是用户名和密码。

这里用到的数据库的用户名、密码，正是以Secret对象的方式交给Kubernetes保存的。完成这个操作的指令，如下所示：

    $ cat ./user.txt
    admin
    $ cat ./pass.txt
    password

    $ kubectl create secret generic user --from-file=./user.txt
    $ kubectl create secret generic pass --from-file=./pass.txt

其中user.txt和pass.txt文件里，存放的就是用户名和密码；而user和pass，则是我为Secret对象指定的名字。我们可以通过kubectl get命令来查看Secret对象：

    $ kubectl get secrets
    NAME           TYPE                                DATA      AGE
    user          Opaque                                1         31s
    pass          Opaque                                1         31s

当然，除了使用`kubectl create secret`命令外，我也可以直接通过编写YAML文件的方式来创建这个Secret对象，比如：

apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: cGFzc3dvcmQK

> 注意: Secret对象要求这些数据必须是经过Base64转码的，以免出现明文密码的安全隐患。

转码操作也很简单，比如：

    $ echo -n 'admin' | base64
    YWRtaW4=
    $ echo -n 'password' | base64
    cGFzc3dvcmQK

这里需要注意的是，像这样创建的Secret对象，它里面的内容仅仅是经过了转码，而并没有被加密。在真正的生产环境中，你需要在Kubernetes中开启 Secret的加密插件，增强数据的安全性。

与Secret类似的是ConfigMap，它与Secret的区别在于，ConfigMap保存的是不需要加密的的配置信息。而ConfigMap的用法几乎与Secret完全相同：你可以使用`kubectl create configmap`从文件或者目录创建ConfigMap，也可以直接编写ConfigMap对象的YAML文件。

接下介绍Downward API，它的作用是：让Pod里的容器能够直接获取到这个Pod API对象本身的信息。

示例（9.pod-projected-downwardapi.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: downwardapi
      labels:
        zone: cn-beijing
        cluster: k8s
    spec:
      containers:
        - name: client-container
          image: busybox
          command: ["sh", "-c"]
          args:
          - while true; do
              if [[ -e /etc/podinfo/labels ]]; then
                echo -en '\n\n'; cat /etc/podinfo/labels; fi;
              sleep 5;
            done;
          volumeMounts:
          - name: podinfo
            mountPath: /etc/podinfo
            readOnly: false
      volumes:
        - name: podinfo
          projected:
            sources:
            - downwardAPI:
                items:
                - path: "labels"
                  fieldRef:
                    fieldPath: metadata.labels

在这个Pod的YAML文件中，我定义了一个简单的容器，声明了一个projected类型的Volume。只不过这次Volume的数据来源，变成了Downward API。而这个Downward API Volume，则声明了要暴露Pod的metadata.labels信息给容器。

容器启动后，Pod的Labels字段的值就会被Kubernetes自动挂载成为容器里的/etc/podinfo/labels文件。然后/etc/podinfo/labels里的内容会被不断打印出来。

目前，Downward API支持的字段已经非常丰富了，比如：

使用fieldRef可以获取:

    spec.nodeName - 宿主机名字
    status.hostIP - 宿主机IP
    metadata.name - Pod的名字
    metadata.namespace - Pod Namespace
    status.podIP - Pod的IP
    spec.serviceAccountName - Pod的ServiceAccount的名字
    metadata.uid - Pod的UID
    metadata.labels['<KEY>'] - 指定<KEY>的Label值
    metadata.annotations['<KEY>'] - 指定<KEY>的Annotation值
    metadata.labels - Pod的所有Label
    metadata.annotations - Pod的所有Annotation
 
使用resourceFieldRef可以获取:

    CPU limit
    CPU request
    memory limit
    memory request

> 注意：Downward API只能够获取到Pod容器进程启动之前就能够确定下来的信息。

另外，像持久化存储卷会在存储部分和大家做详细介绍。

### 为Pod添加hosts

示例（10.pod-hostaliases.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      hostAliases:
      - ip: 1.1.1.1
        hostnames:
        - foo.hipstershop.cn
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

> 如果需要为Pod中的容器添加hosts不能手动添加，如果手动添加，Pod重建后hosts会丢失。

### 共享宿主机的Network、IPC 和 PIDNamespace

示例（11.pod-namespace.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      hostNetwork: true
      hostIPC: true
      hostPID: true
      containers:
      - name: busybox
        image: busybox
        args:
        - sleep
        - "600"

在这个Pod中，我定义了共享宿主机的Network、IPC和PID Namespace。这就意味着，这个Pod里的所有容器，会直接使用宿主机的网络、直接与宿主机进行IPC通信、能够看到宿主机里正在运行的所有进程。

### 将Pod调度到特定节点上

首先为Node添加标签

    $ kubectl get nodes
    $ kubectl label nodes <your-node-name> role=web
    $ kubectl get nodes --show-labels

在Kubernetes 1.4版本中，内置了如下标签：

- kubernetes.io/hostname
- beta.kubernetes.io/os
- beta.kubernetes.io/arch

然后编辑Pod配置文件，并执行

示例（12.pod-nodeselector.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      nodeSelector:
        role: web
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

### 亲和性和反亲和性

nodeSelector提供了一种非常简单的方法来将pod调度到有特定标签的节点上。而亲和性和反亲和性特征极大地扩展了可以表达的约束类型。

- 更加表现力的语法（不仅仅是“精确匹配”，语法还支持以下运算符：In，NotIn，Exists，DoesNotExist，Gt，Lt）
- 可以定义“软性”规则，当调度程序不能满足要求是依然能够调度pod（可以表达尽力而为的语义）
- 可以使用node的标签（nodeAffinity），也可以使用其他pod的标签（podAffinity/podAntiAffinity）

nodeAffinity示例：

示例（13.pod-nodeaffinity.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: role
                operator: In
                values:
                - web
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

> requiredDuringSchedulingIgnoredDuringExecution 为“硬”限制，必须满足定义的规则才会将Pod调度到此节点上；
> preferredDuringSchedulingIgnoredDuringExecution 为“软”限制，调度器会尽量将Pod调度到满足要求的节点上，但不保证。

在Kubernetes 1.4版本中引入了Pod间的亲和性和反亲和性。 Pod间的亲和性和反亲和性允许你根据已经在node上运行的pod的标签来限制pod调度在哪个node上，而不是基于node上的标签。

podAffinity和podAntiAffinity示例：

示例（14.pod-podaffinity.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: security
                operator: In
                values:
                - S1
            topologyKey: kubernetes.io/hostname
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: security
                  operator: In
                  values:
                  - S2
              topologyKey: kubernetes.io/hostname
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

对应Pod的亲和性和反亲和性，出于性能和安全性原因topologyKey不允许为空，需要指定一个拓扑范围。在这个示例中，podAffinity使用的是 requiredDuringSchedulingIgnoredDuringExecution，是一个“硬性”限制，Pod的调度必须满足这个条件，即在可用区范围内（kubernetes.io/hostname），这个Pod要和加了标签`security=S1`的Pod运行在一起；并且在宿主机范围内（kubernetes.io/hostname）最好不要（podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution）和加了标签`security=S2`的Pod运行在一起。

### 污点和容忍污点（Taint 和 toleration）

Node亲和性是将pod吸引到一个node节点上。Taint则刚好相反--它们允许一个node排斥一些pod。如果我们不想让Pod调度某些节点上，可以给这个节点设置污点（Taint），这样只有容忍（toleration ）这个污点的Pod才会被调度到此Node上，但是不强制调度到这些节点上。

比如有几个节点是专门跑任务用的，我们可以给这些节点设置污点，这样默认情况下Pod就不会调度到这些节点上，只有容忍这些污点的Pod才会被调度到这些节点上。

    $ kubectl taint nodes <your-node-name> role=job:NoSchedule

如果我们要保证这些Pod一定会调度到这些节点上，我们还需要给这些节点设置label（如：role=job），然后配置Pod的亲和性以强制让这些Pod调度到这些Node。

    $ kubectl label nodes <your-node-name> role=job

我们为节点设置了污点role=job:NoSchedule，只有容忍这个污点的Pod才会调度都此节点上。

示例（15.pod-toleration.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example
      namespace: default
      labels:
        app: example
    spec:
      tolerations:
        - key: "role"
          operator: "Equal"
          value: "job"
          effect: "NoSchedule"
      nodeSelector:
        role: job
      containers:
      - name: nginx
        image: nginx:1.15.12
        imagePullPolicy: IfNotPresent

### PodPreset

有时候我们需要重复为Pod定义某些自动，那有没有一种简单的方法，我们来统一定义某些Pod自动，在其它Pod启动之前自动注入这些字段。

比如我们想给角色为web服务统一添加一个缓存目录，如果我们有很多角色为web的服务，每一个去添加就比较麻烦，我们可以定义PodPreset，自动为Pod注入某些配置。

#### 启用Pod Preset

- 已启用settings.k8s.io/v1alpha1/podpreset API类型。例如，可以通过在API server的 --runtime-config选项中包含settings.k8s.io/v1alpha1=true来完成此操作。
- 已启用PodPreset准入控制器。 可以通过在API server的 --enable-admission-plugins 选项中包含odPreset。Kubernetes 1.10以上版本可以在/etc/kubernetes/manifests/kube-apiserver.yaml中添加--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,Priority,ResourceQuota,PodPreset
- 已经在要使用的命名空间中创建PodPreset对象。

我们可以定义一个PodPreset对象：

示例（16.pod-perset.yaml）：

    apiVersion: settings.k8s.io/v1alpha1
    kind: PodPreset
    metadata:
      name: cache-pod-perset
    spec:
      selector:
        matchLabels:
          role: web
      volumeMounts:
      - mountPath: /cache
        name: cache-volume
      volumes:
      - name: cache-volume
        emptyDir: {}

然后定义一个Pod，在Pod创建时使用上面定义perSet：

示例（17.pod-example-perset.yaml）：

    apiVersion: v1
    kind: Pod
    metadata:
      name: example-pod-perset
      namespace: default
      labels:
        app: example
        role: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest

在这个PodPreset的定义中，首先是一个selector，这就意味着后面这些追加的字段只会作用于selector所选中的Pod对象。

## 生产环境建议

在生成环境使用Pod的一些建议：

- 不要在生成环境直接使用Pod，而是应该使用Deloyment、StatefulSet、DaemonSet、Job等更高级的控制器来控制Pod的调度，如果直接使用Pod，当Pod调度完成后，即使其所在的物理节点不可用也不会重新做调度。
- 要为Pod配置livenessProbe和readinessProbe，做好健康检查，一旦Pod出问题后Kubernetes可以通过恢复策略尽快服务；另外readinessProbe能够保证Pod真正能够提供服务了才会被加入负载均衡对外提供服务。
- Pod中产生的数据最好只写入Volume中，以免对graph driver造成性能压力，另外，如果数据写到Pod里，Pod生命周期结束后数据也会丢失；临时数据最好写emptyDir Volume中，当Pod被销毁时同时销毁临时数据。
- 对Pod做好资源限制，放置Pod无限占用资源影响其他Pod。

