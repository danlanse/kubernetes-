

## 名词
* Endpoint（IP+Port） 
标识服务进程的访问点；
* Master
集群控制节点，负责整个集群的管理和控制，基本上Kubernetes所有的控制命令都是发给它，它来负责具体的执行过程，我们后面所有执行的命令基本都是在Master节点上运行的；
  * Kubernetes API Server（kube-apiserver）
  提供Http Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程；
  * Kubernetes Controller Manager（kube-controller-manager）
  Kubernetes里所有资源对象的自动化控制中心，可以理解为资源对象的“大总管”；
  * Kubernetes Scheduler（kube-scheduler）
  负责资源调度（Pod调度）的进程，相当于公交公司的“调度室”；
  * etcd Server
  Kubernetes里所有的资源对象的数据全部是保存在etcd中；
  
* Node 
除了Master，Kubernetes集群中的其他机器被称为Node节点，Node节点才是Kubernetes集群中的工作负载节点，每个Node都会被Master分配一些工作负载（Docker容器），当某个Node宕机，其上的工作负载会被Master自动转移到其他节点上去； 
  * kubelet，负责Pod对应的容器的创建、启停等任务，同时与Master节点密切协作，实现集群管理的基本功能。一旦Node被纳入集群管理范围，kubelet进程就会想Master几点汇报自身的情报，这样Master可以获知每个Node的资源使用情况，并实现高效均衡的资源调度策略。而某个Node超过指定时间不上报信息，会被Master判定为“失联”，Node状态被标记为不可用（Not Ready），随后Master会触发“工作负载大转移”的自动流程；
  * kube-proxy，实现Kubernetes Service的通信与负载均衡机制的重要组件；
  * Docker Engine（docker），Docker引擎，负责本机的容器创建和管理工作；

* Pod 
  * 每个Pod都有一个特殊的被称为“根容器”的Pause容器，还包含一个或多个紧密相关的用户业务容器；
  * 一个Pod里的容器与另外主机上的Pod容器能够直接通信；
  * 如果Pod所在的Node宕机，会将这个Node上的所有Pod重新调度到其他节点上；
  * 普通Pod及静态Pod，前者存放在etcd中，后者存放在具体Node上的一个具体文件中，并且只能在此Node上启动运行；
  * Docker Volume对应Kubernetes中的Pod Volume；
  * 每个Pod可以设置限额的计算机资源有CPU和Memory； 
	* Requests，资源的最小申请量；
	* Limits，资源最大允许使用的量；

![avater](https://img-blog.csdn.net/20170421095943116?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2V5c2lsZW5jZTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
![avater](https://img-blog.csdn.net/20170421103557869?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2V5c2lsZW5jZTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

* Event 
  * 是一个事件记录，记录了事件最早产生的时间、最后重复时间、重复次数、发起者、类型，以及导致此事件的原因等信息。Event通常关联到具体资源对象上，式排查故障的重要参考信息；
* Label 
  * Label可以附加到各种资源对象上，一个资源对象可以定义任意数量的Label。给某个资源定义一个Label，相当于给他打一个标签，随后可以通过Label Selector（标签选择器）查询和筛选拥有某些Label的资源对象。我们可以通过给指定的资源对象捆绑一个或多个Label来实现多维度的资源分组管理功能，以便于灵活、方便的进行资源分配、调度、配置、部署等管理工作； 
  * Label Selector示例：select * from pod where pod’s name=’XXX’,env=’YYY’，支持操作符有=、!=、in、not in；
* Replication Controller（RC） 
部署和升级Pod，声明某种Pod的副本数量在任意时刻都符合某个预期值； 
	* Pod期待的副本数；
	* 用于筛选目标Pod的Label Selector；
	* 当Pod副本数量小于预期数量的时候，用于创建新Pod的Pod模板（template）；
* Replica Set 
  * 下一代的Replication Controlle，Replication Controlle只支持基于等式的selector（env=dev或environment!=qa）但Replica Set还支持新的、基于集合的selector（version in (v1.0, v2.0)或env notin (dev, qa)），这对复杂的运维管理带来很大方便。
* Deployment 
拥有更加灵活强大的升级、回滚功能。在新的版本中，官方推荐使用Replica Set和Deployment来代替RC，两者相似度>90%，相对于RC一个最大升级是我们随时指导当前Pod“部署”的进度。Deployment使用了Replica Set，除非需要自定义升级功能或根本不需要升级Pod，一般情况下，我们推荐使用Deployment而不直接使用Replica Set； 
典型使用场景： 
 * 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建过程；
 * 检查更新Deployment的状态来查看部署动作是否完成（Pod副本的数量是否达到预期的值）；
 * 更新Deployment以创建新的Pod；（比如镜像升级）
 * 如果当前Deployment不稳定，则回滚到一个早先的Deployment版本；
 * 挂起或者回复一个Deployment；
* Horizontal Pod Autoscaler（HPA） 
意思是Pod横向自动扩容，目标是实现自动化、智能化扩容或缩容。 
Pod负载度量指标： 
  * CPUUtilizationPercentage 
通常使用一分钟内的平均值，可以通过Heapster扩展组件获取这个值。一个Pod自身的CPU利用率是该Pod当前CPU的使用量除以它的Pod Request的值。例如Pod Request定义值为0.4，当前Pod使用量为0.2，则它的CPU使用率为50%。但如果没有定义Pod Request值，则无法使用CPUUtilizationPercentage来实现Pod横向自动扩容的能力；
应用程序自定义的度量指标，比如服务在每秒内的相应的请求书（TPS或QPS）
  * Service 
Service其实就是我们经常提起的微服务架构中的一个“微服务”，通过分析、识别并建模系统中的所有服务为微服务——Kubernetes Service，最终我们的系统由多个提供不同业务能力而又彼此独立的微服务单元所组成，服务之间通过TCP/IP进行通信，从而形成了我们强大而又灵活的弹性网络，拥有了强大的分布式能力、弹性扩展能力、容错能力； 

![avator](https://img-blog.csdn.net/20170424105142400?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQva2V5c2lsZW5jZTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如图示，每个Pod都提供了一个独立的Endpoint（Pod IP+ContainerPort）以被客户端访问，多个Pod副本组成了一个集群来提供服务，一般的做法是部署一个负载均衡器来访问它们，为这组Pod开启一个对外的服务端口如8000，并且将这些Pod的Endpoint列表加入8000端口的转发列表中，客户端可以通过负载均衡器的对外IP地址+服务端口来访问此服务。运行在Node上的kube-proxy其实就是一个智能的软件负载均衡器，它负责把对Service的请求转发到后端的某个Pod实例上，并且在内部实现服务的负载均衡与会话保持机制。Service不是共用一个负载均衡器的IP地址，而是每个Servcie分配一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP。

* Node IP
Node节点的IP地址，是Kubernetes集群中每个节点的物理网卡的IP地址，是真是存在的物理网络，所有属于这个网络的服务器之间都能通过这个网络直接通信；
* Pod IP
Pod的IP地址，是Docker Engine根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络，位于不同Node上的Pod能够彼此通信，需要通过Pod IP所在的虚拟二层网络进行通信，而真实的TCP流量则是通过Node IP所在的物理网卡流出的；
* Cluster IP 
Service的IP地址。特性如下： 
  * 仅仅作用于Kubernetes Servcie这个对象，并由Kubernetes管理和分配IP地址；
  * 无法被Ping，因为没有一个“实体网络对象”来响应；
  * 只能结合Service Port组成一个具体的通信端口；
  * Node IP网、Pod IP网域Cluster IP网之间的通信，采用的是Kubernetes自己设计的一种编程方式的特殊的路由规则，与IP路由有很大的不同；
* volume

注：Node、Pod、Replication Controller和Service等都可以看作是一种“资源对象”，几乎所有的资源对象都可以通过Kubernetes提供的kubectl工具执行增、删、改、查等操作并将其保存在ectd中持久化存储。

## 优点
* 轻装上阵，小团队即可轻松应付从设计实现到运维的复杂的分布式系统；
* 全面拥抱微服务架构；
* 迁移方便，可以很方便的搬迁到公有云或基于OpenStack的私有云上；
* 超强的横向扩容能力；

