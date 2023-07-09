# 客户端发现pod并与之通信
pod需要一种寻找其他pod的方法来使用其他pod提供的服务，不像在没有Kubernetes的世界，系统管理员要在用户端配置文件中明确指出服务的精确IP地址或者主机名来配置每个客户端应用，但同样的方法在Kubernetes中不适用
- pod是短暂的: 他们随时会启动或关闭，无论是给其他pod提供空间而从节点被移除，或者是减少了pod的数量，又或者是因为集群中存在节点异常
- Kubernetes在pod启动前会给已经调度到节点上的pod分配IP地址——因此客户端不能提前知道供应的pod IP地址
- 水平伸缩意味着多个pod可能会提供相同的服务——每个pod都有自己的IP地址，所有pod可以通过一个单一的IP地址进行访问
## 介绍服务
Kubernetes服务是一种为一组功能相同的pod提供单一不变接入点的资源
### 创建服务
#### 通过kubectl expose创建服务
创建服务最简单的方法是通过kubectl expose
#### 通过YAML描述文档来创建服务
创建名为kubia-svc.yaml文件
```yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: kubia
    spec:
        ports:
        -   port: 80 #该服务的端口
            targetPort: 8080 #服务将连接转发到的容器端口
        selector:
            app: kubia #具有app=kubia标签的pod都属于该服务
```
该yaml文件创建一个名为kubia的服务，它将在端口80接收请求并将连接路由到具有标签选择器app=kubia的pod的8080端口上。接下来使用kubectl cerate发布文档创建服务
#### 检测新的服务
使用一下命令列出所有服务资源
```
    kubectl get svc
```
#### 从内部集群测试服务
可以通过以下几种方法向服务发送请求：
- 创建一个pod，将请求发送到服务器的集群IP并记录响应，可以通过查看pod日志检查服务的响应
- 使用ssh远程登录到其中一个kubernetes节点上，然后使用curl命令
- 通用kubectl exec命令在一个已经存在的pod中执行curl命令
#### 在运行的容器中远程执行命令
可以使用kubectl exec命令远程地在一个已经存在的pod容器上执行任何命令，可以方便的了解pod内容、状态、环境。
```
    kubectl exec pod名 -- curl -s IP地址
```
注： 双横杠（--）代表着kubectl命令项的结束，在双横杠之后的内容是指在pod内部需要执行的命令。
#### 配置服务上的会话亲和性
如果希望特定客户端所产生的所有请求每次都指向同一个pod，可以设置服务sessionAffinity属性为ClientIP（默认None，随机指向pod中的一个）
```yaml
    apiVersion: v1
    kind: Service
    spec:
        sessionAffinity: ClientIP
```
#### 同一个服务暴露多个端口
在创建一个有多个端口的服务的时候，必须给每个端口指定名字
```yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: kubia
    spec:
        ports:
        -   name: http #端口名
            port: 80   #服务暴露端口
            targetPort: 8080 #服务转发pod接收端口
        -   name: https
            port: 443
            targetPort: 8443
        selector:   #标签选择器
            app: kubia

```
注：标签选器应用于整个服务，不能对每个端口做单独的配置。如果不同的pod有不同的端口映射关系，需要创建两个服务
#### 使用命名的端口
在服务spec中可以用数字指定端口，也可以给端口命名，通过名称来指定端口

在pod的定义中指定pod
```yaml
    kind: Pod
    spec:
        containers:
        -   name: http
            containerPort: 8080
        -   name: https
            containerPort: 8443
```

可以在服务spec中按名称引用这些端口
```yaml
    apiVersion: v1
    kind: Service
    spec:
        ports:
        -   name: http
            port: 80
            targetPort: http
        -   name: https
            port: 443
            targetPort: https
```
注：使用端口名称引用，可以屏蔽端口变更对引用处的影响
### 服务发现
通过创建服务，现在就可以通过一个单一稳定的IP地址访问到pod，在服务整个生命周期内这个地址保持不变，在服务后的pod可能删除重建，它们的IP地址可能改变，数量可能增减，但是始终可以通过服务的单一不变的IP地址访问到这些pod

#### 通过环境变量发现服务
在pod开始运行的时候，Kubernets会初始化一系列的环境变量指向现在存在的服务。如果你创建的服务早于你客户端pod的创建，pod上的进程可以根据环境变量获取服务IP地址和端口号

查看pod环境变量
```
    kubectl exec pod名 env
```
{serviceName}_SERVICE_HOST: 服务IP地址
{serviceName}_SERVICE_PORT: 服务端口号

#### 通过DNS发现服务
在每个命名空间都包含一个kube_dns服务，这个服务运行DNS服务，在集群中其他pod都被配置成使用其作为dns（Kubernetes通过修改每个容器 /etc/resolv.conf文件实现）。运行在pod上的进程DNS查询都会被Kubernetes自身的DNS服务器响应，该服务知道系统中运行的所有服务

注：pod是否使用内部DNS服务器是根据pod中spec的dnsPolicy属性来决定

每个服务从内部DNS服务器中获取一个DNS条目，客户端的pod在知道服务名情况下可以通过全限定域名（FQDN）来访问，而不是环境变量

#### 通过FQDN连接服务
可以通过一下方式访问其他服务：
{serviceName}.{命名空间}.{所有集群本地服务名称中使用的可配置集群域后缀}
注：客户端仍然必须知道服务的端口号。如果服务使用标准端口号（例如，HTTP的80端口或Postgres的5432端口），不需指定端口，如果并不是标准端口，客户端可以从环境变量中获取端口号

如果访问的服务在同一个命名空间下，可以省略命名空间及所有集群本地服务名称中使用的可配置集群域后缀，直接用服务名指代服务
#### 在pod容器中运行shell
登陆pod容器运行shell
```
    kubectl exec -it pod名 bash
```
注：/etc/resolv.conf存储DNS解析器配置方式

#### 无法ping通服务的原因
服务集群IP是一个虚拟IP，并且只有在与服务端口结合时才有意义

### 连接集群外部的服务

#### 介绍服务endpoit
服务并不是和pod直接相连，相反有一种资源介于两者之间——它就是Endpoint资源。Endpoint资源就是暴露一个服务的IP地址和端口列表，Endpoint资源和其他Kubernetes资源一样，可以使用kubectl info来获取他的基本信息
```
    kubectl get endpoints 服务名
```
尽管在spec服务中定义了pod选择器，但重定向传入连接时不会直接使用它。相反选择器用于构建IP和端口列表，然后存储在Endpoint资源中。当客户端连接到服务时，服务代理选择这些IP和端口对中的一个，并将传入连接重定向在该位置监听的服务器

#### 手动配置服务的endpoint
服务的endpoint与服务解耦后，可以分别配置和更新他们。如果创建了不包含pod选择器的服务，Kubernetes将不会创建Endpoint资源。这样就需要创建Endpoint资源来指定该服务的endpoint列表

要使用手工配置endpoint的方式创建服务，需要创建服务和Endpoint资源

#### 创建没有选择器的服务
为服务创建YAML文件
```yaml
    apiVerison: v1
    kind: Service
    metadata:
        name: external-service
    spec:
        ports:
        -   port: 80
```
#### 为没有选择器的服务创建Endpoint资源
Endpoint是一个单独的资源并不是服务的一个属性。由于创建的资源中不包含选择器，相关的Endpoint资源并没有自动创建，所以必须手动创建
```yaml
    apiVeriosn: v1
    kind: Endpoints
    metadata:
        name: external-service #Endpoint的名称必须和服务的名称相匹配
    subsets:
        -   addresses:
            -   ip: 11.11.11.11 #将服务连接重定向到endpoint的IP地址
            -   ip: 22.22.22.22
            ports:
            -   port: 80 #endpoint的目标端口
```
Endpoint对象需要与服务具有相同的名称,并包含该服务的目标IP地址和端口列表。服务和Endpoint资源都发布到服务器后，这样服务就可以像具有pod选择器那样的服务正常使用。在服务创建后创建的容器将包含服务的环境变量，并且与其IP：port对的所有连接都将在服务端点之间进行负载均衡
### 为外部服务创建别名

#### 创建ExternalName类型的服务
要创建一个具有别名的外部服务的服务时，要将创建服务资源的一个type字段设置为ExternalName
```yaml
    apiVersion: v1
    kind: Service
    metadata:
        name: external-service
    spec:
        type: ExternalName #代码的type被设置成ExternalName
        externalName: someapi.somecompany.com #实际服务的完全限定域名
        prots:
        -   port: 80
```
服务创建完成后，pod可以通过external-service.default.svc.cluster.local域名连接到外部服务，而不是使用服务的实际FQDN，这隐藏了实际的服务名及其使用该服务的pod的位置，允许修改服务定义，并且以后如果将其指向不同的服务，只需要简单修改externalName属性，或者将类型重新变回ClusterIP并为服务创建Endpoint

ExternalName服务仅在DNS级别实施-为服务创建简单的CNAME DNS记录。因此连接到服务的客户端将直接连接到外部服务，完全绕过服务代理，出于这个原因，这类服务器甚至不会获得集群IP
注：CNAME记录指向完全限定的域名而不是数字IP地址
## 将服务暴露给外部客户端
有几种方式可以在外部访问服务：
- 将服务的类型设置为NodePort——每个集群节点都会在节点上打开一个端口，对于NodePort服务，每个集群节点本身上打开一个端口，并将端口上接收到的流量重定向到基础服务。该服务仅在内部集群IP和端口上才可访问，但也可以通过所有节点上的专用端口访问
- 将服务的类型设置成LoadBalance,NodePort类型的一种扩展——这使得服务可以通过一个专用的负载均衡来访问，这是有Kubernetes中正在运行的云基础设施提供。负载均衡器将流量重定向到跨所有节点到节点端口。客户端通过负载均衡器的IP连接到服务
- 创建一个Ingress资源，这是一个完全不同的机制，通过一个IP地址公开多个服务——它运行在HTTP层上，因此可以提供比工作在第四层的服务更多功能

### 使用NodePort类型服务
创建NodePort服务，可以让Kubernetes在其所有节点上保留一个端口（所有节点都使用相同的端口号），并将传入的连接转发给作为服务部分的pod

这与常规服务类似（他们的实际类型是ClusterIP），但是不仅可以通过服内部集群IP访问NodePort服务，还可以通过节点的IP和预留节点端点访问NodePort服务

#### 创建NodePort类型服务
创建NodePart服务YAML
```yaml
    apiVersion: v1
    kind: Service
    metadate:
        name: kubia-nodeport
    spec:
        type: NodePort  #为NodePort设置服务类型
        ports:
        -   port: 80    #服务集群IP的端口号
            targetPort: 8080    #背后pod的目标端口号
            nodePort: 30123 #通过集群节点的30123端口可以访问该服务
        selector:
            app: kubia
```
将类型设置为NodePort并指定该服务应该绑定的所有集群节点的节点端口。指定端口不是强制的。如果忽略它，Kubernetes将选择一个随机端口
#### 查看NodePort类型的服务
查看NodePort服务的基础信息
```
    kubectl get svc 服务名
```
EXTERNAL-IP列为nodes，表明服务可通过任何集群节点的IP地址访问。PORT列显示集群IP的1内部端点和节点端口，可通过以下方式访问服务：
- 内部集群IP:内部端口号
- 节点IP:节点端口

#### 更改防火墙规则，让外部客户端访问我们的NodePort服务
设置防火墙，以允许外部连接到该端口上的节点
```
    gcloud compute firewall-rules create kubia-svc-rule --allow=tcp:30123
```
查看节点IP地址
```
    kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'
```
通过指定kubectl的JSONPath，使得其只输出需要的信息
- 浏览item属性中的所有元素
- 对于每个元素，输入status属性
- 过滤addresses属性的元素，仅包含那些具有将type属性设置为ExternalIP的元素
- 最后打印过滤元素的address属性

现在可以通过访问任何节点上的30123端口访问到pod

### 通过负载均衡器将服务暴露出
通过创建Load Badancer类型服务创建一个负载均衡器服务，提供自己独一无二的可公开访问IP，将所有连接重定向到服务。如果Kubernetes在不支持Load Badancer服务的环境运行，则不会调用负载均衡器，该服务将表现的像一个NodePort服务。Load Badancer服务是NodePort服务的扩展

#### 创建LoadBalance服务
创建负载均衡器服务YAML
```yaml
    apiVerion: v1
    kind: Service
    metadata:
        name: kubia-loadbalancer
    spec:
        type: loadBalancer
        ports:
        -   port: 80
            targetPort: 8080
        selector:
            app: kubia
```
如果没有指定特定的节点，kubernetes将会选择一个端口

#### 通过负载均衡器连接服务
创建服务后，云基础框架需要一段时间才能创建负载均衡器并将其IP地址写入服务对象，列为服务的外部IP地址
```
    kubectl get svc kubia-loadbalancer
```
EXTERNAL-IP列为负载均衡IP地址，可以通过该IP地址访问该服务。不必通过关闭防火墙来访问服务

#### 会话亲和性和Web浏览器
通过网络浏览器访问服务，发现请求都被同一个pod处理，通过kubectl explain可以再次检查服务的会话亲和性仍然设置为None，因为浏览器使用keep-alive连接，并通过单个连接发送所有请求，服务在连接级别工作，所以当首次打开与服务的连接时，会选择一个随机集群，然后将所有属于该连接的所有网络数据包全部发送到单个集群，即使会话亲和性设置为None，用户也始终使用相同pod

### 了解外部连接的特性

#### 了解并防止不必要的网络跳数
通过服务的spec部分设置externalTrafficPolicy字段设置，当外部客户端通过节点端口连接到服务时，选择当前服务节点上的pod，如果当前服务所在节点没有pod，则连接将挂起，以阻止额外跳数。所存在的缺点
- 如果接收连接的服务没有pod，连接将挂起
- 导致跨pod的负载均衡分布不均衡
```yaml
    spec:
        externalTrafficPolicy: Local
        ...
```

#### 客户端IP是不记录的
当集群内客户端连接到服务时，支持服务的pod可以获取客户端的IP地址，但通过节点端口连接时，由于对数据包执行了源网络地址转换，因此数据包的源IP将发生更改。local外部流量策略会影响客户端IP的保留，因为在接收连接的节点和托管目标pod的节点之间没有额外的跳跃

## 通过Ingress暴露服务

为什么需要Ingress：Ingress只需要一个公网IP就能为许多服务提供访问，当客户端向Ingress发送HTTP请求时，Ingress会根据请求的主机名和路径决定请求转发到的服务，Ingress在网络栈的应用层操作，并且能提供基于cookie的会话亲和性

Ingress控制器：Ingress控制器在集群中运行，Ingress资源才能正常工作，不同kubernetes环境使用不同的控制器实现

### 创建Ingress资源
创建Ingress资源YAML
```yaml
    apiVersion: extensions/v1beta1
    kind: Ingress
    metadata:
        name: kubia
    spec:
        rules:
        -   host: kubia.example.com //Ingress将域名kubia.example.com映射到你的服务
            http:
                paths:              //将所有请求发送到kubia-nodeport服务80端口
                -   path: /
                    backend:
                        serviceName: kubia-nodeport
                        servicePort: 80
```
### 通过Ingress访问服务
通过域名访问服务，需要确保域名解析为Ingress控制器的IP

#### 获取Ingress的IP地址
```
    kubectl get ingresses
```

#### 确保Ingress中配置的Host指向Ingress的IP地址
通过配置DNS服务器将域名解析为Ingress的IP地址

#### 了解Ingress的工作原理
客户端通过对域名执行DNS查找，DNS服务器返回Ingress控制器的IP，客户端向Ingress控制器发送HTTP请求，并设置Host头中指定域名，控制器从该头部确定客户端尝试访问哪个服务，通过与该服务关联的Endpoint对像查看pod IP，并将客户端的请求转发给其中一个pod

### 通过相同的Ingress暴露多个服务
Ingress的规范中rules和paths都是数组，因此他们可以包含多个条目

#### 将不同的服务映射到相同主机的不同路径
```yaml
    -   host: kubia.example.com
        http:
            paths:
            -   path: /kubia            #对kubia.example.com/kubia的请求将会转发至kubia服务
                backend:
                    serviceName: kubia
                    servicePort: 80
            -   path: /foo              #对kubia.example.com/bar的请求将转发至bar服务
                backend:
                    serviceName: bar
                    servicePort: 80
```

#### 将不同的服务映射到不同主机上
```yaml
    spec:
        rules:
        -   host: foo.example.com        #对foo.example.com的请求将会转发至foo服务
            http:
                paths:
                -   path: /
                    backend:
                        serviceName: foo
                        servicePort: 80
        -   host: bar.example.com       #对bar.example.com的请求将转发至bar服务
            http:
                paths:
                -   path: /
                    backend:
                        serviceName: bar
                        servicePort: 80
```
根据请求中的Host头，控制器收到的请求将被转发到foo服务或bar服务。DNS需要将foo.example.com和bar.example.com域名都指向Ingress控制器的IP地址

### 配置Ingress处理TLS传输

#### 为Ingress创建TLS认证
当客户端创建到Ingress控制器的TLS连接时，控制器将终止TLS连接，客户端和控制器之间的通信是加密的，控制器与后端pod之间的通信则不是，运行在pod上的应用程序不需要支持TLS。例如，如果pod运行web服务器，则它只能接收HTTP通信，并让Imgress控制器负责处理与TLS相关的所有内容，要使控制器能够这样做，需要将证书和私钥附加到Ingress，这两个必需资源存储在称为Secret的Kubernets资源中，然后在Ingress manifest中引用它

首先需要创建私钥和证书
```
    openssl genrsa -out tls.key 2048
    openssl req -new -x509 -key tls.key -out tls.cert -days 360 -subj /CN=kubia.example.com
```
存储                   