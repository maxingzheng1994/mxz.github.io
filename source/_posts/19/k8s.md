etcd：保存了整个集群的状态
Api Server：提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和
发现等机制
controller manager：负责维护集群的状态，比如故障检测、自动扩展、滚动更新等
scheduler：负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上
kubelet:负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管
理
Container runtime: 负责镜像管理以及Pod和容器的真正运行（CRI）
kube-proxy:负责为Service提供cluster内部的服务发现和负载均衡
![](https://raw.githubusercontent.com/mxz1994/note/master/20190625192151330_16832.png)
Static Pod  以Pod的方式在Master上创建Controller Manager ApiServer 和 Scheduler
Master 组件的 YAML 文件会被生成在 /etc/kubernetes/manifests 路径下

控制节点，即 Master 节点，由三个紧密协作的独立组件组合而成，它们分别是负责 API 服务的 kube-apiserver、负责调度的 kube-scheduler，以及负责容器编排的 kube-controller-manager。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Ectd 中

而计算节点上最核心的部分，则是一个叫作 kubelet 的组件。
kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数

而具体的容器运行时，比如 Docker 项目，则一般通过 OCI 这个容器运行时规范同底层的 Linux 操作系统进行交互，即：把 CRI 请求翻译成对 Linux 操作系统的调用（操作 Linux Namespace 和 Cgroups 等）。

此外，kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互。这个插件，是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件，也是基于 Kubernetes 项目进行机器学习训练、高性能作业支持等工作必须关注的功能。

而kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与 kubelet 进行交互的接口，分别是 CNI（Container Networking Interface）和 CSI（Container Storage Interface）。

Pod
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202050.png)


##### 创建一个 Master 节点
$ kubeadm init
###### 将一个 Node 节点加入到当前集群中
$ kubeadm join <Master 节点的 IP 和端口 >

kubeadm  init  生成work加入master主节点cmd
一般master禁止运行用户pod 是因为
依靠的是 Kubernetes 的 Taint/Toleration 机制。它的原理非常简单：一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有 Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。
通过 Taint/Toleration 调整 Master 执行 Pod 的策略
为节点打上“污点”（Taint）的命令是：kubectl taint nodes node1 foo=bar:NoSchedule
kubectl taint nodes --all node-role.kubernetes.io/master-  加横杠删除污点

部署容器存储插件 远程挂载存储
Kubernetes 存储插件项目：Rook。
```yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml

kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml
```

为什么会有Pod
而如果事先没有“组”的概念，像这样的运维关系就会非常难以处理。
我还是以前面的 rsyslogd 为例子。已知 rsyslogd 由三个进程组成：一个 imklog 模块，一个 imuxsock 模块，一个 rsyslogd 自己的 main 函数主进程。这三个进程一定要运行在同一台机器上，否则，它们之间基于 Socket 的通信和文件交换，都会出现问题。
现在，我要把 rsyslogd 这个应用给容器化，由于受限于容器的“单进程模型”，这三个模块必须被分别制作成三个不同的容器。而在这三个容器运行的时候，它们设置的内存配额都是 1 GB。
再次强调一下：容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？
Pod，其实是一组共享了某些资源的容器
具体的说：Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。
那这么来看的话，一个有 A、B 两个容器的 Pod，不就是等同于一个容器（容器 A）共享另外一个容器（容器 B）的网络和 Volume 的玩儿法么？
这好像通过 docker run --net --volumes-from 这样的命令就能实现嘛，比如：
但是，你有没有考虑过，如果真这样做的话，容器 B 就必须比容器 A 先启动，这样一个 Pod 里的多个容器就不是对等关系，而是拓扑关系了。
所以，在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。这样的组织关系，可以用下面这样一个示意图来表达：
![pause。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。](https://raw.githubusercontent.com/mxz1994/note/master/20190828202132.png)

例如 如果我们升级用war包下 tomcat启动
需要配置两个镜像，这样升级的时候才能互不影响
```yml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```
上面的配置中  initContainers 会比 下面的container 先执行
将 sample.war copy 到 /app 目录下 /app 与 /root/apache-tomcat-7.0.42-v2/webapps 挂载在同一地方   读取日志也可以
sidecar  
一个容器永远只能管理一个进程
Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序

NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段
HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容
```
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
```

```
 lifecycle:
      postStart: 容器启动时执行
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop: 容器关闭时执行
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```
Pod 生命周期：
1. Pending  
2. Running 
3. Succeeded
4. Failed
5. 5. Unknown 主从节点（Master 和 Kubelet）间的通信出现了问题

Kubernetes 支持的 Projected Volume
Secret；ConfigMap；Downward API；ServiceAccountToken。

**sercret 用来保存密码**
```
echo -n 'admin' | base64

apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```
```
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```
**ConfigMap**
```
# 从.properties 文件创建 ConfigMap
$ kubectl create configmap ui-config --from-file=example/ui.properties
```
kubectl get -o yaml   可将pod api 按yaml方式列出

**Downward API **
它的作用是：让 Pod 里的容器能够直接获取到这个 Pod API 对象本身的信息。没多大用处

##### 容器健康检查和恢复机制
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
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

livenessProbe:
     httpGet:
       path: /healthz
       port: 8080
       httpHeaders:
       - name: X-Custom-Header
         value: Awesome
       initialDelaySeconds: 3
       periodSeconds: 3

```
在启动后创建healthy文件 30s后删除
探针在容器启动后5s每5s 请求一次这个文件，存在返回0
如果Pod健康检查失败后会进行重启，只会在旧的Node上进行重启
restartPolicy : Always   OnFailure  Never

##### 简化Pod.yaml
** PodPreset 对象**  仅作用于selector 所定义的、带有“role: frontend”标签的 Pod 对象
```
**apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}**
```

Pod模型
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202241.png)
ReplicaSet  管理pod的升级
Deployment 操纵着ReplicaSet
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202305.png)
``` 滚动升级
kubectl scale deployment nginx-deployment --replicas=4
```
--record  创建容器加上记录下你每次操作所执行的命令，以方便后面查看。
RollingUpdateStrategy 升级策略
一个应用版本对应一个replicaSet
kubectl rollout undo 命令
```
查看历史信息
$ kubectl rollout history deployment/nginx-deployment --revision=2
回滚到指定版本
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```
spec.revisionHistoryLimit 设置历史版本个数

[金丝雀发布蓝绿发布](https://github.com/ContainerSolutions/k8s-deployment-strategies/tree/master/canary)
金丝雀部署：优先发布一台或少量机器升级，等验证无误后再更新其他机器。优点是用户影响范围小，不足之处是要额外控制如何做自动更新。
蓝绿部署：2组机器，蓝代表当前的V1版本，绿代表已经升级完成的V2版本。通过LB将流量全部导入V2完成升级部署。优点是切换快速，缺点是影响全部用户。
本文学习的滚动更新，我觉得就是一个自动化新的金丝雀发布.

##### StatefulSet  直接管理Pod
statefulSet 其实就是一种特殊的 Deployment，而其独特之处在于，它的每个 Pod 都被编号了。而且，这个编号会体现在 Pod 的名字和 hostname 等标识信息上，这不仅代表了 Pod 的创建顺序，也是 Pod 的重要网络标识
StatefulSet 的设计其实非常容易理解。它把真实世界里的应用状态，抽象为了两种情况：
拓扑状态。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。
存储状态。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

###### 拓扑状态
StatefulSet 这个控制器的主要作用之一，就是使用 Pod 模板创建 Pod 的时候，对它们进行编号，并且按照编号顺序逐一完成创建工作。而当 StatefulSet 的“控制循环”发现 Pod 的“实际状态”与“期望状态”不一致，需要新建或者删除 Pod 进行“调谐”的时候，它会严格按照这些 Pod 编号的顺序，逐一完成这些操作。
所以，StatefulSet 其实可以认为是对 Deployment 的改良（只是多了serviceName=nginx 字段。）

###### 存储状态
第一步：定义一个 PVC，声明想要的 Volume 的属性：
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```
第二步：在应用的 Pod 中，声明使用这个 PVC：
```
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: pv-storage
  volumes:
    - name: pv-storage
      persistentVolumeClaim:
        claimName: pv-claim
```
第三步 运维人员维护的 PV（Persistent Volume）对象 pv Yaml

第二步的配置可以放在StatefulSet 中
```
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1G
```


只要修改 StatefulSet 的 Pod 模板，就会自动触发“滚动更新”
```
 kubectl patch statefulset mysql --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"mysql:5.7.23"}]'
statefulset.apps/mysql patched

```
这样，StatefulSet Controller 就会按照与 Pod 编号相反的顺序，从最后一个 Pod 开始，逐一更新这个 StatefulSet 管理的每个 Pod
金丝雀发布 滚动更新一部分容器
这个字段，正是 StatefulSet 的 spec.updateStrategy.rollingUpdate 的 partition 字段。
比如，现在我将前面这个 StatefulSet 的 partition 字段设置为 2：
```
$ kubectl patch statefulset mysql -p '{"spec":{"updateStrategy":{"type":"RollingUpdate","rollingUpdate":{"partition":2}}}}'
statefulset.apps/mysql patched
```
只有序号大于等于2的会被升级

##### DaemonSet

我们的 DaemonSet Controller 会在创建 Pod 的时候，自动在这个 Pod 的 API 对象里，加上这样一个 nodeAffinity 定义。其中，需要绑定的节点名字，正是当前正在遍历的这个 Node。

DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

##### Job 与 CronJob
Job Controller 控制的对象，直接就是 Pod。
Job Controller 在控制循环中进行的调谐（Reconcile）操作，是根据实际在 Running 状态 Pod 的数目、已经成功退出的 Pod 的数目，以及 parallelism、completions 参数的值共同计算出在这个周期里，应该创建或者删除的 Pod 数目，然后调用 Kubernetes API 来执行这个操作。

#### lstio
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202329.png)
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202401.png)
Istio 是一个功能十分丰富的 Service Mesh，它包括如下功能：

流量管理：这是 Istio 的最基本的功能。
策略控制：通过 Mixer 组件和各种适配器来实现，实现访问控制系统、遥测捕获、配额管理和计费等。
可观测性：通过 Mixer 来实现。
安全认证：Citadel 组件做密钥和证书管理。

Envoy 容器就能够通过配置 Pod 里的 iptables 规则，把整个 Pod 的进出流量接管下来。
Istio 的控制层（Control Plane）里的 Pilot 组件，就能够通过调用每个 Envoy 容器的 API，对这个 Envoy 代理进行配置，从而实现微服务治理。
进行简单的灰度发布

创建文件中没有lstio 容器 怎么做到无感的 
Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 Dynamic Admission Control
Istio 要做的，就是编写一个用来为 Pod“自动注入”Envoy 容器的 Initializer。
ConfigMap 的方式保存在 Kubernetes 当中

##### kubectl api

在 Kubernetes 项目中，一个 API 对象在 Etcd 里的完整资源路径，是由：Group（API 组）、Version（API 版本）和 Resource（API 资源类型）三个部分组成的。

![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202428.png)

APIServer 就可以继续创建这个 CronJob 对象
![](https://raw.githubusercontent.com/mxz1994/note/master/20190828202509.png)
首先，当我们发起了创建 CronJob 的 POST 请求之后，我们编写的 YAML 的信息就被提交给了 APIServer。
而 APIServer 的第一个功能，就是过滤这个请求，并完成一些前置性的工作，比如授权、超时处理、审计等。
然后，请求会进入 MUX 和 Routes 流程。如果你编写过 Web Server 的话就会知道，MUX 和 Routes 是 APIServer 完成 URL 和 Handler 绑定的场所。而 APIServer 的 Handler 要做的事情，就是按照我刚刚介绍的匹配过程，找到对应的 CronJob 类型定义。
接着，APIServer 最重要的职责就来了：根据这个 CronJob 类型定义，使用用户提交的 YAML 文件里的字段，创建一个 CronJob 对象。
而在这个过程中，APIServer 会进行一个 Convert 工作，即：把用户提交的 YAML 文件，转换成一个叫作 Super Version 的对象，它正是该 API 资源类型所有版本的字段全集。这样用户提交的不同版本的 YAML 文件，就都可以用这个 Super Version 对象来进行处理了。
接下来，APIServer 会先后进行 Admission() 和 Validation() 操作。比如，我在上一篇文章中提到的 Admission Controller 和 Initializer，就都属于 Admission 的内容。
而 Validation，则负责验证这个对象里的各个字段是否合法。这个被验证过的 API 对象，都保存在了 APIServer 里一个叫作 Registry 的数据结构中。也就是说，只要一个 API 对象的定义能在 Registry 里查到，它就是一个有效的 Kubernetes API 对象。
最后，APIServer 会把验证过的 API 对象转换成用户最初提交的版本，进行序列化操作，并调用 Etcd 的 API 把它保存起来。

CRD 的全称是 Custom Resource Definition。顾名思义，它指的就是，允许用户在 Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：自定义 API 资源。


