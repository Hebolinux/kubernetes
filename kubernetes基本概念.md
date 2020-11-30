### Service（服务发现）

在kubernetes上，pod具有生命周期，某一pod所在的节点宕机时，此pod会在其他节点上重建，但重建的pod有自己的网络名称空间、UTS等，与原来的pod并不相同，为了使用户能够正确的访问到这些pod的地址，kubernetes提供了一种服务发现机制

在kubernetes上，每新建一个pod，此pod做的第一件事就是先到一个固定位置注册自身的服务，这就是服务发现，重建pod时会先与原有pod在服务发现中注册的信息建立关联关系，用户要访问服务时也不是直接去找pod，而是先到服务发现，然后查找此重建的pod，如果在一定时间内此重建的pod没有与服务发现建立关联关系，则会将此pod提供的服务从服务发现中移除，而用户如果在服务发现中访问不到此pod提供的服务，则会去找一个新的服务提供者

为了尽可能降低用户与pod之间的协调的复杂度，kubernetes为每一组提供同类服务的pod与其客户端之间提供一个中间层service，service只要不删除，那它的名称和地址就是固定的，当客户端需要写在配置文件中访问某个服务时，只需要在配置文件中写入service的地址或名称即可，而service不但能提供一个稳定的访问入口，还是一个调度器，service收到客户端的请求后将其代理到后端的pod上

service则是通过label selector关联pod对象的，关联pod对象后再通过动态检测发现pod的IP和端口，作为自己可以调度的后端服务对象

### Ingress

Service为一组Pod提供对外访问接口时只能对OSI的第4层提供流量调度，表现形式为ip+port，Ingress是k8s集群在OSI的第7层的应用，负责对外暴露接口，Ingress可以调度不同业务域、不同URL访问路径的业务流量，实现将不同域的流量调度到指定的Pod中

#### AddOns（附件）

再kubernetes上，service只是iptables一个DNAT规则，而service的IP也仅存在于DNAT规则中，不配置在任何一个网卡上，所以service的IP是ping不通的，因为没有TCP/IP协议栈，但却可以正常请求作服务端。在kubernetes中，service并不只有一个，以LNMP为例来说，nginx的Pod可能有多个，用户访问nginx的Pod时需要通过与其关联的servcie，而nginx访问后端的mysql时，也必须通过与mysql相关联的service

service作为kubernetes的一个对象来说，service有自己的名称，且service的名称也相当于一个服务的名称，可以把service名称解析为一个IP，而名称解析则需要通过DNS实现，所以kubernetes部署完成第一件事就需要在kubernetes上部署一个DNS的pod，以确保各service名称能被解析，这种pod属于kubernetes自身的服务就需要用到的pod，属于基础性系统架构型的pod，也被称为附件，它并不作为程序本身的一部分存在

> 核心附件
>
> - CNI网络插件：flannel/calico
> - 服务发现用插件：coredns
> - 服务暴露用插件：traefik
> - GUI管理插件：Dashboard

### Kubernetes的网络模型

kubernetes要求整个集群有3种网络模型：节点网络 ---代理--> 集群网络 ---代理--> Pod网络

1. Pod网络：各Pod运行在同一网络中
2. 集群网络：service运行在一个网络，与Pod的网络分离，Pod内部的网络名称空间可以ping通，但service不行
3. 节点网络：各节点处于一个网络

kubernetes集群内的Pod有3种通信模型：

1. Pod：同一Pod内的多个容器间通过loop环回接口通信
2. 各个Pod之间的通信：在kubernetes集群中即便是不同节点上的Pod，也是可以直接使用Pod的地址进行通信的，所以处于同一网段中的Pod不允许地址冲突
3. Pod与Service之间的通信：每一台主机上都有iptables规则，只要创建一个service，此service是需要反映在整个集群中的每一个节点上的，那么每一个nodes上都应该有相应的iptables规则修改，而与Service之间的通信，只需要容器报文的目标地址指向网关，如docker 0桥，而docker 0桥要访问service地址则需要查询iptables规则表

#### kube-proxy

Service随时也有可能会变动，比如删除Service、创建Service或Service后的Pod改变，如果Service后的Pod改变了，那么Service规则中的DIP也需要改变，而Service如何检测Pod是否发生改变，则是需要用到Label Selector

Service如何改变所有节点上相关的地址规则，则需要kube-proxy来实现，kube-proxy随时与API Server进行通信，因为每一个Pod发生改变之后，这个结果都需要保存到API Server中，API Server通信发生改变后会生成一个通知事件，此事件可以被每一个关联的组件接收到，比如kube-proxy，一旦发现某一Service后的Pod发生了改变、地址发生了改变，那么对应的由kube-proxy在本地将改变的IP地址反映到iptables或ipvs规则中，Service管理需要通过kube-proxy实现

#### kubernetes网络模型示例

```shell
-----------------------------------------------------------------------------
Service网络                            |                              |
-----------------------------------------------------------------------------
    |      Pod网络      |     |        |                |     |       |
    |                 pod   pod   kube-proxy          pod   pod   kube-proxy
  Master               |     |        |                |     |       |
   节点                ----------------------          ----------------------
    |                      Worker 节点1                     Worker 节点2
    |      节点网络              |                                 |
-----------------------------------------------------------------------------    
```



### Etcd

kubernetes集群中基本上所有的操作都要通过API Server进行通信，整个集群中的各个对象的状态信息也都要流到API Server，对于Masters节点而言，其产生的数据并不保存到Masters节点本地，而放在共享存储Etcd中，Etcd是一个键值存储数据库，其本身还拥有各种协调工作，整个集群所有的对象状态信息都存储在Etcd中，如果Etcd组件宕机，那么整个集群都会瘫痪，所以etcd必须作冗余

kubernetes集群总共由3个节点组成：Etcd <--> Masters <--> Nodes，各个节点间通过http或https通信，无论是对外还是对内，应该尽量使用https协议，那么各个节点内部、节点与节点之间、节点与客户端之间都使用https通信则需要多个CA证书

#### CNI（容器网络接口）

kubernetes集群中的3种网络模型，kubernetes自己不提供，而是依赖于第三方插件，也可以当做附件使用，无论是哪一个第三方服务商提供的网络解决方案至少都应该负责管理2种网络：Pod网络和集群网络，节点网络属于构建kubernetes之前就应该部署好的网络。kubernetes通过CNI插件体系来接入外部的网络服务解决方案，并且网络解决方案本身也可以通过Pod运行，然后为其他Pod提供网络解决方案，此Pod通常是一个特殊Pod，其虽然运行在集群上，但它需要共享所在节点的网络名称空间，这样此Pod就能以容器的方式操作节点的系统管理

第三方的网络解决方案提供的功能需要2个维度，第一，提供网络功能，为Service和Pod提供IP等，第二，提供网络策略，kubernetes之上的网络解决方案还要求能够提供network policy功能。在集群上托管了Pod后，各Pod之间通过Service进行通信，因为Service是Pod提供的一个固定的访问端点，但实际上两个Pod之间是可以直接通信的。在一个kubernetes集群中有多个项目或多租户的情况下，各不同Pod之间必须通过network policy功能进行网络隔离

常见的第三方CNI插件有：

- flannel：只支持网络配置
- calico：支持网络配置，支持网络策略
- canel：前面两者的折中解决方案

### kubernetes上的NameSpace（不是docker上的namespace）

kubernetes通过NameSpace将网络切割成多个空间，一类Pod只运行在一个空间中，但这个空间提供的不是真正的网络边界，只是管理边界（实现资源分组），便于对某个空间内的Pod实现批量操作，但不同空间的Pod之间依然是可以相互通信的，因为所有Pod仍处于同一个Pod网络中，而network policy则允许管理员自定义各个NameSpace之间能否互相访问、NameSpace内的各Pod之间能够互相通信，通过生成iptables规则限制Pod之间的通信

