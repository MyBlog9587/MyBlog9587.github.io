---
title: kubernetes准备
date: 2023-05-23 15:40:00
categories:
- kubernetes
- 运维
tags:
- kubernetes
- private
---

{% asset_img back.png 背景图片 %}


# 容器系统的本质
- Linux 和 Unix 操作系统可以通过 chroot 系统调用将子目录变成根目录，达到视图级别的隔离；进程在 chroot 的帮助下可以具有独立的文件系统，对于这样的文件系统进行增删改查不会影响到其他进程。
- 使用 Namespace 技术来实现进程在资源的视图上进行隔离。
- 通过 Cgroup 来限制其资源使用率，设置其能够使用的 CPU 以及内存量。

容器的本质实际上是一个进程，是一个视图被隔离，资源受限的进程; 容器里面 PID=1 的进程就是应用本身。

## 容器得生明周期
使用 docker run 的时候会选择一个镜像来提供独立的文件系统并指定相应的运行程序。这里指定的运行程序称之为 initial 进程，这个 initial 进程启动的时候，容器也会随之启动，
当 initial 进程退出的时候，容器也会随之退出。容器内不只有这样的一个 initial 进程，initial 进程本身也可以产生其他的子进程或者通过 docker exec 产生出来的运维操作，
也属于 initial 进程管理的范围内。当 initial 进程退出的时候，所有的子进程也会随之退出，这样也是为了防止资源的泄漏。

## 容器和 VM 之间的差异
- VM 利用 Hypervisor 虚拟化技术来模拟 CPU、内存等硬件资源，这样就可以在宿主机上建立一个 Guest OS，这是常说的安装一个虚拟机。每一个 Guest OS 都有一个独立的内核
- 容器是针对于进程而言的，因此无需 Guest OS，只需要一个独立的文件系统提供其所需要文件集合即可。


# kubernetes
## 核心功能
- 服务的发现与负载的均衡；
- 容器的自动装箱，我们也会把它叫做 scheduling，就是“调度”，把一个容器放到一个集群的某一个机器上，Kubernetes 会帮助我们去做存储的编排，让存储的声明周期与容器的生命周期能有一个连接；
- Kubernetes 会帮助我们去做自动化的容器的恢复。在一个集群中，经常会出现宿主机的问题或者说是 OS 的问题，导致容器本身的不可用，Kubernetes 会自动地对这些不可用的容器进行恢复；
- Kubernetes 会帮助我们去做应用的自动发布与应用的回滚，以及与应用相关的配置密文的管理；
- 对于 job 类型任务，Kubernetes 可以去做批量的执行；
- Kubernetes 也支持水平的伸缩。

### 调度
观察正在被调度的这个容器的大小、规格（比如说它所需要的 CPU以及它所需要的 memory）。 把用户提交的容器放到 Kubernetes 管理的集群的某一台节点上去；

### 自动修复
节点健康检查，它会监测这个集群中所有的宿主机，当宿主机本身出现故障，或者软件出现故障的时候，这个节点健康检查会自动对它进行发现。把运行在这些失败节点上的容器进行自动迁移，迁移到一个正在健康运行的宿主机上，来完成集群内容器的一个自动恢复。

### 水平伸缩
业务负载检查，监测业务上所承担的负载，如果这个业务本身的 CPU 利用率过高，或者响应时间过长，它可以对这个业务进行一次扩容。

## 主要组件

<img src="主要组件.png" alt="主要组件" style="zoom:100%;" />

## 框架
Master 作为中央的管控节点，会去与 Node 进行一个连接。组件只会和 Master 进行连接，把希望的状态或者想执行的命令下发给 Master，Master 会把这些命令或者状态下发给相应的节点，进行最终的执行。

### Master Node
Master 包含四个主要的组件：API Server、Controller、Scheduler 以及 etcd
- API Server：顾名思义是用来处理 API 操作的，Kubernetes 中所有的组件都会和 API Server 进行连接，组件与组件之间一般不进行独立的连接，都依赖于 API Server 进行消息的传送。API 服 务器会暴露一个 RESTful 的 Kubernetes API 并使用 JSON 格式的清单文件(manifest files)；
- Controller Manager：控制管理器，它用来完成对集群状态的一些管理。它运行着所有处理集群日常任务的控制器。包 括了节点控制器、副本控制器、端点(endpoint)控制器以及服务账户等，比如自动对容器进行修复、自动进行水平扩张，都是由 Kubernetes 中的 Controller 来进行完成的；
- Scheduler：调度器，“调度器”顾名思义就是完成调度的操作，会监控新建的 pods(一组或一个容器)并将其分配给节点。比如把一个用户提交的 Container，依据它对 CPU、对 memory 请求大小，找一台合适的节点，进行放置；
- etcd：是一个分布式的一个存储系统，API Server 中所需要的这些原信息都被放置在 etcd 中，etcd 本身是一个高可用系统，通过 etcd 保证整个 Kubernetes 的 Master 组件的高可用性。

### Worker Node
Node 是真正运行业务负载的，每个业务负载会以 Pod 的形式运行。 Node是集群的工作负载节点，默认情况kubelet会向Master注册自己，一旦Node被纳入集群管理范围，kubelet会定时向Master汇报自身的情报，包括操作系统，Docker版本，机器资源情况等。

如果Node超过指定时间不上报信息，会被Master判断为“失联”，标记为Not Ready，随后Master会触发Pod转移。

- 一个 Pod 中运行的一个或者多个容器，真正去运行这些 Pod 的组件的是叫做 kubelet，也就是 Node 上最为关键的组件，它通过 API Server 接收到所需要 Pod 运行的状态，然后提交到 Container Runtime 组件中。
- Kubernetes 并不会直接进行网络存储的操作，他们会靠 Storage Plugin 或者是网络的 Plugin 来进行操作。用户自己或者云厂商都会去写相应的 Storage Plugin 或者 Network Plugin，去完成存储操作或网络操作。
- 真正完成 service 组网的组件的是 Kube-proxy，它是利用了 iptable 的能力来进行组建 Kubernetes 的 Network，就是 cluster network

#### node组件
- kubelet: Pod的管家，与Master通信，负责调度到对应节点的 Pod 的生命周期管理，执行任务并将 Pod 状态报告给主节点的渠道，通过容器运行时(拉取镜像、启动和停止容器等)来运行这些容器。它 还会定期执行被请求的容器的健康探测程序。
- kube-proxy：实现kubernetes Service的通信与负载均衡机制的重要组件，它负责节点的网络，在主机上维护网络规则并执行连接转发。它还负责对正在服务 的 pods 进行负载平衡。
- Docker：容器的创建和管理Pause



### master 和 node 通信流程
1. 用户可以通过 UI 或者 CLI 提交一个 Pod 给 Kubernetes 进行部署，这个 Pod 请求首先会通过 CLI 或者 UI 提交给 Kubernetes API Server，下一步 API Server 会把这个信息写入到它的存储系统 etcd，之后 Scheduler 会通过 API Server 的 watch 或者叫做 notification 机制得到这个信息：有一个 Pod 需要被调度。
2. 这个时候 Scheduler 会根据它的内存状态进行一次调度决策，在完成这次调度之后，它会向 API Server report 说："OK！这个 Pod 需要被调度到某一个节点上。"
3. 这个时候 API Server 接收到这次操作之后，会把这次的结果再次写到 etcd 中，然后 API Server 会通知相应的节点进行这次 Pod 真正的执行启动。相应节点的 kubelet 会得到这个通知，kubelet 就会去调 Container runtime 来真正去启动配置这个容器和这个容器的运行环境，去调度 Storage Plugin 来去配置存储，network Plugin 去配置网络。




## 核心概念
### Pod
Pod 是 Kubernetes 的一个最小调度以及资源单元。用户可以通过 Kubernetes 的 Pod API 生产一个 Pod，让 Kubernetes 对这个 Pod 进行调度，也就是把它放在某一个 Kubernetes 管理的节点上运行起来。一个 Pod 简单来说是对一组容器的抽象，它里面会包含一个或多个容器。Pod 这个抽象也给这些容器提供了一个共享的运行环境，它们会共享同一个网络环境，这些容器可以用 localhost 来进行直接的连接。而 Pod 与 Pod 之间，是互相有 isolation 隔离的。

### Volume
Volume 就是卷的概念，它是用来管理 Kubernetes 存储的，是用来声明在 Pod 中的容器可以访问文件目录的，一个卷可以被挂载在 Pod 中一个或者多个容器的指定路径下面。一个 Volume 可以去支持多种的后端的存储（本地、分布式、云存储等）。

### Deployment
Deployment 是在 Pod 这个抽象上更为上层的一个抽象，它可以定义一组 Pod 的副本数目、以及这个 Pod 的版本。用 Deployment 这个抽象来做应用的真正的管理，而 Pod 是组成 Deployment 最小的单元。
Kubernetes 是通过 Controller 去维护 Deployment 中 Pod 的数目.

#### 使用场景

- 创建一个Deployment对象来生成对应的Replica Set并完成Pod副本的创建过程。
- 检查Deployment的状态来确认部署动作是否完成（Pod副本的数量是否达到预期值）。
- 更新Deployment以创建新的Pod(例如镜像升级的场景)。
- 如果当前Deployment不稳定，回退到上一个Deployment版本。
- 挂起或恢复一个Deployment。

可以通过kubectl describe deployment来查看Deployment控制的Pod的水平拓展过程

### Service
Service 提供了一个或者多个 Pod 实例的稳定访问地址.

Service定义了一个服务的访问入口地址，前端应用通过这个入口地址访问其背后的一组由Pod副本组成的集群实例，Service与其后端的Pod副本集群之间是通过Label Selector来实现“无缝对接”。RC保证Service的Pod副本实例数目保持预期水平。

Service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹配 labels 的 Pod IP 和端口列表组成 endpoints，由 Kube-proxy 负责将服务
IP 负载均衡到这些 endpoints 上。

每个 Service 都会自动分配一个 cluster IP（仅在集群内部可访问的虚拟地址）和 DNS 名，其他容器可以通过该地址或 DNS 来访问服务，而不需要了解后端容器的运行。


> VIP: Virtual IP 地址

<img src="service.png" alt="service" style="zoom:100%;" />

Service Spec:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  ports:
    - port: 8078 # the port that this service should serve on
  name: http
  # the container on each pod to connect to, can be a name
  # (e.g. 'www') or a number (e.g. 80)
  targetPort: 80
  protocol: TCP
  selector:
    app: nginx
```


### Namespace
Namespace即命名空间，主要用于多租户的资源隔离, Namespace 是用来做一个集群内部的逻辑隔离的，它包括鉴权、资源管理等。比如 Pod、Deployment、Service 都属于一个 Namespace，同一个 Namespace 中的资源需要命名的唯一性，不同的 Namespace 中的资源可以重名。 通过将资源对象分配到不同的Namespace上，便于不同的分组在共享资源的同时可以被分别管理。

k8s集群启动后会默认创建一个“default”的Namespace。可以通过`kubectl get namespaecs`查看。

可以通过`kubectl config use-context namespace`配置当前k8s客户端的环境，通过`kubectl get pods`获取当前namespace的Pod。或者通过`kubectl get pods --namespace=NAMESPACE`来获取指定namespace的Pod。


### Annotation(注解)
Annotation与Label类似，也使用key/value的形式进行定义，Label定义元数据（Metadata）,Annotation定义“附加”信息。

通常Annotation记录信息如下：

- build信息，release信息，Docker镜像信息等。
- 日志库、监控库等。


### Replication Controller(RC)
定义了一个期望的场景。 主要包括：
- Pod期望的副本数（replicas）
- 用于筛选目标Pod的Label Selector
- 用于创建Pod的模板（template）

#### RC特性说明：
- Pod的缩放可以通过以下命令实现：`kubectl scale rc redis-slave --replicas=3`
- 删除RC并不会删除该RC创建的Pod，可以将副本数设置为0，即可删除对应Pod。或者通过`kubectl stop /delete`命令来一次性删除RC和其创建的Pod。
- 改变RC中Pod模板的镜像版本可以实现滚动升级（Rolling Update）。具体操作见https://kubernetes.io/docs/tasks/run-application/rolling-update-replication-controller/
- Kubernetes 1.2以上版本将RC升级为Replica Set，它与当前RC的唯一区别在于Replica Set支持基于集合的Label Selector(Set-based selector)，而旧版本RC只支持基于等式的Label Selector(equality-based selector)。
- Kubernetes 1.2以上版本通过Deployment来维护Replica Set而不是单独使用Replica Set。即控制流为：Delpoyment→Replica Set→Pod。即新版本的Deployment+Replica Set替代了RC的作用。


## 外部系统访问Service的问题
| IP类型     | 说明             |
| ---------- | ---------------- |
| Node IP    | Node节点的IP地址 |
| Pod IP     | Pod的IP地址      |
| Cluster IP | Service的IP地址  |

- NodeIP: 是集群中每个节点的物理网卡IP地址，是真实存在的物理网络，kubernetes集群之外的节点访问kubernetes内的某个节点或TCP/IP服务的时候，需要通过NodeIP进行通信
- Pod IP: 是每个Pod的IP地址，是Docker Engine根据docker0网桥的IP段地址进行分配的，是一个虚拟二层网络，集群中一个Pod的容器访问另一个Pod中的容器，是通过Pod IP进行通信的，而真实的TCP/IP流量是通过Node IP所在的网卡流出的
- Cluster IP:
  1. Service的Cluster IP是一个虚拟IP，只作用于Service这个对象，由kubernetes管理和分配IP地址（来源于Cluster IP地址池）。
  2. Cluster IP无法被ping通，因为没有一个实体网络对象来响应。
  3. Cluster IP结合Service Port组成的具体通信端口才具备TCP/IP通信基础，属于kubernetes集群内，集群外访问该IP和端口需要额外处理。
  4. k8s集群内Node IP 、Pod IP、Cluster IP之间的通信采取k8s自己的特殊的路由规则，与传统IP路由不同。



## 组件简述：

### APIServer：

主要提供以下功能:

-   提供集群管理的 REST API 接口，包括:
    -   认证 Authentication;
    -   授权 Authorization;
    -   准入 Admission(Mutating & Valiating)。
-   提供其他模块之间的数据交互和通信的枢纽(其他模块通过 APIServer 查询或 修改数据，只有 APIServer 才直接操作 etcd)。
-   APIServer 提供 etcd 数据缓存以减少集群对 etcd 的访问。

<img src="apiserver流程.png" alt="apiserver流程" style="zoom:100%;" />

### Controller Manager:

-   Controller Manager 是集群的大脑，是确保整个集群动起来的关键;

-   作用是确保 Kubernetes 遵循声明式系统规范，确保系统的真实状态(Actual

    State)与用户定义的期望状态(Desired State)一致;

-   Controller Manager 是多个控制器的组合，每个 Controller 事实上都是一个 control loop，负责侦听其管控的对象，当对象发生变更时完成配置;

-   Controller 配置失败通常会触发自动重试，整个集群会在控制器不断重试的机 制下确保最终一致性( Eventual Consistency)。


<img src="控制器工作流程.png" alt="控制器工作流程" style="zoom:100%;" />
<img src="informer机制.png" alt="informer机制" style="zoom:100%;" />
<img src="控制器协同原理.png" alt="控制器协同原理" style="zoom:100%;" />

### Scheduler:

特殊的 Controller，工作原理与其他控制器无差别。

Scheduler 的特殊职责在于监控当前集群所有未调度的 Pod，并且获取当前集群所有节点的健康状况和资源 使用情况，为待调度 Pod 选择最佳计算节点，完成调度。

调度阶段分为:

-   Predict:过滤不能满足业务需求的节点，如资源不足、端口冲突等。 • Priority:按既定要素将满足调度需求的节点评分，选择最佳节点。

-   Bind:将计算节点与 Pod 绑定，完成调度。

<img src="调度器流程.png" alt="调度器流程" style="zoom:100%;" />

### Kubelet:

Kubernetes 的初始化系统(init system)

-   从不同源获取 Pod 清单，并按需求启停 Pod 的核心组件:

    -   Pod 清单可从本地文件目录，给定的 HTTPServer 或 Kube-APIServer 等源头获取;

    -   Kubelet 将运行时，网络和存储抽象成了 CRI，CNI，CSI。 • 负责汇报当前节点的资源信息和健康状态;

-   负责 Pod 的健康检查和状态汇报。

<img src="kubelet 流程.png" alt="kubelet 流程" style="zoom:100%;" />

### Kube-Proxy:

-   监控集群中用户发布的服务，并完成负载均衡配置。

-   每个节点的 Kube-Proxy 都会配置相同的负载均衡策略，使得整个集群的服务发现建立在分布式负载 均衡器之上，服务调用无需经过额外的网络跳转(Network Hop)。

-   负载均衡配置基于不同插件实现: 
    -   userspace。
    -   操作系统网络协议栈不同的 Hooks 点和插件: 
        -   iptables
        -   ipvs

<img src="kube-proxy流程.png" alt="kube-proxy流程" style="zoom:100%;" />



## API

Kubernetes API 是由 HTTP+JSON 组成的：用户访问的方式是 HTTP，访问的 API 中 content 的内容是 JSON 格式的。 Kubernetes 的 kubectl 也就是 command tool，Kubernetes UI，或者有时候用 curl，直接与 Kubernetes 进行沟通，都是使用 HTTP + JSON 这种形式。

- API 的 version
- kind, 操作哪个资源, 比如说 kind 是 pod, 在 Metadata 中，就写上这个 Pod 的名字.
- Spec 是希望 Pod 达到的一个预期的状态, 比如说它内部需要有哪些 container 被运行；比如说这里面有一个 nginx 的 container，它的 image 是什么？它暴露的 port 是什么
- status，它表达了从 Kubernetes API 中去获取这个资源当前的状态；比如说一个 Pod 的状态可能是正在被调度、或者是已经 running、或者是已经被 terminates，就是被执行完毕了。
- 在 Metadata 中，有时候也会去写 annotation，也就是对资源的额外的一些用户层次的描述。 叫做“label”，这个 label 可以是一组 KeyValuePair. 这些 label 是可以被 selector，也就是选择器所查询的。
  通过 label，kubernetes 的 API 层就可以对这些资源进行一个筛选，那这些筛选也是 kubernetes 对资源的集合所表达默认的一种方式。


## API 对象
每个 API 对象都有四大类属性：
- TypeMeta
- MetaData
- Spec
- Status

### TypeMeta
Kubernetes对象的最基本定义，它通过引入GKV（Group，Kind，Version）模型定义了一个对象的类型。

1. Group
   Kubernetes 定义了非常多的对象，如何将这些对象进行归类是一门学问，将对象依据其功能范围归入不同的分组，比如把支撑最基本功能的对象归入 core 组，把与应用部署有关的对象归入 apps 组，会使这些对象的可维护性和可理解性更高。

2. Kind
   定义一个对象的基本类型，比如 Node、Pod、Deployment 等。

3. Version
   社区每个季度会推出一个 Kubernetes 版本，随着 Kubernetes 版本的演进，对象从创建之初到能够完全生产化就绪的版本是不断变化的。与软件版本类似，通常社区提出一个模型定义以后，随着该对象不断成熟，其版本可能会从 v1alpha1 到 v1alpha2，或者到 v1beta1，最终变成生产就绪版本 v1


### MetaData
Metadata 中有两个最重要的属性：Namespace 和 Name，分别定义了对象的 Namespace 归属及名字，这两个属性唯一定义了某个对象实例。

常见的 pods, services, replication controllers 和 deployments 等都是属于某一个 Namespace 的（默认是 default），而 Node, persistentVolumes 等则不属于任何 Namespace.

1. Label

   顾名思义就是给对象打标签，一个对象可以有任意对标签，其存在形式是键值对。
   Label 定义了对象的可识别属性，Kubernetes API 支持以 Label 作为过滤条件查询对象。
2. Annotation

   Annotation 与 Label 一样用键值对来定义，但 Annotation 是作为属性扩展，更多面向于系统管理员和开发人员，因此需要像其他属性一样做合理归类。

#### Label
- Label 是识别 Kubernetes 对象的标签，以 key/value 的方式附加到对象上。
- key 最长不能超过 63 字节，value 可以为空，也可以是不超过 253 字节的字符串。
- Label 不提供唯一性，并且实际上经常是很多对象（如 Pods）都使用相同的 label 来标志具体的应用。
- Label 定义好后其他对象可以使用 Label Selector 来选择一组相同 label 的对象
- Label Selector 支持以下几种方式：
    - 等式，如 app=nginx 和 env!=production；
    - 集合，如 env in (production, qa)；
    - 多个 label（它们之间是 AND 关系），如 app=nginx,env=test。

#### Annotations
- Annotations 是 key/value 形式附加于对象的注解。
- 不同于 Labels 用于标志和选择对象，Annotations 则是用来记录一些附加信息，用来辅助应用部署、安全策略以及调度策略等。
- 比如 deployment 使用 annotations 来记录 rolling update 的状态。



3. Finalizer

   Finalizer 本质上是一个资源锁，Kubernetes 在接收某对象的删除请求时，会检查 Finalizer 是否为空，如果不为空则只对其做逻辑删除，即只会更新对象中的metadata.deletionTimestamp 字段。
4. ResourceVersion

   ResourceVersion 可以被看作一种乐观锁，每个对象在任意时刻都有其ResourceVersion，当 Kubernetes 对象被客户端读取以后，ResourceVersion 信息也被一并读取。此机制确保了分布式系统中任意多线程能够无锁并发访问对象，极大提升了系统的整体效率

### Spec 和 Status

- Spec 和 Status 才是对象的核心。
- Spec 是用户的期望状态，由创建对象的用户端来定义。
- Status 是对象的实际状态，由对应的控制器收集实际状态并更新。
- 与 TypeMeta 和 Metadata 等通用属性不同，Spec 和 Status 是每个对象独有的。

### 扩展
#### 必须字段
在对象描述文件.yaml中，必须包含以下字段:
- apiVersion：kubernetes API的版本
- kind：kubernetes对象的类型
- metadata：唯一标识该对象的元数据，包括name，UID，可选的namespace
- spec：标识对象的详细信息，不同对象的spec的格式不同，可以嵌套其他对象的字段。

#### Spec and Status
每个kubernetes对象的结构描述都包含spec和status两个部分。

spec：该内容由用户提供，描述用户期望的对象特征及集群状态。

status：该内容由kubernetes集群提供和更新，描述kubernetes对象的实时状态。

任何时候，kubernetes都会控制集群的实时状态status与用户的预期状态spec一致。

例如：当你定义Deployment的描述文件，指定集群中运行3个实例，那么kubernetes会始终保持集群中运行3个实例，如果任何实例挂掉，kubernetes会自动重建新的实例来保持集群中始终运行用户预期的3个实例。

#### Metadata
- 用来识别资源的标签：Labels
- 用来描述资源的注解；Annotations
- 用来描述多个资源之间相互关系的 OwnerReference

##### labels
资源标签是一种具有标识型的 Key：Value 元数据; 标签主要用来筛选资源和组合资源，可以使用类似于 SQL 查询 select，来根据 Label 查询相关的资源。

1. Label是一个键值对，可以附加在任何对象上，比如Node,Pod,Service,RC等。Label和资源对象是多对多的关系，即一个Label可以被添加到多个对象上，一个对象也可以定义多个Label。
2. Label的作用主要用来实现精细的、多维度的资源分组管理，以便进行资源分配，调度，配置，部署等工作。
3. Label通俗理解就是“标签”，通过标签来过滤筛选指定的对象，进行具体的操作。k8s通过Label Selector(标签选择器)来筛选指定Label的资源对象，类似SQL语句中的条件查询（WHERE语句）。
4. Label Selector有基于等式和基于集合的两种表达方式，可以多个条件进行组合使用。
5. 基于等式：name=redis-slave（匹配name=redis-slave的资源对象）;env!=product(匹配所有不具有标签env=product的资源对象)
6. 基于集合：name in (redis-slave,redis-master);name not in (php-frontend)（匹配所有不具有标签name=php-frontend的资源对象）

###### 使用场景
- kube-controller进程通过资源对象RC上定义的Label Selector来筛选要监控的Pod副本数，从而实现副本数始终保持预期数目。
- kube-proxy进程通过Service的Label Selector来选择对应Pod，自动建立每个Service到对应Pod的请求转发路由表，从而实现Service的智能负载均衡机制。
- kube-scheduler实现Pod定向调度：对Node定义特定的Label，并且在Pod定义文件中使用NodeSelector标签调度策略。


##### Selector
指定选择条件

##### Annotations
一般是系统或者工具用来存储资源的非标示性信息，可以用来扩展资源的 spec/status 的描述

##### Ownereference
一般就是指集合类的资源，集合类资源的控制器会创建对应的归属资源。Ownereference 使得用户可以方便地查找一个创建资源的对象，另外，还可以用来实现级联删除的效果。



### 常用 Kubernetes 对象及其分组
<img src="常用 Kubernetes 对象及其分组.png" alt="常用 Kubernetes 对象及其分组" style="zoom:100%;" />


## Pod 剖析
Pod 就是进程组，也就是 Linux 里的线程组。Pod 在 Kubernetes 里面只有一个逻辑单位，没有一个真实的东西对应说这个就是 Pod. Pod 是 Kubernetes 分配资源的一个单位，因为里面的容器要共享某些资源，所以 Pod 也是 Kubernetes 的原子调度单位。

1. Kubernetes集群中，同宿主机的或不同宿主机的Pod之间要求能够TCP/IP直接通信，因此采用虚拟二层网络技术来实现，例如Flannel，Openvswitch(OVS)等，这样在同个集群中，不同的宿主机的Pod IP为不同IP段的IP，集群中的所有Pod IP都是唯一的，不同Pod之间可以直接通信。
2. Pod有两种类型：普通Pod和静态Pod。静态Pod即不通过K8S调度和创建，直接在某个具体的Node机器上通过具体的文件来启动。普通Pod则是由K8S创建、调度，同时数据存放在ETCD中。
3. Pod IP和具体的容器端口（ContainnerPort）组成一个具体的通信地址，即Endpoint。一个Pod中可以存在多个容器，可以有多个端口，Pod IP一样，即有多个Endpoint。
4. Pod Volume是定义在Pod之上，被各个容器挂载到自己的文件系统中，可以用分布式文件系统实现后端存储功能。
5. Pod中的Event事件可以用来排查问题，可以通过kubectl describe pod xxx 来查看对应的事件。
6. 每个Pod可以对其能使用的服务器上的计算资源设置限额，一般为CPU和Memory。K8S中一般将千分之一个的CPU配置作为最小单位，用m表示，是一个绝对值，即100m对于一个Core的机器还是48个Core的机器都是一样的大小。Memory配额也是个绝对值，单位为内存字节数。
   资源配额的两个参数
7. Pod中可以共享网络和存储（可以简单理解为一个逻辑上的虚拟机，但并不是虚拟机）。 
8. Pod被创建后用一个UID来唯一标识，当Pod生命周期结束，被一个等价Pod替代，UID将重新生成。

Requests: 该资源的最小申请量，系统必须满足要求。

Limits: 该资源最大允许使用量，当超过该量，K8S会kill并重启Pod。

### Pod中容器的运行方式
1. 只运行一个单独的容器

即one-container-per-Pod模式，是最常用的模式，可以把这样的Pod看成单独的一个容器去管理.

2. 运行多个强关联的容器

即sidecar模式，Pod 封装了一组紧耦合、共享资源、协同寻址的容器，将这组容器作为一个管理单元.

### Pod网络

Pod的多个容器是共享网络 Namespace 

- 每个Pod被分配了唯一的IP地址，该Pod内的所有容器共享一个网络空间，包括IP和端口。
- 同个Pod不同容器之间通过localhost通信，Pod内端口不能冲突。
- 不同Pod之间的通信则通过IP+端口的形式来访问到Pod内的具体服务（容器）。


### Pause 容器
每个Pod中有个根容器(Pause容器)，Pause容器的状态代表整个容器组的状态，其他业务容器共享Pause的IP，即Pod IP，共享Pause挂载的Volume，这样简化了同个Pod中不同容器之间的网络问题和文件共享问题。

### 共享网络
Pod 里的多个容器怎么去共享网络？
- 在每个 Pod 里，额外起一个 Infra container 小容器来共享整个 Pod 的 Network Namespace。
- Infra container 是一个非常小的镜像，大概 100~200KB 左右，是一个汇编语言写的、永远处于“暂停”状态的容器。由于有了这样一个 Infra container 之后，其他所有容器都会通过 Join Namespace 的方式加入到 Infra container 的 Network Namespace 中。
- 在 Pod 里面，一定有一个 IP 地址，是这个 Pod 的 Network Namespace 对应的地址，也是这个 Infra container 的 IP 地址。
- 整个 Pod 里面，必然是 Infra container 第一个启动。并且整个 Pod 的生命周期是等同于 Infra container 的生命周期的，与其它容器无关。

> 为什么在 Kubernetes 里面，它是允许去单独更新 Pod 里的某一个镜像的，即：做这个操作，整个 Pod 不会重建，也不会重启，这是非常重要的一个设计。

### 共享存储
可以在Pod中创建共享存储卷的方式来实现不同容器之间数据共享。把 volume 变成了 Pod level。然后所有容器，就是所有同属于一个 Pod 的容器，他们共享所有的 volume。

### 资源限制

Kubernetes 通过 Cgroups 提供容器资源管理的功能，可以限制每个容器的 CPU 和内存使用，比如可以通过下面的命令限制容器最多只用 50% 的 CPU 和 128MB 的内存:

`kubectl set resources deployment nginx-app -c=nginx -- limits=cpu=500m,memory=128Mi`

等同于在每个 Pod 中设置 resources limits

```yaml
apiVersion: v1 
kind: Pod 
metadata:
  labels: 
    app: nginx
  name: nginx 
spec:
  containers:
    - image: nginx
    name: nginx 
    resources:
      limits:
        cpu: "500m" 
        memory: "128Mi"
```

### Sidecar
什么是 Sidecar？就是说其实在 Pod 里面，可以定义一些专门的容器，来执行主业务容器所需要的一些辅助工作;


### Pod的使用
Pod一般是通过各种不同类型的Controller对Pod进行管理和控制，包括自我恢复（例如Pod因异常退出，则会再起一个相同的Pod替代该Pod，而该Pod则会被清除）。也可以不通过Controller单独创建一个Pod，但一般很少这么操作，因为这个Pod是一个孤立的实体，并不会被Controller管理。


#### Controller
Controller是kubernetes中用于对Pod进行管理的控制器，通过该控制器让Pod始终维持在一个用户原本设定或期望的状态。如果节点宕机或者Pod因其他原因死亡，则会在其他节点起一个相同的Pod来替代该Pod。

常用的Controller有：
- Deployment
- StatefulSet
- DaemonSet

Controller是通过用户提供的Pod模板来创建和控制Pod。


### Pod的终止
用户发起一个删除Pod的请求，系统会先发送TERM信号给每个容器的主进程，如果在宽限期（默认30秒）主进程没有自主终止运行，则系统会发送KILL信号给该进程，接着Pod将被删除。

#### Pod终止的流程
1. 用户发送一个删除 Pod 的命令， 并使用默认的宽限期（30s)。
2. 把 API server 上的 pod 的时间更新成 Pod 与宽限期一起被认为 “dead” 之外的时间点。
3. 使用客户端的命令，显示出的Pod的状态为 terminating。
4. （与第3步同时发生）Kubelet 发现某一个 Pod 由于时间超过第2步的设置而被标志成 terminating 状态时， Kubelet 将启动一个停止进程。
   1. 如果 pod 已经被定义成一个 preStop hook，这会在 pod 内部进行调用。
   2. 如果宽限期已经过期但 preStop 锚依然还在运行，将调用第2步并在原来的宽限期上加一个小的时间窗口（2 秒钟）。
   把 Pod 里的进程发送到 TERM信号。
5. （与第3步同时发生），Pod 被从终端的服务列表里移除，同时也不再被 replication controllers 看做时一组运行中的 pods。 在负载均衡（比如说 service proxy）会将它们从轮询中移除前， Pods 这种慢关闭的方式可以继续为流量提供服务。
6. 当宽期限过期时， 任何还在 Pod 里运行的进程都会被 SIGKILL杀掉。
7. Kubelet 通过在 API server 把宽期限设置成0(立刻删除)的方式完成删除 Pod的过程。 这时 Pod 在 API 里消失，也不再能被用户看到。


#### 强制删除Pod
强制删除Pod是指从k8s集群状态和Etcd中立刻删除对应的Pod数据，API Server不会等待kubelet的确认信息。被强制删除后，即可重新创建一个相同名字的Pod。

删除默认的宽限期是30秒，通过将宽限期设置为0的方式可以强制删除Pod。

通过kubectl delete 命令后加--force和--grace-period=0的参数强制删除Pod。

`kubectl delete pod <pod_name> --namespace=<namespace>  --force --grace-period=0`


### 静态pod
静态Pod是由kubelet进行管理，仅存在于特定Node上的Pod。它们不能通过API Server进行管理，无法与ReplicationController、Deployment或DaemonSet进行关联，并且kubelet也无法对其健康检查。

静态Pod总是由kubelet创建，并且总在kubelet所在的Node上运行。

### Pod phase

phase有以下几种值：

| 状态值              | 说明                                                         |
| ------------------- | ------------------------------------------------------------ |
| `挂起（Pending）`   | Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间。 |
| `运行中（Running）` | 该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。 |
| `成功（Succeeded）` | Pod 中的所有容器都被成功终止，并且不会再重启。               |
| `失败（Failed）`    | Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。 |
| `未知（Unknown）`   | 因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。 |


### Pod重启策略
Pod通过restartPolicy字段指定重启策略，重启策略类型为：Always、OnFailure 和 Never，默认为 Always。

restartPolicy 仅指通过同一节点上的 kubelet 重新启动容器。

| 重启策略  | 说明                                                   |
| --------- | ------------------------------------------------------ |
| Always    | 当容器失效时，由kubelet自动重启该容器                  |
| OnFailure | 当容器终止运行且退出码不为0时，由kubelet自动重启该容器 |
| Never     | 不论容器运行状态如何，kubelet都不会重启该容器          |

说明：

可以管理Pod的控制器有Replication Controller，Job，DaemonSet，及kubelet（静态Pod）。
- RC和DaemonSet：必须设置为Always，需要保证该容器持续运行。
- Job：OnFailure或Never，确保容器执行完后不再重启。
- kubelet：在Pod失效的时候重启它，不论RestartPolicy设置为什么值，并且不会对Pod进行健康检查。


### Pod健康检查
#### 探针
Pod的健康状态由两类探针来检查：LivenessProbe和ReadinessProbe。

##### livenessProbe(存活探针)
- 表明容器是否正在运行。
- 如果存活探测失败，则 kubelet 会杀死容器，并且容器将受到其 重启策略的影响。
- 如果容器不提供存活探针，则默认状态为 Success。

###### LivenessProbe参数
- initialDelaySeconds：启动容器后首次进行健康检查的等待时间，单位为秒。
- timeoutSeconds:健康检查发送请求后等待响应的时间，如果超时响应kubelet则认为容器非健康，重启该容器，单位为秒。

###### LivenessProbe三种实现方式
1）ExecAction:在一个容器内部执行一个命令，如果该命令状态返回值为0，则表明容器健康
```yml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: tomcagcr.io/google_containers/busybox
    args:
    - /bin/sh
    - -c
    - echo ok > /tmp/health;sleep 10;rm -fr /tmp/health;sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/health
      initialDelaySeconds: 15
      timeoutSeconds: 1
```
2）TCPSocketAction:通过容器IP地址和端口号执行TCP检查，如果能够建立TCP连接，则表明容器健康。
```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containnerPort: 80
    livenessProbe:
      tcpSocket:
        port: 80
      initialDelaySeconds: 15
      timeoutSeconds: 1
```
3）HTTPGetAction:通过容器的IP地址、端口号及路径调用HTTP Get方法，如果响应的状态码大于等于200且小于等于400，则认为容器健康。
```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-healthcheck
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containnerPort: 80
    livenessProbe:
      httpGet:
        path: /_status/healthz
        port: 80
      initialDelaySeconds: 15
      timeoutSeconds: 1
```

##### readinessProbe(就绪探针)
- 表明容器是否可以正常接受请求。
- 如果就绪探测失败，端点控制器将从与 Pod 匹配的所有 Service 的端点中删除该 Pod 的 IP 地址。
- 初始延迟之前的就绪状态默认为 Failure。
- 如果容器不提供就绪探针，则默认状态为 Success。


探针是 kubelet 对容器执行定期的诊断，主要通过调用容器配置的三类 Handler 实现：

#### Handler的类型：
- Exec Action：在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
- TCP Socket Action：对指定端口上的容器的 IP 地址进行 TCP 检查。如果端口打开，则诊断被认为是成功的。
- HTTP Get Action：对指定的端口和路径上的容器的 IP 地址执行 HTTP Get 请求。如果响应的状态码大于等于200 且小于 400，则诊断被认为是成功的。

探测结果为以下三种之一：
- 成功：容器通过了诊断。
- 失败：容器未通过诊断。
- 未知：诊断失败，因此不会采取任何行动。

#### 探针使用方式
- 如果容器异常可以自动崩溃，则不一定要使用探针，可以由Pod的restartPolicy执行重启操作。
- 存活探针适用于希望容器探测失败后被杀死并重新启动，需要指定restartPolicy 为 Always 或 OnFailure。
- 就绪探针适用于希望Pod在不能正常接收流量的时候被剔除，并且在就绪探针探测成功后才接收流量。

存活探针由 kubelet 来执行，因此所有的请求都在 kubelet 的网络命名空间中进行。

### Pod调度
在kubernetes集群中，Pod（container）是应用的载体，一般通过RC、Deployment、DaemonSet、Job等对象来完成Pod的调度与自愈功能。

#### 1. RC、Deployment:全自动调度
RC的功能即保持集群中始终运行着指定个数的Pod。

在调度策略上主要有：

- 系统内置调度算法[最优Node]
- NodeSelector[定向调度]
- NodeAffinity[亲和性调度]

##### 1.1 NodeSelector[定向调度]
k8s中kube-scheduler负责实现Pod的调度，内部系统通过一系列算法最终计算出最佳的目标节点。如果需要将Pod调度到指定Node上，则可以通过Node的标签（Label）和Pod的nodeSelector属性相匹配来达到目的。

1、`kubectl label nodes {node-name} {label-key}={label-value}`

2、`nodeSelector: {label-key}:{label-value}`

如果给多个Node打了相同的标签，则scheduler会根据调度算法从这组Node中选择一个可用的Node来调度。

如果Pod的nodeSelector的标签在Node中没有对应的标签，则该Pod无法被调度成功。

Node标签的使用场景：

对集群中不同类型的Node打上不同的标签，可控制应用运行Node的范围。例如role=frontend;role=backend;role=database。

##### 1.2 NodeAffinity[亲和性调度]
NodeAffinity 意为Node亲和性调度策略，NodeSelector为精确匹配，NodeAffinity为条件范围匹配，通过In（属于）、NotIn（不属于）、Exists（存在一个条件）、DoesNotExist（不存在）、Gt（大于）、Lt（小于）等操作符来选择Node，使调度更加灵活。

- RequiredDuringSchedulingRequiredDuringExecution：类似于NodeSelector，但在Node不满足条件时，系统将从该Node上移除之前调度上的Pod。
- RequiredDuringSchedulingIgnoredDuringExecution：与上一个类似，区别是在Node不满足条件时，系统不一定从该Node上移除之前调度上的Pod。
- PreferredDuringSchedulingIgnoredDuringExecution：指定在满足调度条件的Node中，哪些Node应更优先地进行调度。同时在Node不满足条件时，系统不一定从该Node上移除之前调度上的Pod。

如果同时设置了NodeSelector和NodeAffinity，则系统将需要同时满足两者的设置才能进行调度.

#### 2. DaemonSet：特定场景调度 
DaemonSet是用于管理在集群中每个Node上仅运行一份Pod的副本实例。
该用法适用的应用场景：

- 在每个Node上运行一个GlusterFS存储或者Ceph存储的daemon进程。
- 在每个Node上运行一个日志采集程序：fluentd或logstach。
- 在每个Node上运行一个健康程序，采集该Node的运行性能数据，例如：Prometheus Node Exportor、collectd、New Relic agent或Ganglia gmond等。

DaemonSet的Pod调度策略与RC类似，除了使用系统内置算法在每台Node上进行调度，也可以通过NodeSelector或NodeAffinity来指定满足条件的Node范围进行调度。

#### 3. Job：批处理调度
可以通过kubernetes Job资源对象来定义并启动一个批处理任务。批处理任务通常并行（或串行）启动多个计算进程去处理一批工作项（work item），处理完后，整个批处理任务结束。

##### 3.1 批处理按任务实现方式不同分为以下几种模式：

- Job Template Expansion模式 一个Job对象对应一个待处理的Work item，有几个Work item就产生几个独立的Job，通过适用于Work item数量少，每个Work item要处理的数据量比较大的场景。例如有10个文件（Work item）,每个文件（Work item）为100G。
- Queue with Pod Per Work Item 采用一个任务队列存放Work item，一个Job对象作为消费者去完成这些Work item，其中Job会启动N个Pod，每个Pod对应一个Work item。
- Queue with Variable Pod Count 采用一个任务队列存放Work item，一个Job对象作为消费者去完成这些Work item，其中Job会启动N个Pod，每个Pod对应一个Work item。但Pod的数量是可变的。

##### 3.2 Job的三种类型
1）Non-parallel Jobs

通常一个Job只启动一个Pod,除非Pod异常才会重启该Pod,一旦此Pod正常结束，Job将结束。

2）Parallel Jobs with a fixed completion count

并行Job会启动多个Pod，此时需要设定Job的.spec.completions参数为一个正数，当正常结束的Pod数量达到该值则Job结束。

3）Parallel Jobs with a work queue

任务队列方式的并行Job需要一个独立的Queue，Work item都在一个Queue中存放，不能设置Job的.spec.completions参数。

此时Job的特性：

- 每个Pod能独立判断和决定是否还有任务项需要处理
- 如果某个Pod正常结束，则Job不会再启动新的Pod
- 如果一个Pod成功结束，则此时应该不存在其他Pod还在干活的情况，它们应该都处于即将结束、退出的状态
- 如果所有的Pod都结束了，且至少一个Pod成功结束，则整个Job算是成功结束


### Pod伸缩
k8s中RC的用来保持集群中始终运行指定数目的实例，通过RC的scale机制可以完成Pod的扩容和缩容（伸缩）.

1. 手动伸缩（scale）

    `kubectl scale rc redis-slave --replicas=3`

2. 自动伸缩（HPA）
   
    Horizontal Pod Autoscaler（HPA）控制器用于实现基于CPU使用率进行自动Pod伸缩的功能。HPA控制器基于Master的kube-controller-manager服务启动参数--horizontal-pod-autoscaler-sync-period定义是时长（默认30秒），周期性监控目标Pod的CPU使用率，并在满足条件时对ReplicationController或Deployment中的Pod副本数进行调整，以符合用户定义的平均Pod CPU使用率。Pod CPU使用率来源于heapster组件，因此需安装该组件。

    可以通过kubectl autoscale命令进行快速创建或者使用yaml配置文件进行创建。创建之前需已存在一个RC或Deployment对象，并且该RC或Deployment中的Pod必须定义resources.requests.cpu的资源请求值，以便heapster采集到该Pod的CPU。

2.1 通过kubectl autoscale创建

php-apache-rc.yaml
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: php-apache
spec:
  replicas: 1
  template:
    metadata:
      name: php-apache
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: gcr.io/google_containers/hpa-example
        resources:
          requests:
            cpu: 200m
        ports:
        - containerPort: 80
```

创建php-apache的RC
```yaml
kubectl create -f php-apache-rc.yaml
```

php-apache-svc.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  ports:
  - port: 80
  selector:
    app: php-apache
```

创建php-apache的Service

`kubectl create -f php-apache-svc.yaml`

创建HPA控制器

`kubectl autoscale rc php-apache --min=1 --max=10 --cpu-percent=50`


2.2 通过yaml配置文件创建

hpa-php-apache.yaml

```yaml
apiVersion: v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: v1
    kind: ReplicationController
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

创建hpa

`kubectl create -f hpa-php-apache.yaml`

查看hpa

`kubectl get hpa`


### Pod滚动升级
k8s中的滚动升级通过执行kubectl rolling-update命令完成，该命令创建一个新的RC（与旧的RC在同一个命名空间中），然后自动控制旧的RC中的Pod副本数逐渐减少为0，同时新的RC中的Pod副本数从0逐渐增加到附加值，但滚动升级中Pod副本数（包括新Pod和旧Pod）保持原预期值。

#### 1. 通过配置文件实现
redis-master-controller-v2.yaml

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis-master-v2
  labels:
    name: redis-master
    version: v2
spec:
  replicas: 1
  selector:
    name: redis-master
    version: v2
  template:
    metadata:
      labels:
        name: redis-master
        version: v2
    spec:
      containers:
      - name: master
        image: kubeguide/redis-master:2.0
        ports:
        - containerPort: 6371
```

注意事项：

- RC的名字（name）不能与旧RC的名字相同
- 在selector中应至少有一个Label与旧的RC的Label不同，以标识其为新的RC。例如本例中新增了version的Label。

运行kubectl rolling-update

`kubectl rolling-update redis-master -f redis-master-controller-v2.yaml`

####  2. 通过kubectl rolling-update命令实现

`kubectl rolling-update redis-master --image=redis-master:2.0`

与使用配置文件实现不同在于，该执行结果旧的RC被删除，新的RC仍使用旧的RC的名字

#### 3. 升级回滚

kubectl rolling-update加参数--rollback实现回滚操作

`kubectl rolling-update redis-master --image=kubeguide/redis-master:2.0 --rollback`


### 如何通过 Pod 对象定义支撑应用运行
环境变量：
- 直接设置值；
- 读取 Pod Spec 的某些属性；
- 从 ConfigMap 读取某个值；
- 从 Secret 读取某个值。

<img src="Pod定义支撑应用.png" alt="Pod定义支撑应用" style="zoom:100%;" />




## 传感器
控制循环中逻辑的传感器主要由 Reflector、Informer、Indexer 三个组件构成
- Reflector 通过 List 和 Watch K8s server 来获取资源的数据。Reflector 在获取新的资源数据后，会在 Delta 队列中塞入一个包括资源对象信息本身以及资源对象事件类型的 Delta 记录，Delta 队列中可以保证同一个对象在队列中仅有一条记录，从而避免 Reflector 重新 List 和 Watch 的时候产生重复的记录。
  - List 用来在 Controller 重启以及 Watch 中断的情况下，进行系统资源的全量更新；
  - Watch 则在多次 List 之间进行增量的资源更新；
- Informer 组件不断地从 Delta 队列中弹出 delta 记录，然后把资源对象交给 indexer
- indexer 把资源记录在一个缓存中，缓存在默认设置下是用资源的命名空间来做索引的，并且可以被 Controller Manager 或多个 Controller 所共享。之后，再把这个事件交给事件的回调函数

控制循环中的控制器组件主要由事件处理函数以及 worker 组成，事件处理函数之间会相互关注资源的新增、更新、删除的事件，并根据控制器的逻辑去决定是否需要处理。对需要处理的事件，会把事件关联资源的命名空间以及名字塞入一个工作队列中，并且由后续的 worker 池中的一个 Worker 来处理，工作队列会对存储的对象进行去重，从而避免多个 Woker 处理同一个资源的情况。
Worker 在处理资源对象时，一般需要用资源的名字来重新获得最新的资源数据，用来创建或者更新资源对象，或者调用其他的外部服务，Worker 如果处理失败的时候，一般情况下会把资源的名字重新加入到工作队列中，从而方便之后进行重试。

## ReplicaSet (副本集)

ReplicaSet 是一个用来描述无状态应用的扩缩容行为的资源， ReplicaSet controler 通过监听 ReplicaSet 资源来维持应用希望的状态数量，ReplicaSet 中通过 selector 来匹配所关联的 Pod

- 其允许用户定义 Pod 的副本数，每一个 Pod 都会被当作一
个无状态的成员进行管理，Kubernetes 保证总是有用户期望的数量的 Pod 正常运行。
- 当某个副本宕机以后，控制器将会创建一个新的副本。
- 当因业务负载发生变更而需要调整扩缩容时，可以方便地调整副本数量。

## Service

Service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹 配 labels 的 Pod IP 和端口列表组成 endpoints，由 Kube-proxy 负责将服务 IP 负载均衡到这些 endpoints 上。

每个 Service 都会自动分配一个 cluster IP(仅在集群内部可访问的虚拟地址) 和 DNS 名，其他容器可以通过该地址或 DNS 来访问服务，而不需要了解后端 容器的运行。

## Deployment (部署)

每个 Deployment 其实是管理的一组相同的应用 Pod，这组 Pod 我们认为它是相同的一个副本。
- Deployment 定义了一种 Pod 期望数量
- 配置 Pod 发布方式，也就是说 controller 会按照用户给定的策略来更新 Pod，而且更新过程中，也可以设定不可用 Pod 数量在多少范围内
- 如果更新过程中发生问题的话，即所谓“一键”回滚，也就是说你通过一条命令或者一行修改能够将 Deployment 下面所有 Pod 更新为某一个旧版本 

换而言之：
- 部署表示用户对 Kubernetes 集群的一次更新操作。
- 部署是一个比 RS 应用模式更广的 API 对象，可以是创建一个新的服务，更新一个新的服务，也可以是滚动升级一个服务。
- 滚动升级一个服务，实际是创建一个新的 RS，然后逐渐将新 RS 中副本数增加到理想状态，将旧 RS 中的副本数减小到 0 的
复合操作。
- 这样一个复合操作用一个 RS 是不太好描述的，所以用一个更通用的 Deployment 来描述。
- 以 Kubernetes 的发展方向，未来对所有长期伺服型的的业务的管理，都会通过 Deployment 来管理。

<img src="kubernetes-deployment.png" alt="kubernetes-deployment" style="zoom:100%;" />


### 语法
- apiVersion：apps/v1 也就是说 Deployment 当前所属的组是 apps，版本是 v1
- metadata 是我们看到的 Deployment 元信息，也就是Labels、Selector、Pod.image等
- Deployment.spec 中首先要有一个核心的字段 
  - replicas，这里定义期望的 Pod 数量
  - selector 其实是 Pod 选择器，那么所有扩容出来的 Pod，它的 Labels 必须匹配 selector 层上的 image.labels
  - template 
    - metadata: 期望 Pod 的 metadata，其中包含了 labels，即跟 selector.matchLabels 相匹配的一个 Labels；
    - spec: Pod.spec 其实是 Deployment 最终创建出来 Pod 的时候，它所用的 Pod.spec
```yml
apiVersion: apps/v1         # Deployment 当前所属的组是 apps，版本是 v1
kind: Deployment            # Deployment 元信息
metadata:
  name: nginx-deployment    
  labels: 
    app: nginx
spec:
  replicas: 3               # 期望Pod数量
  selector:
    matchLabels:            # Pod得选择器
      app: nginx
  template:                 # Pod模板
    metadata: 
      labels: 
        app: nginx
    spec: 
      containers: 
      - name: nginx
        image: nginx:1.7.9
        port:
        - containerPort: 80
```

## StatefulSet(有状态服务集)

- 对于 StatefulSet 中的 Pod，每个 Pod 挂载自己独立的存储，如果一个 Pod 出现故障，从其他节点启动一个同样名字的
Pod，要挂载上原来 Pod 的存储继续以它的状态提供服务。
- 适合于 StatefulSet 的业务包括数据库服务 MySQL 和 PostgreSQL，集群化管理服务 ZooKeeper、etcd 等有状态服务。
- 使用 StatefulSet，Pod 仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储来提供高可靠性，
StatefulSet 做的只是将确定的 Pod 与确定的存储关联起来保证状态的连续性。

<img src="kubernetes-statefulset.png" alt="kubernetes-statefulset" style="zoom:100%;" />


### Statefulset 与 Deployment 的差异

- 身份标识
  - StatefulSet Controller 为每个 Pod 编号，序号从0开始。
- 数据存储
  - StatefulSet 允许用户定义 volumeClaimTemplates，Pod 被创建的同时，Kubernetes 会以
  volumeClaimTemplates 中定义的模板创建存储卷，并挂载给 Pod。
- StatefulSet 的升级策略不同
  - onDelete
  - 滚动升级
  - 分片升级


## Job(任务)
### 用途
- Job 是一个管理任务的控制器，它可以创建一个或多个 Pod 来指定 Pod 的数量，并可以监控它是否成功地运行或终止
- 可以根据 Pod 的状态来给 Job 设置重置的方式及重试的次数
- 可以根据依赖关系，保证上一个任务运行完成之后再运行下一个任务
- 可以控制任务的并行度，根据并行度来确保 Pod 运行过程中的并行次数和总体完成大小

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec: 
      containers:
      - name: pi
        image: perl
        command: ["perl", "....."]
      restartPolicy: Nerver
  backoffLimit: 4
```

### 区别
- kind 叫 Job
- restartPolicy，在 Job 里面我们可以设置 Never、OnFailure、Always 这三种重试策略。在希望 Job 需要重新运行的时候，我们可以用 Never；希望在失败的时候再运行，再重试可以用 OnFailure；或者不论什么情况下都重新运行时 Alway
- Job 在运行的时候不可能去无限的重试，所以需要一个参数来控制重试的次数。这个 backoffLimit 就是来保证一个 Job 到底能重试多少次
- ownerReferences 明此 pod 是归哪个上一层 controller 来管理
- completions 指定本 Pod 队列执行次数，即这个任务一共会被执行几次
- parallelism 代表这个并行执行的个数，其实就是一个管道或者缓冲器中缓冲队列的大小。组合 completions 即为 Job 一定要执行 n 次，每次并行 m 个 Pod, 一共执行n/m个批次。

#### restartPolicy 重启策略

#### backoffLimit 重试次数限制

### 状态
- AGE: 指这个 Pod 从当前时间算起，减去它当时创建的时间
- DURATION: 主要来看Job 里面的实际业务到底运行了多长时间
- COMPLETIONS 主要来看任务里面这个 Pod 一共有几个，然后它其中完成了多少个状态

### Cronjob
#### 跟job 区别
- schedule：主要是设置时间格式
- startingDeadlineSeconds：即：每次运行 Job 的时候，它最长可以等多长时间
- concurrencyPolicy：是否允许并行运行，如果这个 policy 设置为 true 的话，那么不管你前面的 Job 是否运行完成，每分钟都会去执行；如果是 false，它就会等上一个 Job 运行完成之后才会运行下一个；
- JobsHistoryLimit：每一次 CronJob 运行完之后，它都会遗留上一个 Job 的运行历史、查看时间，即设置历史存留数

### 流程
1. 所有的 job 都是一个 controller，它会 watch 这个 API Server，我们每次提交一个 Job 的 yaml 都会经过 api-server 传到 ETCD 里面去，然后 Job Controller 会注册几个 Handler，每当有添加、更新、删除等操作的时候，它会通过一个内存级的消息队列，发到 controller 里面。
2. 通过 Job Controller 检查当前是否有运行的 pod，如果没有的话，通过 Scale up 把这个 pod 创建出来；如果有的话，或者如果大于这个数，对它进行 Scale down，如果这时 pod 发生了变化，需要及时 Update 它的状态。
3. 同时要去检查它是否是并行的 job，或者是串行的 job，根据设置的配置并行度、串行度，及时地把 pod 的数量给创建出来。最后，它会把 job 的整个的状态更新到 API Server 里面去，这样就能看到呈现出来的最终效果了。

## DaemonSet(后台支撑服务集)
DaemonSet 也是 Kubernetes 提供的一个 default controller，它实际是做一个守护进程的控制器，它能帮我们做到以下几件事情：
- 首先能保证集群内的每一个节点都运行一组相同的 pod；
- 同时还能根据节点的状态保证新加入的节点自动创建对应的 pod；
- 在移除节点的时候，能删除对应的 pod；
- 而且它会跟踪每个 pod 的状态，当这个 pod 出现异常、Crash 掉了，会及时地去 recovery 这个状态。

### 区别
- kind: DaemonSet
- RollingUpdate：会一个一个的更新。先更新第一个 pod，然后老的 pod 被移除，通过健康检查之后再去见第二个 pod，这样对于业务上来说会比较平滑地升级，不会中断
- OnDelete：模板更新之后，pod 不会有任何变化，需要我们手动控制。我们去删除某一个节点对应的 pod，它就会重建，不删除的话它就不会重建，这样的话对于一些我们需要手动控制的特殊需求也会有特别好的作用。

### 流程
1. 当有 node 状态节点发生变化时，它会通过一个内存消息队列发进来，然后DaemonSet controller 会去 watch 这个状态，看一下各个节点上是都有对应的 Pod，如果没有的话就去创建。当然它会去做一个对比，如果有的话，它会比较一下版本，然后加上刚才提到的是否去做 RollingUpdate？如果没有的话就会重新创建，Ondelete 删除 pod 的时候也会去做 check 它做一遍检查，是否去更新，或者去创建对应的 pod。
2. 当然最后的时候，如果全部更新完了之后，它会把整个 DaemonSet 的状态去更新到 API Server 上，完成最后全部的更新。

## 配置管理
- 可变配置就用 ConfigMap；
- 敏感信息是用 Secret；
- 身份认证是用 ServiceAccount 这几个独立的资源来实现的；
- 资源配置是用 Resources；
- 安全管控是用 SecurityContext；
- 前置校验是用 InitContainers 

> 这几个在 spec 里面加的字段，来实现的这些配置管理。

![image](https://user-images.githubusercontent.com/32731294/134451744-15489695-bd4b-4d0e-ba92-7dee35c8b146.png)

### ConfigMap
主要管理一些可变配置信息（比如配置文件、环境变量、命令行参数等），它的好处在于它可以让一些可变配置和容器镜像进行解耦，这样也保证了容器的可移植性。

- ConfigMap 用来将非机密性的数据保存到键值对中。
- 使用时， Pods 可以将其用作环境变量、命令行参数或者存储卷中的配置文件。
- ConfigMap 将环境配置信息和 容器镜像解耦，便于应用配置的修改。

#### 格式
```yml
apiVersion: v1
kind: ConfigMap
metadata:                 # ConfigMap 元信息
  labels:
    app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-system
data:                     # 管理配置文件
  # 输入键值对
  special.now: very
  special.type: charm
  # 配置文件
  cni-conf.json: |        # 文件名作为key
    {
      ...                 # 文件内容作为 value
    }
  net-conf.json: |        # 配置文件
    {
      ...
    }
```

#### 创建
用 kubectl 命令来创建，它带的参数主要有两个：一个是指定 name，第二个是 DATA。其中 DATA 可以通过指定文件或者指定目录，以及直接指定键值对。
> 指定文件的话，文件名就是 Map 中的 key，文件内容就是 Map 中的 value。然后指定键值对就是指定数据键值对，即：key:value 形式，直接映射到 Map 的key:value。

##### 1. 通过yaml文件方式
cm-appvars.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appvars
data:
  apploglevel: info
  appdatadir: /var/data
```

常用命令:

- kubectl create -f cm-appvars.yaml
- kubectl get configmap
- kubectl describe configmap cm-appvars
- kubectl get configmap cm-appvars -o yaml


##### 2. 通过kubectl命令行方式

通过kubectl create configmap创建，使用参数--from-file或--from-literal指定内容，可以在一行中指定多个参数。

1）通过--from-file参数从文件中进行创建，可以指定key的名称，也可以在一个命令行中创建包含多个key的ConfigMap。

`kubectl create configmap NAME --from-file=[key=]source --from-file=[key=]source`

2）通过--from-file参数从目录中进行创建，该目录下的每个配置文件名被设置为key，文件内容被设置为value。

`kubectl create configmap NAME --from-file=config-files-dir`

3）通过--from-literal从文本中进行创建，直接将指定的key=value创建为ConfigMap的内容。

`kubectl create configmap NAME --from-literal=key1=value1 --from-literal=key2=value2`

容器应用对ConfigMap的使用有两种方法：

- 通过环境变量获取ConfigMap中的内容。
- 通过Volume挂载的方式将ConfigMap中的内容挂载为容器内部的文件或目录。


##### 3. 通过环境变量的方式

ConfigMap的yaml文件:cm-appvars.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cm-appvars
data:
  apploglevel: info
  appdatadir: /var/data
```

Pod的yaml文件：cm-test-pod.yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cm-test-pod
spec:
  containers:
  - name: cm-test
    image: busybox
    command: ["/bin/sh","-c","env|grep APP"]
    env:
    - name: APPLOGLEVEL
      valueFrom:
        configMapKeyRef:
          name: cm-appvars
          key: apploglevel
    - name: APPDATADIR
      valueFrom:
        configMapKeyRef:
          name: cm-appvars
          key: appdatadir
```

创建命令：

- kubectl create -f cm-test-pod.yaml
- kubectl get pods --show-all
- kubectl logs cm-test-pod


#### 使用
![image](https://user-images.githubusercontent.com/32731294/134452608-d6c85c75-dd38-409b-8159-f63d54a38fd1.png)

- 第一种是环境变量。环境变量的话通过 valueFrom，然后 ConfigMapKeyRef 这个字段，下面的 name 是指定 ConfigMap 名，key 是 ConfigMap.data 里面的 key。这样的话，在 busybox 容器启动后容器中执行 env 将看到一个 SPECIAL_LEVEL_KEY 环境变量；
- 第二个是命令行参数。命令行参数其实是第一行的环境变量直接拿到 cmd 这个字段里面来用；
- 最后一个是通过 volume 挂载的方式直接挂到容器的某一个目录下面去。上面的例子是把 special-config 这个 ConfigMap 里面的内容挂到容器里面的 /etc/config 目录下，这个也是使用的一种方式。

#### 使用ConfigMap的限制条件
- ConfigMap必须在Pod之前创建
- ConfigMap也可以定义为属于某个Namespace。只有处于相同Namespace中的Pod可以引用它。
- kubelet只支持可以被API Server管理的Pod使用ConfigMap。静态Pod无法引用。
- 在Pod对ConfigMap进行挂载操作时，容器内只能挂载为“目录”，无法挂载为文件。

#### 注意点
- ConfigMap 文件的大小。虽然说 ConfigMap 文件没有大小限制，但是在 ETCD 里面，数据的写入是有大小限制的，现在是限制在 1MB 以内；
- 第二个注意点是 pod 引入 ConfigMap 的时候，必须是相同的 Namespace 中的 ConfigMap，前面其实可以看到，ConfigMap.metadata 里面是有 namespace 字段的；
- 第三个是 pod 引用的 ConfigMap。假如这个 ConfigMap 不存在，那么这个 pod 是无法创建成功的，其实这也表示在创建 pod 前，必须先把要引用的 ConfigMap 创建好；
- 第四点就是使用 envFrom 的方式。把 ConfigMap 里面所有的信息导入成环境变量时，如果 ConfigMap 里有些 key 是无效的，比如 key 的名字里面带有数字，那么这个环境变量其实是不会注入容器的，它会被忽略。但是这个 pod 本身是可以创建的。这个和第三点是不一样的方式，是 ConfigMap 文件存在基础上，整体导入成环境变量的一种形式；
- 最后一点是：什么样的 pod 才能使用 ConfigMap？这里只有通过 K8s api 创建的 pod 才能使用 ConfigMap，比如说通过用命令行 kubectl 来创建的 pod，肯定是可以使用 ConfigMap 的，但其他方式创建的 pod，比如说 kubelet 通过 manifest 创建的 static pod，它是不能使用 ConfigMap 的。

### 密钥对象（Secret）
主要用来存储密码 token 等一些敏感信息的资源对象。其中，敏感信息是采用 base-64 编码保存.

- Secret 是用来保存和传递密码、密钥、认证凭证这些敏感信息的对象。
- 使用 Secret 的好处是可以避免把敏感信息明文写在配置文件里。
- Kubernetes 集群中配置和使用服务不可避免的要用到各种敏感信息实现登录、认证等功能，例如访问 AWS 存储的用户名密码。
- 为了避免将类似的敏感信息明文写在所有需要使用的配置文件中，可以将这些信息存入一个 Secret 对象，而在配置文件中通过 Secret 对象引用这些敏感信息。
- 这种方式的好处包括：意图明确，避免重复，减少暴漏机会。


#### 格式
```yml
apiVersion: v1
kind: Secret
metadata:                 # Secret 元数据
  name: mysectet
  namespace: kube-system
type: Opaque              # Secret 类型
data:                     # secret 存储的数据
  username: adfas
  password: daf23cfdsa
```

>  data 是存储的 Secret 的数据，它也是 key-value 的形式存储的。

Secret 常用的四种类型：(默认是 Opaque 类型)
- Opaque，它是普通的 Secret 文件；
- service-account-token，是用于 service-account 身份认证用的 Secret；
- dockerconfigjson，这是拉取私有仓库镜像的用的一种 Secret；
- bootstrap.token，是用于节点接入集群校验用的 Secret。

#### 使用
![image](https://user-images.githubusercontent.com/32731294/134616763-2f1f21cb-31bd-498d-b8bd-acb8f1112fc2.png)
![image](https://user-images.githubusercontent.com/32731294/134616884-0dce7b85-db3b-4927-897f-62dcc746b440.png)

#### 注意点
1. Secret 的文件大小限制。这个跟 ConfigMap 一样，也是 1MB；
2. Secret 采用了 base-64 编码，但是它跟明文也没有太大区别。所以说，如果有一些机密信息要用 Secret 来存储的话，还是要很慎重考虑。也就是说谁会来访问你这个集群，谁会来用你这个 Secret，还是要慎重考虑，因为它如果能够访问这个集群，就能拿到这个 Secret。
   如果是对 Secret 敏感信息要求很高，对加密这块有很强的需求，推荐可以使用 Kubernetes 和开源的 vault做一个解决方案，来解决敏感信息的加密和权限管理。
3. Secret 读取的最佳实践，建议不要用 list/watch，如果用 list/watch 操作的话，会把 namespace 下的所有 Secret 全部拉取下来，这样其实暴露了更多的信息。推荐使用 GET 的方法，这样只获取你自己需要的那个 Secret。


### 用户 (User Account)

用户帐户为人提供账户标识，而服务账户为计算机进程和 Kubernetes 集群中运行的 Pod 提供账户标识。

用户帐户和服务帐户的一个区别是作用范围：
- 用户帐户对应的是人的身份，人的身份与服务的 Namespace 无关，所以用户账户是跨Namespace 的；
- 而服务帐户对应的是一个运行中程序的身份，与特定 Namespace 是相关的。


### 服务帐户 (Service Account)
用于解决 pod 在集群里面的身份认证问题，身份认证信息是存在于 Secret 里面

#### 使用
![image](https://user-images.githubusercontent.com/32731294/134630772-2a26ba61-9192-4ac8-85d2-b03100c2a552.png)
![image](https://user-images.githubusercontent.com/32731294/134631130-31935223-d572-4046-a452-9099033b1de8.png)

### Resources
容器的一个资源配置管理。

#### 支持类型
目前内部支持类型有三种：CPU、内存，以及临时存储。也可以自己来定义，但配置时，指定的数量必须为整数。
- CPU：单位：millicore （1 core = 1000millicore）
- Memory：单位：Byte
- ephemeral storage（临时存储）：单位：Byte
- 自定义资源：配置时必须为整数

#### 配置方法
目前资源配置主要分成 request 和 limit 两种类型，一个是需要的数量，一个是资源的界限。
- CPU:
    spec.containers[].resources.limitts.cpu
    spec.containers[].resources.requests.cpu
- Memeory:
    spec.containers[].resources.limitts.memory
    spec.containers[].resources.requests.memory
- ephemeral storage（临时存储）
    spec.containers[].resources.limitts.ephemeral-storage
    spec.containers[].resources.requests.ephemeral-storage

#### 使用
```yml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: wp
    image: wordpress
    resources:
      requests:                 # 申明需要的资源
        memory: "64Mi"
        cpu: "250m"
        ephemeral-storage: "2Gi"
     limits:                    # 申明需要资源的边界
      requests:
        memory: "128Mi"
        cpu: "500m"
        ephemeral-storage: "4Gi"       
```

根据 CPU 对容器内存资源的需求，我们对 pod 的服务质量进行一个分类，分别是 Guaranteed、Burstable 和 BestEffort。
- Guaranteed ：pod 里面每个容器都必须有内存和 CPU 的 request 以及 limit 的一个声明，且 request 和 limit 必须是一样的，这就是 Guaranteed；
- Burstable：Burstable 至少有一个容器存在内存和 CPU 的一个 request；
- BestEffort：只要不是 Guaranteed 和 Burstable，那就是 BestEffort。

> 资源配置好后，当这个节点上 pod 容器运行，比如说节点上 memory 配额资源不足，kubelet会把一些低优先级的，或者说服务质量要求不高的（如：BestEffort、Burstable）pod 驱逐掉。它们是按照先去除 BestEffort，再去除 Burstable 的一个顺序来驱逐 pod 的。

### SecurityContext
用于限制容器的一个行为，它能保证系统和其他容器的安全。**这一块的能力不是 Kubernetes 或者容器 runtime 本身的能力，而是 Kubernetes 和 runtime 通过用户的配置，最后下传到内核里，再通过内核的机制让 SecurityContext 来生效。**

SecurityContext 主要分为三个级别：
- 容器级别，仅对容器生效；
- pod 级别，对指定 pod 里所有容器生效；
- 集群级别，就是 PSP，对集群内所有 pod 生效。

权限和访问控制设置项，现在一共列有七项（这个数量后续可能会变化）：
- Discretionary Access Control: 根据用户ID 和组 ID 来控制文件访问权限；
- SELinux：通过SELinux的策略配置来控制用户或者进程对文件的访问控制；
- privileged: 容器是否为特权模式；
- Linux Capabilities: 给特定进程配置privileged能力；
- AppArmor：通过一些配置文件来控制可执行文件的一个访问控制权限，比如说一些端口的读写；
- Seccomp: 控制进程可以操作的系统调用；
- AllowPrivilegeEscalcation: 控制一个进程是否能获取比父亲更多的权限的一个限制。

### InitContainer
主要为普通 container 服务

#### InitContainer 和普通 container 的区别
- InitContainer 首先会比普通 container 先启动，并且直到所有的 InitContainer 执行成功后，普通 container 才会被启动；
- InitContainer 之间是按定义的次序去启动执行的，执行成功一个之后再执行第二个，而普通的 container 是并发启动的；
- InitContainer 执行成功后就结束退出，而普通容器可能会一直在执行。


## 存储卷

- 通过存储卷可以将外挂存储挂载到 Pod 内部使用。
- 存储卷定义包括两个部分: Volume 和 VolumeMounts。
  - Volume：定义 Pod 可以使用的存储卷来源；
  - VolumeMounts：定义存储卷如何 Mount 到容器内部。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-volume
spec:
  containers:
    - image: nginx:1.15
  name: nginx
  volumeMounts:
    - name: data
  mountPath: /data
  volumes:
    - name: data
    emptyDir: {}
```

### Volumes
#### Volume的功能
1. Volume是Pod中能够被多个容器访问的共享目录，可以让容器的数据写到宿主机上或者写文件到网络存储中
2. 可以实现容器配置文件集中化定义与管理，通过ConfigMap资源对象来实现

#### Volume的特点
k8s中的Volume与Docker的Volume相似，但不完全相同。

1. k8s上Volume定义在Pod上，然后被一个Pod中的多个容器挂载到具体的文件目录下。
2. k8s的Volume与Pod生命周期相关而不是容器是生命周期，即容器挂掉，数据不会丢失但是Pod挂掉，数据则会丢失。
3. k8s中的Volume支持多种类型的Volume：Ceph、GlusterFS等分布式系统。

#### Volume的使用方式
先在Pod上声明一个Volume，然后容器引用该Volume并Mount到容器的某个目录.

#### Volume分类
- 本地存储，常用的有 emptydir/hostpath；
- 网络存储：网络存储当前的实现方式有两种，
  1. 一种是 in-tree，它的实现的代码是放在 K8s 代码仓库中的，随着k8s对存储类型支持的增多，这种方式会给k8s本身的维护和发展带来很大的负担；
  2. 第二种实现方式是 out-of-tree，它的实现其实是给 K8s 本身解耦的，通过抽象接口将不同存储的driver实现从k8s代码仓库中剥离，因此out-of-tree 是后面社区主推的一种实现网络存储插件的方式；
- Projected Volumes：它其实是将一些配置信息，如 secret/configmap 用卷的形式挂载在容器中，让容器中的程序可以通过POSIX接口来访问配置数据；
- PV 与 PVC

> Volume定义在Pod上，属于“计算资源”的一部分，而Persistent Volume和Persistent Volume Claim是网络存储，简称PV和PVC，可以理解为k8s集群中某个网络存储中对应的一块存储

##### emptyDir
emptyDir Volume是在Pod分配到Node时创建的，初始内容为空，无须指定宿主机上对应的目录文件，由K8S自动分配一个目录，当Pod被删除时，对应的emptyDir数据也会永久删除

###### 作用：
1. 临时空间，例如程序的临时文件，无须永久保留
2. 长时间任务的中间过程CheckPoint的临时保存目录
3. 一个容器需要从另一个容器中获取数据的目录（即多容器共享目录）

###### 说明：
目前用户无法设置emptyVolume的使用介质，如果kubelet的配置使用硬盘则emptyDir将创建在该硬盘上。


##### hostPath
hostPath是在Pod上挂载宿主机上的文件或目录。

###### 作用：
1. 容器应用日志需要持久化时，可以使用宿主机的高速文件系统进行存储
2. 需要访问宿主机上Docker引擎内部数据结构的容器应用时，可以通过定义hostPath为宿主机/var/lib/docker目录，使容器内部应用可以直接访问Docker的文件系统。

###### 注意点：
1. 在不同的Node上具有相同配置的Pod可能会因为宿主机上的目录或文件不同导致对Volume上目录或文件的访问结果不一致。
2. 如果使用了资源配额管理，则kubernetes无法将hostPath在宿主机上使用的资源纳入管理。


#### PV（Persistent Volumes）
##### 来源 
Pod Volumes 很难准确地表达它的复用/共享语义，对它的扩展也比较困难。因此 K8s 中又引入了 Persistent Volumes 概念，它可以将存储和计算分离，通过不同的组件来管理存储资源和计算资源，然后解耦 pod 和 Volume 之间生命周期的关联。这样，当**把 pod 删除之后，它使用的PV仍然存在，还可以被新建的 pod 复用**。

##### 特点
- PV是网络存储，不属于任何Node，但可以在每个Node上访问。
- PV不是定义在Pod上，而是独立于Pod之外定义。
- PV常见类型：GCE Persistent Disks、NFS、RBD等。

##### 状态类型(PV是有状态的对象)
- Available: 空闲状态
- Bound: 已经绑定到某个PVC上
- Released: 对应的PVC已经删除，但资源还没有回收
- Failed: PV自动回收失败

#### PVC（PersistentVolumeClaim）
##### 来源
为什么有了 PV 又设计了 PVC 呢？主要原因是为了简化K8s用户对存储的使用方式，做到职责分离。
- 职责分离：PVC中只用声明自己需要的存储size、access mode（单node独占还是多node共享？只读还是读写访问？）等业务真正关心的存储需求（不关心存储实现细节），PV和其对应的后端存储信息由cluster admin 统一运维和管控，安全访问策略更容易实现。
- PVC简化了User对存储的需求，PV才是存储的实际信息的承载体，通过kube-controller-manager 中的PersisentVolumeController 将PVC与合适的PV bound到一起，从而满足User 对存储的实际需求。
- PVC像是面向对象编程中抽象出来的接口，PV是接口对应的实现。

##### **分类：**

###### Static Volume Provisioning
由集群管理员事先去规划这个集群中的用户会怎样使用存储，它会先预分配一些存储，也就是预先创建一些 PV；然后用户在提交自己的存储需求（也就是 PVC）的时候，K8s 内部相关组件会帮助它把 PVC 和 PV 做绑定；之后用户再通过 pod 去使用存储的时候，就可以通过 PVC 找到相应的 PV，它就可以使用了。

**不足:**

​	首先需要集群管理员预分配，预分配其实是很难预测用户真实需求的



###### Dynamic Volume Provisioning
集群管理员不预分配 PV，写一个模板文件，这个模板文件是用来表示创建某一类型存储（块存储，文件存储等）所需的一些参数，这些参数是用户不关心的，给存储本身实现有关的参数。用户只需要提交自身的存储需求，也就是PVC文件，并在 PVC 中指定使用的存储模板（StorageClass）。
K8s 集群中的管控组件，会结合 PVC 和 StorageClass 的信息动态，生成用户所需要的存储（PV），将 PVC 和 PV 进行绑定后，pod 就可以使用 PV 了。通过 StorageClass 配置生成存储所需要的存储模板，再结合用户的需求动态创建 PV 对象，做到按需分配，在没有增加用户使用难度的同时也解放了集群管理员的运维工作。

#### 使用
![image](https://user-images.githubusercontent.com/32731294/134868640-220b1532-5742-40b9-92ae-447f0fc5db77.png)

- 在 pod yaml 文件中的 Volumes 字段中，声明我们卷的名字以及卷的类型。声明的两个卷，一个是用的是 emptyDir，另外一个用的是 hostPath，这两种都是本地卷。在容器中应该怎么去使用这个卷呢？它其实可以通过 volumeMounts 这个字段，volumeMounts 字段里面指定的 name 其实就是它使用的哪个卷，mountPath 就是容器中的挂载路径。
- 两个容器都指定使用了同一个卷，就是这个 cache-volume。那么，在多个容器共享同一个卷的时候，为了隔离数据，我们可以通过 subPath 来完成这个操作。它会在卷里面建立两个子目录，然后容器 1 往 cache 下面写的数据其实都写在子目录 cache1 了，容器 2 往 cache 写的目录，其数据最终会落在这个卷里子目录下面的 cache2 下。
- readOnly 的意思其实就是只读挂载。
- emptyDir 其实是在 pod 创建的过程中会临时创建的一个目录，这个目录随着 pod 删除也会被删除，里面的数据会被清空掉；hostPath 顾名思义，其实就是宿主机上的一个路径，在 pod 删除之后，这个目录还是存在的，它的数据也不会被丢失。

![image](https://user-images.githubusercontent.com/32731294/134884853-1a5d3d91-a6d3-49a0-9eb9-60666941566d.png)
![image](https://user-images.githubusercontent.com/32731294/134884892-fe42c3a0-3fde-45fa-80fb-6894369e91ef.png)

#### 结构解析
- Capacity：存储总空间
- AccessModes：PV访问策略控制列表（必须同PVC得访问策略控制列表匹配才能绑定 bound）
  - ReadWriteOnce：只允许单 node 读写访问；
  - ReadOnlyMany：允许多个 node 只读访问，是常见的一种数据的共享方式；
  - ReadWriteMany：允许多个 node 读写访问。
> k8s 集群中的相关组件是如何去找到合适的 PV 呢？首先它是通过为 PV 建立的 AccessModes 索引找到所有能够满足用户的 PVC 里面的 AccessModes 要求的 PV list，然后根据PVC的 Capacity，StorageClassName, Label Selector 进一步筛选 PV，如果满足条件的 PV 有多个，选择 PV 的 size 最小的，accessmodes 列表最短的 PV，也即最小适合原则。
- ReclaimPolicy：PV 被 release 之后（与之bound得PVC 被删除）回收再利用策略
  - Recycle（已废弃）
  - Delete: volume 被released 之后直接delete， 需要volume plugin支持
  - Retain：默认策略，由系统管理员来手动管理该volume
- StorageClassName：PVC可通过该字段找到相同值得PV（静态provisioning），也可通过该字段对应的storageclass从而动态provisioning新PV对象。
- NodeAffinity：限制可以访问该volume 的nodes。



#### PV 、PVC 区别:

-   PV 描述的，是持久化存储数据卷。这个 API 对象主要定义的是一个持久化存储在宿主机上的目录，比如一个 NFS 的挂载目录。PV 对象是由运维人员事先创建在 Kubernetes 集群里待用的。

    ```yaml
    apiVersion: v1
    kind: PersistentVolume
    metadata:
      name: nfs
    spec:
      storageClassName: manual
      capacity:
        storage: 1Gi
      accessModes:
        - ReadWriteMany
      nfs:
        server: 10.244.1.4
        path: "/"
    ```

-   PVC 描述的，则是 Pod 所希望使用的持久化存储的属性。比如，Volume 存储的大小、可读写权限等等。PVC 对象通常由开发人员创建；或者以 PVC 模板的方式成为 StatefulSet 的一部分，然后由 StatefulSet 控制器负责创建带编号的 PVC。

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: nfs
    spec:
      accessModes:
        - ReadWriteMany
      storageClassName: manual
      resources:
        requests:
          storage: 1Gi
    ```



#### PV、PVC 绑定前提：

-   第一个条件，当然是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
-   第二个条件，则是 PV 和 PVC 的 storageClassName 字段必须一样。

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    role: web-frontend
spec:
  containers:
  - name: web
    image: nginx
    ports:
      - name: web
        containerPort: 80
    volumeMounts:
        - name: nfs
          mountPath: "/usr/share/nginx/html"
  volumes:
  - name: nfs
    persistentVolumeClaim:
      claimName: nfs
```

PVC 可以理解为持久化存储的“接口”，它提供了对某种持久化存储的描述，但不提供具体的实现；而这个持久化存储的实现部分则由 PV 负责完成。

`PersistentVolumeController` 会不断地查看当前每一个 PVC，是不是已经处于 `Bound`（**已绑定**）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个 "单身" 的 PVC 进行绑定。这样，Kubernetes 就可以保证用户提交的每一个 PVC，只要有合适的 PV 出现，它就能够很快进入绑定状态。PV 与 PVC 进行 **"绑定"**，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 `spec.volumeName` 字段上。所以，接下来 Kubernetes 只要获取到这个 PVC 对象，就一定能够找到它所绑定的 PV。



#### **PV持久化两阶段：**

-   当一个 Pod 调度到一个节点上之后，kubelet 就要负责为这个 Pod **创建**它的 Volume 目录，把这个阶段称为 **Attach**， 对于“第一阶段”（Attach），Kubernetes 提供的可用参数是 `nodeName`，即宿主机的名字。
-   将磁盘设备**格式化并挂载**到 Volume 宿主机目录的操作，对应的正是“两阶段处理”的第二个阶段称为：**Mount**, 对于“第二阶段”（Mount），Kubernetes 提供的可用参数是 `dir`，即 Volume 的宿主机目录。

经过了“两阶段处理”，就得到了一个“持久化”的 Volume 宿主机目录。所以，接下来，kubelet 只要把这个 Volume 目录通过 CRI 里的 Mounts 参数，传递给 Docker，然后就可以为 Pod 里的容器挂载这个“持久化”的 Volume 了。其实，这一步相当于执行了如下所示的命令：

```bash
$ docker run -v /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>:/<容器内的目标目录> 我的镜像 ...
```



​	**功能划分：**

​	关于 PV 的“两阶段处理”流程，是靠独立于 kubelet 主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的。

​	“第一阶段”的 **Attach**（以及 Dettach）操作，是由 Volume Controller 负责维护的，这个控制循环的名字叫作：`AttachDetachController`。而它的作用，就是不断地检查每一个 Pod 对应的 PV，和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作。Volume Controller 自然是 kube-controller-manager 的一部分。所以，AttachDetachController 也一定是运行在 Master 节点上的.

​	“第二阶段”的 **Mount**（以及 Unmount）操作，必须发生在 Pod 对应的宿主机上，所以它必须是 kubelet 组件的一部分。这个控制循环的名字，叫作：`VolumeManagerReconciler`，它运行起来之后，是一个独立于 kubelet 主循环的 Goroutine。



### **StorageClass:**

自动创建 PV 的机制，即：`Dynamic Provisioning`。相比之下，人工管理 PV 的方式就叫作 `Static Provisioning`。`Dynamic Provisioning` 机制工作的核心，在于一个名叫 `StorageClass` 的 API 对象。**而 StorageClass 对象的作用，其实就是创建 PV 的模板**。

`StorageClass` 对象会定义如下两个部分内容：

-   第一，PV 的属性。比如，存储类型、Volume 的大小等等。

-   第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

    有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。只有同属于一个 StorageClass 的 PV 和 PVC，才可以绑定在一起.



**总结为：**

​	用户提交请求创建pod，Kubernetes发现这个pod声明使用了PVC，靠`PersistentVolumeController`帮它找一个PV配对。 没有现成的PV，就去找对应的`StorageClass`，帮它新创建一个PV，然后和PVC完成绑定。 新创建的PV，还只是一个API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用： 第一阶段由运行在`master`上的`AttachDetachController`负责，为这个PV完成 `Attach` 操作，为宿主机挂载远程磁盘； 第二阶段是运行在每个节点上kubelet组件的内部，把第一步`attach`的远程磁盘 mount 到宿主机目录。这个控制循环叫`VolumeManagerReconciler`，运行在独立的Goroutine，不会阻塞kubelet主循环。 完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。



**本地磁盘用作PV 举例：**

1.   把本地磁盘定义成 PV：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    # 指定这个 PV 对应的本地磁盘的路径
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          # 如果 Pod 要想使用这个 PV，那它就必须运行在 node-1 上
          - node-1
```

​	PV 创建后，进入了 Available（可用）状态
```bash
$ kubectl create -f local-pv.yaml 
persistentvolume/example-pv created

$ kubectl get pv
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY  STATUS      CLAIM             STORAGECLASS    REASON    AGE
example-pv   5Gi        RWO            Delete           Available                     local-storage             16s
```

2.   创建一个 `StorageClass`来描述这个 PV

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  # 指定的是 no-provisioner。这是因为 Local Persistent Volume 目前尚不支持 Dynamic Provisioning，所以它没办法在用户创建 PVC 的时候，就自动创建出对应的 PV
  name: local-storage
provisioner: kubernetes.io/no-provisioner
# WaitForFirstConsumer 延迟绑定
volumeBindingMode: WaitForFirstConsumer
```

​	`WaitForFirstConsumer` 告诉 Kubernetes 里的 Volume 控制循环：虽然已经发现这个 `StorageClass` 关联的 PVC 与 PV 可以绑定在一起，但请**不要现在**就执行绑定操作（即：设置 PVC 的 `VolumeName` 字段）。而要等到**第一个声明使用**该 PVC 的 Pod 出现在调度器之后，调度器再综合考虑所有的调度规则，当然也包括每个 PV 所在的节点位置，来统一决定，这个 Pod 声明的 PVC，到底应该跟哪个 PV 进行绑定, 从而保证了这个绑定结果不会影响 Pod 的正常调度。

```bash
$ kubectl create -f local-sc.yaml 
storageclass.storage.k8s.io/local-storage created
```

3.   定义PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  # 指定 StorageClass 
  storageClassName: local-storage
```

```bash
$ kubectl create -f local-pvc.yaml 
persistentvolumeclaim/example-local-claim created

$ kubectl get pvc
NAME                  STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS    AGE
example-local-claim   Pending                                       local-storage   7s
```

​	可以看到已经存在了一个可以与 PVC 匹配的 PV，但这个 PVC 依然处于 Pending 状态，也就是等待绑定的状态.

4.   编写一个 Pod 来声明使用这个 PVC

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: example-pv-pod
spec:
  volumes:
    # 指定 PVC
    - name: example-pv-storage
      persistentVolumeClaim:
       claimName: example-local-claim
  containers:
    - name: example-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: example-pv-storage
```

```bash
$ kubectl create -f local-pod.yaml 
pod/example-pv-pod created

$ kubectl get pvc
NAME                  STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS    AGE
example-local-claim   Bound     example-pv   5Gi        RWO            local-storage   6h
```

​	此时，PVC 会立刻变成 Bound 状态，与前面定义的 PV 绑定在一起. 也就是说，在创建的 Pod 进入调度器之后，“绑定”操作才开始进行



## CSI:












## CustomResourceDefinition(CRD) 自定义资源定义

- CRD 就像数据库的开放式表结构，允许用户自定义 Schema。
- 有了这种开放式设计，用户可以基于 CRD 定义一切需要的模型，满足不同业务的需求。
- 社区鼓励基于 CRD 的业务抽象，众多主流的扩展应用都是基于 CRD 构建的，比如 Istio、Knative。
- 甚至基于 CRD 推出了 Operator Mode 和 Operator SDK，可以以极低的开发成本定义新对象，并构建新对象的控制器。


### operator
    字面意思是“承担运维任务的程序”， 它们的基本逻辑都是一样的：时刻盯着资源状态，一有变化马上作出反应（也就是 reconcile 逻辑）


## 健康检查

Kubernetes 作为一个面向应用的集群管理工具，需要确保容器在部署后确实处在正常的运行状态。
1. 探针类型：
   - LivenessProbe
     - 探测应用是否处于健康状态，如果不健康则删除并重新创建容器。
   - ReadinessProbe
     - 探测应用是否就绪并且处于正常服务状态，如果不正常则不会接收来自 Kubernetes Service 的流量。
   - StartupProbe
     - 探测应用是否启动完成，如果在 failureThreshold*periodSeconds 周期内未就绪，则会应用进程会被重启。

2. 探活方式：
   - Exec
   - TCP socket
   - HTTP

### 健康检查 spec

<img src="健康检查spec.png" alt="健康检查spec" style="zoom:100%;" />


## 开放接口

- 容器运行时接口（CRI）：提供计算资源
- 容器网络接口（CNI）：提供网络资源
- 容器存储接口（CSI），提供存储资源


### CRI  容器运行时接口（Container Runtime Interface）


### CNI  容器网络接口（Container Network Interface）


### CSI  容器存储接口（Container Storage Interface）



### Informer
![image](https://user-images.githubusercontent.com/32731294/164966654-9d60a039-58e8-469c-8984-a5016b40937a.png)

`Informer` 组件一直在监听 `etcd` 中 `Pod` 信息的变化，准确来说，监听的是Pod信息中 `Spec.nodeName` 字段的变化，一旦检测到该字段为空，则认为集群中有`Pod`尚未调度到`Node`中，这时`Informer` 开始将这个 `Pod` 的信息加入队列中，同时更新 `Scheduler Cache` 缓存。接下来 `Pod` 信息从队列中出队，进入 `Predicates（预选阶段）`，该阶段通过一系列的预选算法选出集群中适合`Pod`运行的节点，带着这些信息进入`Priorities（优选阶段）`。同理，该阶段通过一系列的优选算法为适合该 `Pod` 调度对每个 `Node` 进行打分，最后选出集群中最适合（也就是分数最高的）Pod运行的一个节点，最后将这个节点和Pod进行绑定（`Bind`），更新缓存，从而实现Pod的调度。


```golang
// 为单个 pod 执行整个调度工作流程。它被序列化在调度算法的适合的主机上
func (sched *Scheduler) scheduleOne(ctx context.Context) {
  ...
  scheduleResult, err := sched.Algorithm.Schedule(schedulingCycleCtx, prof, state, pod)
  ...
}
```

调用 pkg/scheduler/core/generic_scheduler.go:
```golang
func (g *genericScheduler) Schedule(ctx context.Context, prof *profile.Profile, state *framework.CycleState, pod *v1.Pod) (result ScheduleResult, err error) {
  ...
  // 谓词评估开始时间
	startPredicateEvalTime := time.Now()
	// filter 过滤
	// 根据框架过滤器插件和过滤器扩展器找到适合 pod 的节点
	feasibleNodes, filteredNodesStatuses, err := g.findNodesThatFitPod(ctx, prof, state, pod)
	if err != nil {
		return result, err
	}
	// 计算谓词完成
	trace.Step("Computing predicates done")
  
  ...
  
  // 优先评估开始时间
	startPriorityEvalTime := time.Now()
	// When only one node after predicate, just use it.
	// 当谓词后面只有一个节点时，直接使用，不再进行后续打分等操作
	if len(feasibleNodes) == 1 {
		metrics.DeprecatedSchedulingAlgorithmPriorityEvaluationSecondsDuration.Observe(metrics.SinceInSeconds(startPriorityEvalTime))
		return ScheduleResult{SuggestedHost: feasibleNodes[0].Name,
			EvaluatedNodes: 1 + len(filteredNodesStatuses),
			FeasibleNodes:  1,
		}, nil
	}

	// score 打分
	priorityList, err := g.prioritizeNodes(ctx, prof, state, pod, feasibleNodes)
	if err != nil {
		return result, err
	}
  
  ...
  
  // 7.选择节点
	host, err := g.selectHost(priorityList)
	trace.Step("Prioritizing done")

	return ScheduleResult{
		SuggestedHost:  host,
		EvaluatedNodes: len(feasibleNodes) + len(filteredNodesStatuses),
		FeasibleNodes:  len(feasibleNodes),
	}, err
}
```

#### Predicates（预选阶段）
`predicates` 算法主要是对集群中的 `node` 进行过滤，选出符合当前 pod 运行的一组 nodes。过滤 node 的预选算法有很多,预选算法执行顺序如下：

```golang

```

#### Priorities（优选阶段）

### factory
Factory包含了众多的ListerAndWatcher，主要是为了 watch apiserver，及时获取最新的资源。

两个重要成员PodQueue和PodLister：
- PodQueue存放着等待kube-scheduler调度的pod
- PodLister=schedulerCache


## **RBAC:**






















