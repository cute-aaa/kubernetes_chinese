## 配置pod和容器

### 为容器和pod分配内存资源

这里展示了怎样为容器指定内存*请求*和内存*限制*，容器会保证拥有所请求的内存，但不允许超过限制的内存。

#### 准备工作

你需要一个kubernetes集群，并且必须配置kubectl命令行工具使其能够与你的集群通信。如果你没有集群，可以使用[minikube](https://kubernetes.io/docs/setup/minikube)创建一个，或者你也可以使用这些kubernetes练习场：

-   [Katacoda](https://www.katacoda.com/courses/kubernetes/playground)
-   [Play with Kubernetes](http://labs.play-with-k8s.com/)

输入 `kubectl version` 查看版本。

集群中的每个节点都至少有300M的内存（memory）。

这一节的一些步骤需要在集群中开启 [metrics-server](https://github.com/kubernetes-incubator/metrics-server)服务，如果你已经有了在运行的metrics-server，可以跳过这些步骤：

如果你用的是Minikube，使用以下名来来开启metrics-server：

```shell
minikube addons enable metrics-server
```

查看metrics-server或者另一个资源调度API提供器（metrics.k8s.io）是否在运行，使用：

```shell
kubectl get apiservices
```

如果资源调度API可用，将会包含以下输出：

```shell
NAME      
v1beta1.metrics.k8s.io
```

>   这里提到了一个[metrics-server](https://github.com/kubernetes-incubator/metrics-server)，说是一些步骤需要这个。还说在 `kube-up.sh` 中会自动获取。我试了一下，不安装metrics的话，一会使用kubectl top是会报一个Error from server (NotFound): the server could not find the requested resource (get services http:heapster:)的错误。
>
>   `kube-up.sh` 在[这里](<https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh>)，是k8s源码下的一个文件。使用apt安装的是没有这个的。所以自己搞一下metrics-server吧。
>
>   把[这几个文件](<https://github.com/kubernetes-incubator/metrics-server/tree/master/deploy/1.8%2B>)下载到一个文件夹里，这里是 metrics/，然后执行 
>
>   ```sh
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/aggregated-metrics-reader.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/auth-delegator.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/auth-reader.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/metrics-apiservice.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/metrics-server-deployment.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/metrics-server-service.yaml
>   
>   wget https://github.com/kubernetes-incubator/metrics-server/raw/master/deploy/1.8%2B/resource-reader.yaml
>   ```
>
>   ```bash
>   kubectl create -f metrics/
>   ```
>
>   查看一下
>
>   ```bash
>   kubectl get apiservices | grep metrics
>   ```



#### 创建一个命名空间

创建一个[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces)，以便在练习中创建的资源与集群的其他部分隔离。

```shell
kubectl create namespace mem-example
```



#### 指定内存请求和内存限制

要指定容器的内存请求，请在容器的资源清单（manifest）中指定 `resources:requests` 字段，指定内存限制，指定 `resources:limits` 字段

这里，你将会创建一个拥有一个容器的pod，这个容器的内存请求为100MiB，内存限制为200MiB，下面是这个pod的配置文件：

[`pods/resource/memory-request-limit.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/memory-request-limit.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

配置文件中的 `args` 部分提供了容器启动时的参数， `--vm-bytes`  `150M` 参数告诉容器尝试去申请150MiB的内存

创建一个pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit.yaml --namespace=mem-example
```

验证容器是否在运行:

```shell
kubectl get pod memory-demo --namespace=mem-example
```

查看pod的详细信息:

```shell
kubectl get pod memory-demo --output=yaml --namespace=mem-example
```

输出显示pod中的每个容器的内存请求为100MB，内存限制为200MB

```yaml
...
resources:
  limits:
    memory: 200Mi
  requests:
    memory: 100Mi
...
```

运行 `kubectl top`  来获取pod的

```shell
kubectl top pod memory-demo --namespace=mem-example
```

输出显示这个pod大概用了162,900,000B内存，约为150MB。比pod请求的100MB高，但还在200MB的限制之内。

```
NAME                        CPU(cores)   MEMORY(bytes)
memory-demo                 <something>  162856960
```

删除pod：

```shell
kubectl delete pod memory-demo --namespace=mem-example
```



#### 超出内存限制的容器

我们知道在节点有可用内存时，容器可以超出他的内存请求的大小，但不允许容器内存超过其内存限制。如果容器分配的内存超过了他的内存限制，那么这个容器将会成为终止的候选对象。如果容器继续超出内存限制，这个容器将会被终止。如果被终止的容器可以进行重启，那么kubelet会像解决其他类型的运行故障一样重启它。

这里，你将会创建一个尝试超出内存限制的pod。下面是pod的配置文件，显示每个容器有50MB的内存请求和100MB的内存限制。

[`pods/resource/memory-request-limit-2.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/memory-request-limit-2.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-2
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-2-ctr
    image: polinux/stress
    resources:
      requests:
        memory: "50Mi"
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "250M", "--vm-hang", "1"]
```

可以看到，在配置文件中的 `args` 部分，容器尝试申请250MB内存，超出了100MB的内存限制。

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-2.yaml --namespace=mem-example
```

查看pod信息：

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

此时，容器可能被杀死了，也可能还在运行。重复上面的命令直到容器被杀死：

```shell
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          24s
```

获取更详细的容器状态：

```shell
kubectl get pod memory-demo-2 --output=yaml --namespace=mem-example
```

输出结果显示容器因为超出内存限制而被杀死（out of memory, OOM)：

```shell
lastState:
   terminated:
     containerID: docker://65183c1877aaec2e8427bc95609cc52677a454b56fcb24340dbd22917c23b10f
     exitCode: 137
     finishedAt: 2017-06-20T20:52:19Z
     reason: OOMKilled
     startedAt: null
```

这个练习中的容器可以被重启，所以kubelet把它重启了。重复以下命令，可以看到容器一直重复地被杀和重启。

```shell
kubectl get pod memory-demo-2 --namespace=mem-example
```

输出显示容器被杀死，然后重启，然后被杀死，然后重启，循环往复：

```
kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS      RESTARTS   AGE
memory-demo-2   0/1       OOMKilled   1          37s
kubectl get pod memory-demo-2 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-2   1/1       Running   2          40s
```

查看pod的详细历史记录

```
kubectl describe pod memory-demo-2 --namespace=mem-example
```

输出显示容器一次次启动，又一次次失败。

```
... Normal  Created   Created container with id 66a3a20aa7980e61be4922780bf9d24d1a1d8b7395c09861225b0eba1b1f8511
... Warning BackOff   Back-off restarting failed container
```

查看集群中节点的详细信息：

```
kubectl describe nodes
```

输出显示了容器因为超出内存而被杀死的记录

```
Warning OOMKilling Memory cgroup out of memory: Kill process 4481 (stress) score 1994 or sacrifice child
```

删除pod：

```shell
kubectl delete pod memory-demo-2 --namespace=mem-example
```



#### 指定超出节点内存的内存请求

内存的请求和限制与容器相关联，但看作成pod有内存请求和限制更好。pod的内存请求是pod中所有容器的内存请求总和，同样，pod的内存限制也是pod中所有容器的内存限制总和。

pod的调度基于请求。只有当节点具有足够的可用内存来满足Pod的内存请求时，pod才会被调度运行。

在这个练习中，你将创建一个Pod，它的内存请求很大，超出了你的集群中任何节点的容量。下面是Pod的配置文件，该Pod中的每个容器都会请求1000 GiB内存，这可能超过集群中任何节点的容量。

[`pods/resource/memory-request-limit-3.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/memory-request-limit-3.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo-3
  namespace: mem-example
spec:
  containers:
  - name: memory-demo-3-ctr
    image: polinux/stress
    resources:
      limits:
        memory: "1000Gi"
      requests:
        memory: "1000Gi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]
```

创建这个pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/memory-request-limit-3.yaml --namespace=mem-example
```

查看pod状态：

```shell
kubectl get pod memory-demo-3 --namespace=mem-example
```

输出显示pod的状态是 PENDING（挂起），意思是，pod没有在任何节点上调度运行，并且将无限期地保持PENDING状态。

```sh
kubectl get pod memory-demo-3 --namespace=mem-example
NAME            READY     STATUS    RESTARTS   AGE
memory-demo-3   0/1       Pending   0          25s
```

查看pod的详细信息，包括事件：

```shell
kubectl describe pod memory-demo-3 --namespace=mem-example
```

输出显示由于节点的内存不够，所以容器不能被调度运行。

```shell
Events:
  ...  Reason            Message
       ------            -------
  ...  FailedScheduling  No nodes are available that match all of the following predicates:: Insufficient memory (3).
```



#### 内存单元

内存资源以字节为单位。您可以将内存表示为一个整数或带以下后缀之一的定点整数: E、P、T、G、M、K、Ei、Pi、Ti、Gi、Mi、Ki。例如，下面表示的值大致相同:

```shell
128974848, 129e6, 129M , 123Mi
```

删除pod：

```shell
kubectl delete pod memory-demo-3 --namespace=mem-example
```



#### 如果不指定内存限制

如果你没有指定容器的内存限制，适用于以下情况：

-   容器使用的内存量没有上限，容器运行时可以使用节点上的所有可用内存，但可能会被OOM杀死。此外，在OOM杀容器时，没有内存限制的容器被杀死的可能性更大。
-   容器在具有默认内存限制的命名空间中运行，容器将自动分配默认限制的内存。集群管理员可以使用LimitRange指定内存限制的默认值。

#### 内存请求和限制的动机

为集群中的容器配置内存请求和内存限制，可以有效地利用节点可用的内存资源。保持pod有较少的的内存请求，那么这个pod就有更多机会被调度。通过设置大于内存请求的内存限制，你可以完成两件事：

-   pod可以在有可用内存的地方执行突发事件
-   突发事件中使用的内存被限制在一个合理范围内。



#### 清理

删除命令空间，同时会删除里面的所有pod。

```shell
kubectl delete namespace mem-example
```



#### 下一步

#### 对于 app开发人员

-   [为容器和pod分配CPU资源](https://kubernetes.io/docs/tasks/configure-pod-container/assign-cpu-resource/)
-   [配置pod的服务质量集群

#### 集群管理员

-   [为命名空间指定默认的内存请求和内存限制](https://kubernetes.io/docs/tasks/administer-cluster/memory-default-namespace/)
-   [为命名空间指定默认的CPU请求和限制](https://kubernetes.io/docs/tasks/administer-cluster/cpu-default-namespace/)
-   [为命名空间指定最小和最大的内存约束](https://kubernetes.io/docs/tasks/administer-cluster/memory-constraint-namespace/)
-   [为命名空间指定最小和最大的CPU约束](https://kubernetes.io/docs/tasks/administer-cluster/cpu-constraint-namespace/)
-   [为命名空间指定CPU配额](https://kubernetes.io/docs/tasks/administer-cluster/quota-memory-cpu-namespace/)
-   [为命名空间指定pod配额](https://kubernetes.io/docs/tasks/administer-cluster/quota-pod-namespace/)
-   [为API对象指定配额](https://kubernetes.io/docs/tasks/administer-cluster/quota-api-object/)





这一节显示了怎样为容器分配CPU请求和CPU限制。容器使用的CPU不能超过设置的CPU限制。当CPU有空闲的CPU时间时，容器将保证拥有他所请求的CPU资源

#### 准备工作

同上一节的准备工作部分。



#### 创建命名空间

```shell
kubectl create namespace cpu-example
```



#### 指定CPU请求和CPU限制

在容器的资源清单中，使用 `resources:requests` 字段来指定容器的CPU请求，使用 `resources:limits` 来指定容器的CPU限制。

这里，你将会创建一个包含一个容器的pod，这个容器将请求0.5个CPU，限制为1个CPU。下面是pod的配置文件：

[`pods/resource/cpu-request-limit.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/cpu-request-limit.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr
    image: vish/stress
    resources:
      limits:
        cpu: "1"
      requests:
        cpu: "0.5"
    args:
    - -cpus
    - "2"
```

`args` 字段提供了容器启动时的参数，参数 `-cpus "2"` 指定容器尝试使用2个CPU（超出限制了）

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit.yaml --namespace=cpu-example
```

验证pod是否在运行：

```shell
kubectl get pod cpu-demo --namespace=cpu-example
```

查看pod的详细信息：

```shell
kubectl get pod cpu-demo --output=yaml --namespace=cpu-example
```

输出显示pod中的容器有500m（m表示千分之一，500m即0.5）个CPU的请求和1个CPU的限制

```yaml
resources:
  limits:
    cpu: "1"
  requests:
    cpu: 500m
```

使用 `kubectl top` 来获取pod 的运行状态

```shell
kubectl top pod cpu-demo --namespace=cpu-example
```

输出显示这个pod用了974m的CPU，只比pod配置中限制的1个CPU小一点点。

```sh
NAME                        CPU(cores)   MEMORY(bytes)
cpu-demo                    974m         <something>
```

回想以下 `-cpu "2"` 的设置，你让容器尝试使用2个CPU，但容器只允许使用1个CPU。容器的CPU使用正在被节流，因为容器一直在尝试使用更多超出限制的CPU资源。

>   **Note:** 另一个对CPU使用低于1.0的可能的解释是节点可能没有足够的可用的CPU资源。回想一下，这个练习的先决条件要求每个节点至少有一个CPU。如果您的容器运行在只有一个CPU的节点上，那么无论为容器指定的CPU限制是多少，容器都不能使用超过一个CPU。



#### CPU单位

CPU资源以CPU单位为度量。在Kubernetes中，一个CPU相当于:

-   1 AWS vCPU

    An **AWS vCPU** is a single hyperthread of a two-thread Intel Xeon core for M5, M4, C5, C4, R4, and R4 instances. A simple way to think about this is that an **AWS vCPU**is equal to half a physical core.

-   1 GCP Core

     Google Cloud Platform Core

-   1 Azure vCore

    没有搜到，Azure是微软的，vcore在Hadoop里倒是有个定义：'**vcores**' – short for virtual **cores。**还有个概念是关于CPU工作电压的。

-   1 Hyperthread on a bare-metal Intel processor with Hyperthreading

    直译：使用超线程的裸金属Intel处理器上的超线程

    超线程（Hyper-Threading），英特尔研发的，在一个实体CPU中，提供两个逻辑线程。

允许使用小数，请求0.5个CPU的容器请求的CPU是请求1个CPU的容器的一半。可以使用后缀m来代表千分之一。比如100m CPU，100 milliCPU和0.1CPU是相同的，精度不能超过1m。

CPU请求总是绝对量，而不是相对量。0.1不管在单核，双核，还是48核都是一样的。（即都是0.1个CPU）

删除pod：

```shell
kubectl delete pod cpu-demo --namespace=cpu-example
```



#### 指定超出节点CPU的CPU请求

与内存请求一样，CPU请求和限制与容器相关联，但看作Pod的CPU请求和限制更好。Pod的CPU请求是Pod中所有容器的CPU请求之和，Pod的CPU限制是Pod中所有容器的CPU限制之和。

Pod调度基于请求。只有当节点具有足够的CPU资源来满足Pod的CPU请求时，才在节点上调度运行Pod。

在本练习中，你将创建一个Pod，它的CPU请求非常大，超出了集群中任何节点的CPU数量。下面是Pod的配置文件。容器请求100个CPU，这应该超过了集群中任何节点的容量。

[`pods/resource/cpu-request-limit-2.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/cpu-request-limit-2.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cpu-demo-2
  namespace: cpu-example
spec:
  containers:
  - name: cpu-demo-ctr-2
    image: vish/stress
    resources:
      limits:
        cpu: "100"
      requests:
        cpu: "100"
    args:
    - -cpus
    - "2"
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/cpu-request-limit-2.yaml --namespace=cpu-example
```

查看pod状态：

```shell
kubectl get pod cpu-demo-2 --namespace=cpu-example
```

输出显示pod处于 PENDING（挂起）状态，这表示pod没有在任何节点上调度运行，并且将无限期地处于挂起状态：

```shell
kubectl get pod cpu-demo-2 --namespace=cpu-example
NAME         READY     STATUS    RESTARTS   AGE
cpu-demo-2   0/1       Pending   0          7m
```

查看pod的详细信息，包括事件：

```shell
kubectl describe pod cpu-demo-2 --namespace=cpu-example
```

输出显示容器不能运行，因为节点上的CPU不够。

```
Events:
  Reason                        Message
  ------                        -------
  FailedScheduling      No nodes are available that match all of the following predicates:: Insufficient cpu (3).
```

删除pod：

```shell
kubectl delete pod cpu-demo-2 --namespace=cpu-example
```



#### 如果不指定CPU限制

以下两种情况适用于不指定CPU限制：

-   容器使用的CPU没有上限，容器运行时可以使用节点上所有的可用CPU资源
-   容器在具有默认CPU限制的命名空间中运行，容器会自动分配默认限制。集群管理员可以使用LimitRange指定CPU限制的默认值。



#### CPU请求和限制的动机

与内存一样，CPU使用少的调度机会多，CPU限制对突发事件更友好。



#### 清理

删除命名空间：

```shell
kubectl delete namespace cpu-example
```





### 为Windows的pod和容器设置GMSA

应该很少有人用Windows服务器，这里不再赘述。

GMSA即Group Managed Service Accounts，有兴趣的话看这里<https://kubernetes.io/docs/tasks/configure-pod-container/configure-gmsa/>





### 设置pod的服务质量

这一节展示了如何配置Pods，以便为它们分配特定的服务质量(Quality of Service，QoS)类。Kubernetes使用QoS类来决定如何调度和删除pod。



#### 准备工作

需要一个kubernetes集群



#### QoS类

当Kubernetes创建Pod时，会将这些QoS类中的一个分配给Pod:

-   Guaranteed（有保证）
-   Burstable（可超频）
-   BestEffort（最佳效果）



#### 创建命名空间

老规矩，先创命名空间

```shell
kubectl create namespace qos-example
```



#### 创建pod并给他分配QoS类中的Guaranteed

对于使用Guaranteed的pod：

-   Pod中的每个容器都必须有内存限制和内存请求，而且它们必须相同。

-   Pod中的每个容器都必须有CPU限制和CPU请求，而且它们必须相同。

下面是pod的配置文件，容器有内存请求和内存限制，但都是200MB，也有CPU请求和限制，也都是700m。

[`pods/qos/qos-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/qos/qos-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod.yaml --namespace=qos-example
```

查看pod的详细信息：

```shell
kubectl get pod qos-demo --namespace=qos-example --output=yaml
```

输出显示kubernetes给了这个pod一个QoS中的Guaranteed类，输出也验证了pod中容器和CPU的请求、限制是否相匹配。

```yaml
spec:
  containers:
    ...
    resources:
      limits:
        cpu: 700m
        memory: 200Mi
      requests:
        cpu: 700m
        memory: 200Mi
...
  qosClass: Guaranteed
```

>   **Note:** 如果容器指定了内存限制，但没有指定内存请求，kubernetes会自动分配一个与内存限制匹配的内存请求。CPU也一样。

删除pod：

```shell
kubectl delete pod qos-demo --namespace=qos-example
```



#### 创建pod并给他分配QoS类中的Burstable

以下情况会被定义为QoS中的Burstable类：

-   Pod不满足QoS类的Guaranteed标准。

-   Pod中至少有一个容器具有内存或CPU请求。

配置文件：

[`pods/qos/qos-pod-2.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/qos/qos-pod-2.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
      requests:
        memory: "100Mi"
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-2.yaml --namespace=qos-example
```

查看详细信息：

```shell
kubectl get pod qos-demo-2 --namespace=qos-example --output=yaml
```

输出显示kubernetes给了这个pod一个QoS中的Burstable类：

```yaml
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: qos-demo-2-ctr
    resources:
      limits:
        memory: 200Mi
      requests:
        memory: 100Mi
...
  qosClass: Burstable
```

删除pod：

```shell
kubectl delete pod qos-demo-2 --namespace=qos-example
```





#### 创建pod并给他分配QoS类中的BestEffort

要创建一个BestEffort类的pod，pod中的容器必须不能有任何内存或CPU的请求、限制。

下面是有一个容器的pod的配置文件，这个容器没有内存和CPU的请求、限制。

Here is the configuration file for a Pod that has one Container. The Container has no memory or CPU limits or requests:

[`pods/qos/qos-pod-3.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/qos/qos-pod-3.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-3.yaml --namespace=qos-example
```

查看pod的详细信息

```shell
kubectl get pod qos-demo-3 --namespace=qos-example --output=yaml
```

输出显示kubernetes给了这个pod一个BestEffort类

```yaml
spec:
  containers:
    ...
    resources: {}
  ...
  qosClass: BestEffort
```

删除pod：

```shell
kubectl delete pod qos-demo-3 --namespace=qos-example
```





#### 创建一个有俩容器的pod

下面是有两个容器的pod的配置文件，一个容器制定了200M的内存请求，另一个没有指定内存请求和限制。

[`pods/qos/qos-pod-4.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/qos/qos-pod-4.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-4
  namespace: qos-example
spec:
  containers:

  - name: qos-demo-4-ctr-1
    image: nginx
    resources:
      requests:
        memory: "200Mi"

  - name: qos-demo-4-ctr-2
    image: redis
```

注意，这个pod满足QoS中Burstable类的标准，其中一个容器有内存请求。（就是一个容器只有内存请求，另一个什么也没有，所以不满足Guaranteed的标准，也不满足BestEffort的标准）

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/qos/qos-pod-4.yaml --namespace=qos-example
```

查看pod的详细信息：

```shell
kubectl get pod qos-demo-4 --namespace=qos-example --output=yaml
```

输出显示kubernetes给了这个pod一个Burstable类（因为不满足其他两个）：

```yaml
spec:
  containers:
    ...
    name: qos-demo-4-ctr-1
    resources:
      requests:
        memory: 200Mi
    ...
    name: qos-demo-4-ctr-2
    resources: {}
    ...
  qosClass: Burstable
```

Delete your Pod:

```shell
kubectl delete pod qos-demo-4 --namespace=qos-example
```



#### 清理

删除命名空间：

```shell
kubectl delete namespace qos-example
```



### 分配扩展资源给容器

这一节展示怎么给容器分配扩展资源。



#### 准备工作

需要一个集群。

在做这一节的练习之前，先把 [Advertise Extended Resources for a Node](https://kubernetes.io/docs/tasks/administer-cluster/extended-resource-node/)做了，他会在你的一个节点上发布4个dongle资源。（管理集群.md -> 为节点发布资源扩展）。



#### 为pod分配扩展资源

要请求扩展资源，需要在容器的清单（manifest）中包含 `resources:requests` 字段。扩展资源的域名可以是 `*.kubernetes.io/` 外的任何域名。比如一个有效的扩展资源名的形式是example.com/foo，其中example.com替换为你组织的域名，foo是一个资源名。

下面是一个有一个容器的pod的配置文件：

[`pods/resource/extended-resource-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/extended-resource-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo
spec:
  containers:
  - name: extended-resource-demo-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 3
      limits:
        example.com/dongle: 3
```

在配置文件中可以看到，这个容器请求了3个dongle。

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod.yaml
```

验证pod是否在运行：

```shell
kubectl get pod extended-resource-demo
```

查看pod信息：

```shell
kubectl describe pod extended-resource-demo
```

输出显示了dongle的请求：

```yaml
Limits:
  example.com/dongle: 3
Requests:
  example.com/dongle: 3
```



#### 尝试创建第二个pod

下面是一个包含一个容器的pod的配置文件，这个容器请求2个dongle。

[`pods/resource/extended-resource-pod-2.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/resource/extended-resource-pod-2.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo-2
spec:
  containers:
  - name: extended-resource-demo-2-ctr
    image: nginx
    resources:
      requests:
        example.com/dongle: 2
      limits:
        example.com/dongle: 2
```

Kubernetes 满足不了2个dongle的请求，因为只有4个dongle，第一个pod用了三个。

尝试创建容器：

```shell
kubectl apply -f https://k8s.io/examples/pods/resource/extended-resource-pod-2.yaml
```

查看容器信息：

```shell
kubectl describe pod extended-resource-demo-2
```

输出显示pod不能被调度，因为没有节点拥有2个可用的dongle。

```shell
Conditions:
  Type    Status
  PodScheduled  False
...
Events:
  ...
  ... Warning   FailedScheduling  pod (extended-resource-demo-2) failed to fit in any node
fit failure summary on nodes : Insufficient example.com/dongle (1)
```

查看pod状态

```shell
kubectl get pod extended-resource-demo-2
```

输出显示pod已经被创建了，但没有被调度运行，处于挂起状态：

```yaml
NAME                       READY     STATUS    RESTARTS   AGE
extended-resource-demo-2   0/1       Pending   0          6m
```



#### 清理

删除本次练习中创建的pod：

```shell
kubectl delete pod extended-resource-demo
kubectl delete pod extended-resource-demo-2
```





### 将pod配置为使用卷来存储

这一节展示如何将pod配置为使用卷来存储。

容器的文件系统与容器共存亡，所以当容器被终止和重启时，文件系统改动，之前的文件丢失。想要使用更持久的独立于容器的存储，可以使用 [卷](https://kubernetes.io/docs/concepts/storage/volumes/)。这对于有状态的应用程序，如键值存储(如Redis)和数据库来说非常重要。



#### 准备工作

需要个集群。



#### 为pod配置一个卷

在这次练习中，你会创建一个运行一个容器的pod，这个Pod有一个[emptyDir](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir) 类型的卷，该卷在Pod的生命周期内持续存在，即使容器终止或重新启动也不会消失。

[`pods/storage/redis.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/redis.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```

1.  创建pod：

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/storage/redis.yaml
    ```

2.  验证pod是否在运行，并观察pod的变化：

    ```shell
    kubectl get pod redis --watch
    ```

    输出大致是：

    ```shell
    NAME      READY     STATUS    RESTARTS   AGE
    redis     1/1       Running   0          13s
    ```

3.  在另一个终端，获取正在运行的容器的shell：（与容器进行交互）

    ```shell
    kubectl exec -it redis -- /bin/bash
    ```

4.  在shell中，进入 `data/redis` 文件夹，然后创建一个文件：

    ```shell
    root@redis:/data# cd /data/redis/
    root@redis:/data/redis# echo Hello > test-file
    ```

5.  在shell中，列出正在运行的程序：

    ```shell
    root@redis:/data/redis# apt-get update
    root@redis:/data/redis# apt-get install procps
    root@redis:/data/redis# ps aux
    ```

    输出大致是：

    ```shell
    USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
    redis        1  0.1  0.1  33308  3828 ?        Ssl  00:46   0:00 redis-server *:6379
    root        12  0.0  0.0  20228  3020 ?        Ss   00:47   0:00 /bin/bash
    root        15  0.0  0.0  17500  2072 ?        R+   00:48   0:00 ps aux
    ```

6.  在shell中，杀死redis进程：

    ```shell
    root@redis:/data/redis# kill <pid>
    ```

    `<pid>` 是redis的进程ID (PID).

7.  在原来的的终端，查看redis的pod的变化。大致是：

    ```shell
    NAME      READY     STATUS     RESTARTS   AGE
    redis     1/1       Running    0          13s
    redis     0/1       Completed  0         6m
    redis     1/1       Running    1         6m
    ```

此时，容器已经被终止并重启了，这是因为redis pod 的[重启策略(restartPolicy)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podspec-v1-core) 是 `Always`。

1.  获取重启之后的shell：

    ```shell
    kubectl exec -it redis -- /bin/bash
    ```

2.  进入 `/data/redis` 目录，看看 `test-file` 是否还在那里：

    ```shell
    root@redis:/data/redis# cd /data/redis/
    root@redis:/data/redis# ls
    test-file
    ```

3.  删除本地练习中创建的pod：

    ```shell
    kubectl delete pod redis
    ```



#### 下一步

-   查看[卷](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#volume-v1-core).
-   查看[Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#pod-v1-core).
-   除了emptyDir提供的本地磁盘存储之外，Kubernetes还支持许多不同的 网络附加 存储（network-attached storage）解决方案，包括GCE上的PD和EC2上的EBS，它们是存储关键数据的首选，并将处理诸如在节点上安装和卸载设备等细节。有关详细信息，请参阅 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)。





### 将Pod配置为使用持久卷进行存储

这一节展示了如何将Pod配置为使用持久卷进行存储。下面是整个过程的总结：

1.  集群管理员创建一个由物理存储支持的持久卷，且此卷不与任何pod关联。
2.  集群用户创建一个持久卷声明（PersistentVolumeClaim），它会自动绑定到合适的持久卷。
3.  用户创建pod并使用持久卷声明来存储。



#### 准备工作

-   只有一个节点的集群。
-   熟悉[持久卷](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)的概念



#### 在节点上创建一个index.html文件

打开节点上的shell，怎么打开取决于你怎么设置的集群。比如，如果你用的是Minikube，可以使用 `minikube ssh` 来打开节点的shell。

在shell中，创建 `/mnt/data` 目录

```sh
sudo mkdir /mnt/data
```

在 `/mnt/data` 目录中创建 `index.html`

```sh
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
```



#### 创建一个持久卷

在本次练习中，你将创建一个 *hostPath* （主机路径？）持久卷，Kubernetes支持在单节点集群上开发和测试hostPath。hostPath持久卷使用节点上的文件或目录模拟网络附加存储。

在生产集群中，您不会使用hostPath。相反，集群管理员将提供网络资源，如谷歌计算引擎持久磁盘、NFS共享或Amazon弹性块存储卷（Google Compute Engine persistent disk，NFS share，Amazon Elastic Block Store volume）。集群管理员还可以使用[存储类](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#storageclass-v1-storage)设置[动态供应](https://kubernetes.io/blog/2016/10/dynamic-provisioning-and-storage-in-kubernetes).。

下面是hostPath持久卷的配置文件：

[`pods/storage/pv-volume.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/pv-volume.yaml)

```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```

配置文件指定卷位于集群节点上的 `/mnt/data` 目录。还指定了10 G的大小和 `ReadWriteOnce` 的访问权限，这意味着卷可以由单个节点以读写方式挂载。将存储类名定义为 `manual` ，`manual` 意味着将用于持久卷声明绑定持久卷的请求。

创建持久卷：

```sh
kubectl apply -f https://k8s.io/examples/pods/storage/pv-volume.yaml
```

查看持久卷信息：

```sh
kubectl get pv task-pv-volume
```

输出显示持久卷的状态是 `Available` ，这意味着还没有持久卷声明绑定他。

```sh
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO           Retain          Available             manual                   4s
```





#### 创建一个持久卷声明

下一步，创建一个持久卷声明，pod使用持久卷声明来请求物理存储。在这个练习中，你将会创建一个持久卷声明来请求一个3G的至少可以为一个节点提供读写访问的卷。

下面是持久卷声明的配置文件：

[`pods/storage/pv-claim.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/pv-claim.yaml)

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```

创建一个持久卷声明：

```sh
kubectl apply -f https://k8s.io/examples/pods/storage/pv-claim.yaml
```

创建持久卷声明之后，kubernetes控制面板会去找满足声明请求的持久卷，如果找到了具有相同存储类的持久卷，就会把声明绑定到持久卷上。

再次查看持久卷：

```shell
kubectl get pv task-pv-volume
```

现在持久卷的状态变成了 `Bound`.

```
NAME             CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                   STORAGECLASS   REASON    AGE
task-pv-volume   10Gi       RWO           Retain          Bound     default/task-pv-claim   manual                   2m
```

查看持久卷声明：

```shell
kubectl get pvc task-pv-claim
```

输出显示持久卷声明已经绑定到了持久卷 `task-pv-volume` 上。

```shell
NAME            STATUS    VOLUME           CAPACITY   ACCESSMODES   STORAGECLASS   AGE
task-pv-claim   Bound     task-pv-volume   10Gi       RWO           manual         30s
```



#### 创建pod

下一步，创建一个使用持久卷声明作为卷的pod。

下面是这个pod的配置文件：

[`pods/storage/pv-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/pv-pod.yaml)

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```

注意，pod的配置文件指定了一个持久卷声明，而不是指定了持久卷。在pod看来，这个声明就是他的卷。

创建这个pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/storage/pv-pod.yaml
```

验证pod中的容器是否在运行：

```shell
kubectl get pod task-pv-pod
```

获取pod中容器的shell：

```shell
kubectl exec -it task-pv-pod -- /bin/bash
```

在shell中，验证nginx正在使用hostPath卷中的 `index.html` 来提供服务：

```shell
root@task-pv-pod:/# apt-get update
root@task-pv-pod:/# apt-get install curl
root@task-pv-pod:/# curl localhost
```

输出显示了你向hostPath卷中的 `index.html` 写入的文本

```shell
Hello from Kubernetes storage
```



#### 访问控制

配置了group ID（GID）的存储只允许具有相同GID的pod写入。GID不匹配或不设置GID会导致拒绝访问的错误。为了减少与用户协调的需要，管理员可以使用GID对持久性卷进行注释。然后这个GID就会自动添加到所有使用这个持久卷的pod上。

像下面这样使用 `pv.beta.kubernetes.io/gid` 注释：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
  annotations:
    pv.beta.kubernetes.io/gid: "1234"
```

当Pod使用具有GID注释的持久卷时，注释的GID将应用到Pod中的所有容器，这与在Pod的security上下文中指定GID的方式相同。每个GID，无论是源自PersistentVolume注释还是Pod的指定，都应用于每个容器中运行的第一个进程。

>   **Note:** 当pod使用了一个持久卷时，与持久卷关联的GID并不在pod身上。



#### 下一步

-   学习 [持久卷](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).
-   阅读 [持久存储设计文档](https://git.k8s.io/community/contributors/design-proposals/storage/persistent-storage.md)

##### 参考

-   [持久卷](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#persistentvolume-v1-core)
-   [持久卷规范](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#persistentvolumespec-v1-core)
-   [持久卷声明](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#persistentvolumeclaim-v1-core)
-   [持久卷声明规范](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#persistentvolumeclaimspec-v1-core)





### 将pod配置为使用投影卷存储

这一节展示如何使用  [`投影卷`](https://kubernetes.io/docs/concepts/storage/volumes/#projected) 将几个已经存在的源卷挂载到同一个目录中。目前可以投影 `secret` `configMap` `downwardAPI` `serviceAccountToken` 卷。

>   **Note:** `serviceAccountToken` 不是卷类型。



#### 准备工作

一个集群。

#### 为pod配置投影卷

本次练习中，你将创建从本地文件中创建用户名和密码的机密文件文件，然后创建一个pod，使用投影卷来把这些机密文件挂载到同一个共享目录。

下面是配置文件：

[`pods/storage/projected.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/storage/projected.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume
spec:
  containers:
  - name: test-projected-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

1.  创建机密文件：

    ```shell
    # Create files containing the username and password:
    echo -n "admin" > ./username.txt
    echo -n "1f2d1e2e67df" > ./password.txt
    
    # Package these files into secrets:
    kubectl create secret generic user --from-file=./username.txt
    kubectl create secret generic pass --from-file=./password.txt
    ```

2.  创建pod：

    ```shell
    kubectl apply -f https://k8s.io/examples/pods/storage/projected.yaml
    ```

3.  验证pod中的容器是否在运行，然后查看pod的变化：

    ```shell
    kubectl get --watch pod test-projected-volume
    ```

    输出大致是：

    ```
    NAME                    READY     STATUS    RESTARTS   AGE
    test-projected-volume   1/1       Running   0          14s
    ```

4.  在另一个终端，获取运行中的容器的shell：

    ```shell
    kubectl exec -it test-projected-volume -- /bin/sh
    ```

5.  在shell中，验证 `projected-volume` 目录是否包含映射过来的资源：

    ```shell
    ls /projected-volume/
    ```



#### 清理

删除pod和机密文件

```shell
kubectl delete pod test-projected-volume
kubectl delete secret user pass
```



#### 下一步

-   学习关于 [`投影卷`](https://kubernetes.io/docs/concepts/storage/volumes/#projected) 的知识.
-   查看 [一体化卷(all-in-one volume)](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/node/all-in-one-volume.md) 设计文档





### 为pod或容器配置安全上下文（Security Context）

安全上下文定义了pod或容器的特权和访问控制设置。安全上下文设置包括：

-   自由访问控制：访问对象的权限是基于 [user ID (UID) and group ID (GID)](https://wiki.archlinux.org/index.php/users_and_groups) 的，比如文件。

-   [SELinux](https://en.wikipedia.org/wiki/Security-Enhanced_Linux): 对象被分配了安全标签。

-   以特权或无特权模式运行

-   [Linux 的功能](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/): 给用户一些特权，但不是root用户的所有特权。

-   [AppArmor](https://kubernetes.io/docs/tutorials/clusters/apparmor/): 使用程序配置文件来限制个人程序。

-   [Seccomp](https://en.wikipedia.org/wiki/Seccomp): 过滤掉进程的系统调用

-   允许特权升级（AllowPrivilegeEscalation）: 控制一个进程是否可以获取比父进程更多的特权，这个bool直接控制是否在容器中设置[no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt) 标签。以下情况中允许特权升级为true：

    1) run as Privileged 

    2) has `CAP_SYS_ADMIN`.

更多关于Linux安全机制的信息请看[Linux内核安全特性概述](https://www.linux.com/learn/overview-linux-kernel-security-features)



#### 准备工作

一个集群。



#### 为pod设置安全上下文

要指明pod的安全设置，在pod配置中包含 `securityContext` 字段，这个字段是一个 [pod安全上下文](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podsecuritycontext-v1-core) 对象，这个为pod指定的设置将应用于pod中的所有容器。

下面是一个有`emptyDir` 卷 和 `securityContext` 的pod的配置文件：

[`pods/security/security-context.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/security/security-context.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
    securityContext:
      allowPrivilegeEscalation: false
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-2.yaml
```

验证pod中的容器是否在运行：

```shell
kubectl get pod security-context-demo-2
```

获取运行中容器的shell：

```shell
kubectl exec -it security-context-demo-2 -- sh
```

在shell中列出正在运行的进程：

```shell
ps aux
```

输出显示进程在以用户2000的身份运行。这是在 `runAsUser` 中为容器指定的值，覆盖了为pod指定的值1000.

```shell
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
2000         1  0.0  0.0   4336   764 ?        Ss   20:36   0:00 /bin/sh -c node server.js
2000         8  0.1  0.5 772124 22604 ?        Sl   20:36   0:00 node server.js
...
```

退出shell：

```shell
exit
```



#### 设置容器的功能

有了[Linux权限设定](http://man7.org/linux/man-pages/man7/capabilities.7.html)，你可以授予进程特定的权限，而不是授予root用户的所有权限。要为容器添加或删除Linux权限，请在容器配置清单的 `securityContext` 部分包含 `capabilities` 字段。

首先，看看如果不包含 `capabilities` 字段会发生什么。下面是没有添加或删除容器权限的配置文件：

[`pods/security/security-context-3.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/security/security-context-3.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-3
spec:
  containers:
  - name: sec-ctx-3
    image: gcr.io/google-samples/node-hello:1.0
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-3.yaml
```

验证pod中的容器是否运行：

```shell
kubectl get pod security-context-demo-3
```

获取运行中容器的shell：

```shell
kubectl exec -it security-context-demo-3 -- sh
```

在shell中查看正在运行的进程：

```shell
ps aux
```

输出显示了容器的进程ID（PID）

```shell
USER  PID %CPU %MEM    VSZ   RSS TTY   STAT START   TIME COMMAND
root    1  0.0  0.0   4336   796 ?     Ss   18:17   0:00 /bin/sh -c node server.js
root    5  0.1  0.5 772124 22700 ?     Sl   18:17   0:00 node server.js
```

在shell中查看PID为1的进程的状态：

```shell
cd /proc/1
cat status
```

输出显示了进程的权限位图

```
...
CapPrm:	00000000a80425fb
CapEff:	00000000a80425fb
...
```

记下权限位图，然后退出shell：

```shell
exit
```

接下来，运行一个与前面相同的容器，但这次的容器具有额外的权限集。

下面是一个运行一个容器的pod的配置文件，配置中添加了 `CAP_NET_ADMIN` 和 `CAP_SYS_TIME` 权限：

[`pods/security/security-context-4.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/security/security-context-4.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-4
spec:
  containers:
  - name: sec-ctx-4
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
```

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/security/security-context-4.yaml
```

获取运行中容器的shell：

```shell
kubectl exec -it security-context-demo-4 -- sh
```

在shell中，查看PID为1的进程的权限：

```shell
cd /proc/1
cat status
```

输出显示了这个进程的权限位图：

```shell
...
CapPrm:	00000000aa0435fb
CapEff:	00000000aa0435fb
...
```

比较两个位图：

```
00000000a80425fb
00000000aa0435fb
```

在第一个容器的权限位图中，第12位和第25位啥也没有，在第二个容器中，第12位和第25位被设置了。第12位是 `CAP_NET_ADMIN`，第25位是 `CAP_SYS_TIME`。查看 [capability.h](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h) 更多关于权限常量的定义。

>   **Note:** Linux权限常量的格式是 `CAP_XXX`. 但当你在容器配置清单中列出权限时，必须省略掉 `CAP_` 部分。比如想要添加 `CAP_SYS_TIME` 权限，就在清单中添加 `SYS_TIME` 。



#### 为容器分配SELinux标签

要给容器分配SELinux标签，请在pod或容器的配置清单中的 `securityContext` 部分添加 `seLinuxOptions` 字段。`seLinuxOptions` 字段是一个 [SELinuxOptions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#selinuxoptions-v1-core) 对象. 下面是一个应用了SELinux等级的例子：

```yaml
...
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

>   **Note:** 要分配SELinux标签，必须在主机操作系统上加载SELinux安全模块



#### 讨论

安全上下文不仅可以应用到pod中的容器，也适用于pod中的卷。指定应用到卷的 `fsGroup` 和 `seLinuxOptions` 。

-   `fsGroup`: 支持所有权管理的卷被更改为由 `fsGroup`中指定的GID所有和写入 。详见 [所有权管理设计文档](https://git.k8s.io/community/contributors/design-proposals/storage/volume-ownership-management.md) 。
-   `seLinuxOptions`: 支持SELinux标记的卷将被重新标记为可由 `seLinuxOptions` 下指定了的标签来访问。通常你需要设置 `level` 部分，这将为卷和pod中的所有容器设置 [Multi-Category Security (MCS)](https://selinuxproject.org/page/NB_MLS) 标签。（即为pod设置一个标签，然后这个卷就只能被这个标签访问了）

>   **Warning:** 当你为pod指定了MCS标签之后，所有拥有同样标签的pod都可以访问这个卷。如果需要对pod进行隔离，则必须为每个pod都指定不同的标签。



#### 下一步

-   [pod安全上下文](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podsecuritycontext-v1-core)
-   [安全上下文](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#securitycontext-v1-core)
-   [使用最新的安全增强来调优docker](https://opensource.com/business/15/3/docker-security-tuning)
-   [安全上下文设计文档](https://git.k8s.io/community/contributors/design-proposals/auth/security_context.md)
-   [所有者管理 设计文档](https://git.k8s.io/community/contributors/design-proposals/storage/volume-ownership-management.md)
-   [pod安全策略](https://kubernetes.io/docs/concepts/policy/pod-security-policy/)
-   [允许特权升级 设计文档](https://git.k8s.io/community/contributors/design-proposals/auth/no-new-privs.md)





### 为pod配置服务账户

服务账户（service account）为pod中的进程标明了身份。

*这是服务账户的介绍*。另请参阅 [服务账户的集群管理指南](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)

>   **Note:** 这篇文档描述了在kubernetes项目中的集群怎样设置为推荐设置。不过如果集群管理员自定义了集群中的行为，那这篇文档可能就不适用了。

当你访问集群的时候（比如通过 `kubectl` ），apiserver会将你认证为一个特定的用户帐户(在管理员没有定制的情况下默认是admin)。

容器中的进程也可以与apiserver联系。当他们这样做时，会被认证为一个特殊的服务账户（比如 `default` ）。



#### 准备工作

一个集群。



#### 使用默认的服务账户来访问APIserver

如果创建pod是没有指定服务账户，会在同一命名空间下自动分配一个 `default` 服务账户。查看创建的pod的json或yaml（比如使用 `kubectl get pods/<podname> -o yaml` ），你就会看到一个自动设置的`spec.serviceAccountName` 字段。

像 [访问集群](https://kubernetes.io/docs/user-guide/accessing-the-cluster/#accessing-the-api-from-a-pod) （访问集群中的应用.md -> 访问集群）Accessing Clusters中说的那样，你可以使用pod中自动挂载的服务账户证书来访问API。服务帐户的API权限取决于使用的[授权插件和策略](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules)。

在1.6+版本中，可以通过在服务帐户上设置`automountServiceAccountToken: false`来选择不自动挂载服务帐户的API证书:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
automountServiceAccountToken: false
...
```

在1.6+版本中，还可以选择不自动加载特定pod的API证书:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: build-robot
  automountServiceAccountToken: false
  ...
```

如果两者都指定了 `automountServiceAccountToken` ，pod规范将倾向于服务帐户。



#### 使用多个服务账户

每个命名空间都有一个叫作 `default` 的默认服务账户，可以使用以下命令来列出命名空间中的服务账户：

```shell
kubectl get serviceAccounts
```

输出大致是：

```
NAME      SECRETS    AGE
default   1          1d
```

可以像这样创建额外的服务账户对象：

```shell
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: build-robot
EOF
```

如果你获得了服务帐户对象的完整转储，像下面这样：

```shell
kubectl get serviceaccounts/build-robot -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-06-16T00:12:59Z
  name: build-robot
  namespace: default
  resourceVersion: "272500"
  selfLink: /api/v1/namespaces/default/serviceaccounts/build-robot
  uid: 721ab723-13bc-11e5-aec2-42010af0021e
secrets:
- name: build-robot-token-bvbk5
```

那么你就会看到一个自动创建的令牌，它已经被服务帐户引用了。

还可以使用授权插件来[设置服务帐户的权限](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions).

要使用非默认的服务帐户，只需将pod的`spec.serviceAccountName`字段设置为你希望使用的服务帐户名。

创建pod时服务账户必须存在，否则将被拒绝。

您不能更新已创建的pod的服务帐户。

可以像这样清理服务帐户：

```shell
kubectl delete serviceaccount/build-robot
```



#### 手动创建服务账户API令牌

假设我们有一个上面提到的名为 “build-robot” 的服务账户，然后我们手动创建一个新的 Secret。

```shell
kubectl create -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: build-robot-secret
  annotations:
    kubernetes.io/service-account.name: build-robot
type: kubernetes.io/service-account-token
EOF
secret/build-robot-secret created
```

现在，您可以确认新构建的 Secret 中填充了 “build-robot” 服务帐户的 API 令牌。

令牌控制器将清理不存在的服务帐户的所有令牌。

```shell
kubectl describe secrets/build-robot-secret
Name:           build-robot-secret
Namespace:      default
Labels:         <none>
Annotations:    kubernetes.io/service-account.name=build-robot
                kubernetes.io/service-account.uid=da68f9c6-9d26-11e7-b84e-002dc52800da

Type:   kubernetes.io/service-account-token

Data
====
ca.crt:         1338 bytes
namespace:      7 bytes
token:          ...
```

>   Note:
>
>   这里省略了 `token` 的内容。

#### 为服务账户添加 ImagePullSecrets

首先，创建一个 ImagePullSecrets，可以参考[这里](https://v1-14.docs.kubernetes.io/docs/concepts/containers/images/#specifying-imagepullsecrets-on-a-pod) 的描述。 然后，确认创建是否成功。例如：

```shell
kubectl get secrets myregistrykey
NAME             TYPE                              DATA    AGE
myregistrykey    kubernetes.io/.dockerconfigjson   1       1d
```

接着修改命名空间的默认服务帐户，以将该 Secret 用作 imagePullSecret。

```shell
kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "myregistrykey"}]}'
```

需要手动编辑的交互式版本：

```shell
kubectl get serviceaccounts default -o yaml > ./sa.yaml

cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  resourceVersion: "243024"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge

vi sa.yaml
[editor session not shown]
[delete line with key "resourceVersion"]
[add lines with "imagePullSecrets:"]

cat sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: 2015-08-07T22:02:39Z
  name: default
  namespace: default
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 052fb0f4-3d50-11e5-b066-42010af0d7b6
secrets:
- name: default-token-uudge
imagePullSecrets:
- name: myregistrykey

kubectl replace serviceaccount default -f ./sa.yaml
serviceaccounts/default
```

现在，在当前命名空间中创建的每个新 Pod 的 spec 中都会添加下面的内容：

```yaml
spec:
  imagePullSecrets:
  - name: myregistrykey
```

#### 服务帐户令牌卷投影

**FEATURE STATE:** `Kubernetes v1.12` [beta](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/configure-service-account/#)

>   Note:
>
>   ServiceAccountTokenVolumeProjection 在 1.12 版本中是 **beta** 阶段，可以通过向 API 服务器传递以下所有参数来启用它：
>
>   -   `--service-account-issuer`
>   -   `--service-account-signing-key-file`
>   -   `--service-account-api-audiences`

kubelet 还可以将服务帐户令牌投影到 Pod 中。 您可以指定令牌的所需属性，例如受众和有效持续时间。 这些属性在默认服务帐户令牌上无法配置。 当删除 Pod 或 ServiceAccount 时，服务帐户令牌也将对 API 无效。

使用名为 [ServiceAccountToken](https://v1-14.docs.kubernetes.io/docs/concepts/storage/volumes/#projected) 的 ProjectedVolume 类型在 PodSpec 上配置此功能。 要向 Pod 提供具有 “vault” 观众以及两个小时有效期的令牌，可以在 PodSpec 中配置以下内容：

[`pods/pod-projected-svc-token.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-projected-svc-token.yaml)

```yaml
kind: Pod
apiVersion: v1
spec:
  containers:
  - image: nginx
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: vault-token
  volumes:
  - name: vault-token
    projected:
      sources:
      - serviceAccountToken:
          path: vault-token
          expirationSeconds: 7200
          audience: vault
```

创建：

```shell
kubectl create -f https://k8s.io/examples/pods/pod-projected-svc-token.yaml
```

Kubelet 将代表 Pod 来请求和存储令牌，使令牌在可配置的文件路径上对 Pod 可用，并在令牌接近到期时刷新令牌。 如果令牌存活时间大于其总 TTL 的 80% 或者大于 24 小时，Kubelet 则会主动旋转令牌。

应用程序负责在令牌旋转时重新加载令牌。 对于大多数情况，定期重新加载（例如，每 5 分钟一次）就足够了。





### 从私有仓库拉取镜像

本文介绍如何使用 Secret 从私有的 Docker 镜像仓库或代码仓库拉取镜像来创建 Pod。

#### 准备开始

-   一个集群。

-   您需要 [Docker ID](https://docs.docker.com/docker-id/) 和密码来进行本练习。

#### 登录 Docker 镜像仓库

在个人电脑上，要想拉取私有镜像必须在镜像仓库上进行身份验证。

```shell
docker login
```

当提示时，输入 Docker 用户名和密码。

登录过程会创建或更新保存有授权令牌的 `config.json` 文件。

查看 `config.json` 文件：

```shell
cat ~/.docker/config.json
```

输出结果包含类似于以下内容的部分：

```json
{
    "auths": {
        "https://index.docker.io/v1/": {
            "auth": "c3R...zE2"
        }
    }
}
```

>   Note:
>
>   如果使用 Docker 凭证仓库，则不会看到 `auth` 条目，看到的将是以仓库名称作为值的 `credsStore` 条目。

#### 在集群中创建保存授权令牌的 Secret

Kubernetes 集群使用 `docker-registry` 类型的 Secret 来通过容器仓库的身份验证，进而提取私有映像。

创建 Secret，命名为 `regcred`：

```shell
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

在这里：

-   `<your-registry-server>` 是你的私有 Docker 仓库全限定域名（FQDN）。(参考 <https://index.docker.io/v1/> 中关于 DockerHub 的部分)
-   `<your-name>` 是你的 Docker 用户名。
-   `<your-pword>` 是你的 Docker 密码。
-   `<your-email>` 是你的 Docker 邮箱。

这样您就成功地将集群中的 Docker 凭据设置为名为 `regcred` 的 Secret。

#### 检查 Secret `regcred`

要了解你创建的 `regcred` Secret 的内容，可以用 YAML 格式进行查看：

```shell
kubectl get secret regcred --output=yaml
```

输出和下面类似：

```yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJodHRwczovL2luZGV4L ... J0QUl6RTIifX0=
kind: Secret
metadata:
  ...
  name: regcred
  ...
type: kubernetes.io/dockerconfigjson
```

`.dockerconfigjson` 字段的值是 Docker 凭据的 base64 表示。

要了解 `dockerconfigjson` 字段中的内容，请将 Secret 数据转换为可读格式：

```shell
kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode
```

输出和下面类似：

```json
{"auths":{"yourprivateregistry.com":{"username":"janedoe","password":"xxxxxxxxxxx","email":"jdoe@example.com","auth":"c3R...zE2"}}}
```

要了解 `auth` 字段中的内容，请将 base64 编码过的数据转换为可读格式：

```shell
echo "c3R...zE2" | base64 --decode
```

输出结果中，用户名和密码用 `:` 链接，类似下面这样：

```none
janedoe:xxxxxxxxxxx
```

注意，Secret 数据包含与本地 `~/.docker/config.json` 文件类似的授权令牌。

这样您就已经成功地将 Docker 凭据设置为集群中的名为 `regcred` 的 Secret。

#### 创建一个使用您的 Secret 的 Pod

下面是一个 Pod 配置文件，它需要访问 `regcred` 中的 Docker 凭据：

[`pods/private-reg-pod.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/pods/private-reg-pod.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-reg
spec:
  containers:
  - name: private-reg-container
    image: <your-private-image>
  imagePullSecrets:
  - name: regcred
```

下载上述文件：

```shell
wget -O my-private-reg-pod.yaml https://k8s.io/examples/pods/private-reg-pod.yaml
```

在`my-private-reg-pod.yaml` 文件中，使用私有仓库的镜像路径替换 `<your-private-image>`，例如：

```none
janedoe/jdoe-private:v1
```

要从私有仓库拉取镜像，Kubernetes 需要凭证。 配置文件中的 `imagePullSecrets` 字段表明 Kubernetes 应该通过名为 `regcred` 的 Secret 获取凭证。

创建使用了你的 Secret 的 Pod，并检查它是否正常运行：

```shell
kubectl create -f my-private-reg-pod.yaml
kubectl get pod private-reg
```

#### 接下来

-   进一步了解 [Secrets](https://v1-14.docs.kubernetes.io/docs/concepts/configuration/secret/)。
-   进一步了解 [使用私有仓库](https://v1-14.docs.kubernetes.io/docs/concepts/containers/images/#using-a-private-registry)。
-   参考 [kubectl create secret docker-registry](https://v1-14.docs.kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#-em-secret-docker-registry-em-)。
-   参考 [Secret](https://v1-14.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#secret-v1-core)。
-   参考 [PodSpec](https://v1-14.docs.kubernetes.io/docs/reference/generated/kubernetes-api/v1.14/#podspec-v1-core) 中的 `imagePullSecrets` 字段 。





### 配置活性检测和就绪检测（Liveness and Readiness Probes）

这一节展示如何为容器配置活性检测和就绪检测。

[kubelet](https://kubernetes.io/docs/admin/kubelet/) 使用活性探针（liveness probes）来知道何时重启容器。例如，活动探针可以捕获死锁，即应用程序正在运行，但不能取得进展（make progress）。在这种状态下重新启动容器有助于使应用程序更可用，尽管存在bug。

kubelet使用就绪探针来知道容器何时准备好开始接受流量。当所有容器都准备好时，就认为Pod已经准备好了。这个信号的一个用途是控制哪些pod用作服务的后端。当Pod没有准备好时，它将从服务负载平衡器中移除。

#### 准备工作

一个集群。

#### Define a liveness command

许多长时间运行的应用程序最终会变为中断状态，除非重新启动，否则无法恢复。Kubernetes提供活性探针来检测和纠正这种情况。

在本次练习中，你会创建一个pod，里面运行了一个基于 `k8s.gcr.io/busybox` 镜像的容器。下面是pod的配置文件：

[`pods/probe/exec-liveness.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/probe/exec-liveness.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/busybox
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
```

在配置文件中，可以看到这个pod里有单个容器， `periodSeconds` 字段指定kubelet应该每5秒执行一次活性检测。`initialDelaySeconds`  字段告诉kubelet在执行第一次检测之前应该等待5秒。为了执行检测，kubelet在容器中执行 `cat /tmp/healthy` 命令。如果命令执行成功，会返回0，kubelet就认为容器是活动的和健康的。如果命令返回一个非零值，kubelet将杀死容器并重新启动它。

（配置文件中指定）当容器启动时会执行下面的命令：

```shell
/bin/sh -c "touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600"
```

（配置文件中指定）容器生命中的前30秒，会有一个 `/tmp/healthy` 文件。所以在前30秒内， `cat /tmp/healthy` 命令返回成功代码，30秒后， `cat /tmp/healthy` 返回失败代码（文件被删除了）。

创建pod：

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/exec-liveness.yaml
```

30秒内查看pod事件：

```shell
kubectl describe pod liveness-exec
```

输出表明至今没有失败的活性检测：

```shell
FirstSeen    LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
24s       24s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
23s       23s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
```

35秒之后，再次查看pod事件：

```shell
kubectl describe pod liveness-exec
```

在输出的底部，有一些消息表明活性检测失败，容器已被杀死并重新创建。

```shell
FirstSeen LastSeen    Count   From            SubobjectPath           Type        Reason      Message
--------- --------    -----   ----            -------------           --------    ------      -------
37s       37s     1   {default-scheduler }                    Normal      Scheduled   Successfully assigned liveness-exec to worker0
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulling     pulling image "k8s.gcr.io/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Pulled      Successfully pulled image "k8s.gcr.io/busybox"
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Created     Created container with docker id 86849c15382e; Security:[seccomp=unconfined]
36s       36s     1   {kubelet worker0}   spec.containers{liveness}   Normal      Started     Started container with docker id 86849c15382e
2s        2s      1   {kubelet worker0}   spec.containers{liveness}   Warning     Unhealthy   Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory
```

再等30秒，再等30秒，确认容器是否已经重启：

```shell
kubectl get pod liveness-exec
```

输出显示 `RESTARTS` 已经增加了：

```shell
NAME            READY     STATUS    RESTARTS   AGE
liveness-exec   1/1       Running   1          1m
```

#### 定义一个活性HTTP请求

另一种活性检测使用HTTP GET请求。下面是一个pod，运行了一个基于 `k8s.gcr.io/liveness` 的镜像。

[`pods/probe/http-liveness.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/probe/http-liveness.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: k8s.gcr.io/liveness
    args:
    - /server
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

在配置文件中，您可以看到Pod只有一个容器。 `periodSeconds` 字段指定kubelet应该每3秒执行一次活动检测。`initialDelaySeconds` 字段告诉kubelet在执行第一个检测之前应该等待3秒。为了执行检测，kubelet会向正在容器中运行并监听8080端口的服务器发送HTTP GET请求。如果服务器的 `/healthz` 路径的处理程序返回成功代码，kubelet就认为容器是活动的和健康的。如果处理程序返回一个失败代码，kubelet将杀死容器并重新启动它。

任何大于等于200且小于400的代码都表示成功。其他代码都表示失败。

您可以在 [server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/agnhost/liveness/server.go) 中看到该服务器的源代码。

在容器处于活动状态的前10秒， `/healthz` 处理程序返回200状态码。之后，处理程序返回的状态码为500。

```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```

kubelet在容器启动后3秒开始执行健康检查。所以第一批健康检查会成功。但10秒后，健康检查将失败，kubelet将杀死并重新启动容器。

创建一个pod来尝试HTTP活性检测：

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/http-liveness.yaml
```

10秒后，查看Pod事件，确认活性检测失败并且容器已重新启动：

```shell
kubectl describe pod liveness-http
```

在v1.13之前的版本中(包括v1.13)，如果在运行pod的节点上设置了环境变量 `http_proxy` (或 `HTTP_PROXY`)， HTTP活动检测将使用该代理。在v1.13之后的版本中，本地HTTP代理环境变量设置不会影响HTTP活动检测。

#### 定义一个TCP活性探针

第三种活性检测使用TCP套接字。使用此配置，kubelet将尝试打开容器上指定端口的套接字。如果能够建立连接，则认为容器是健康的;如果不能，则认为是不健康的。

[`pods/probe/tcp-liveness-readiness.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/probe/tcp-liveness-readiness.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: goproxy
  labels:
    app: goproxy
spec:
  containers:
  - name: goproxy
    image: k8s.gcr.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 20
```

可以看到，TCP检查的配置非常类似于HTTP检查。这个例子同时使用了就绪和活性检测。kubelet将在容器启动5秒后发送第一个就绪检测器。这将在8080端口上尝试连接goproxy容器。如果检测成功，pod将被标记为ready。kubelet继续每10秒运行一次这个检查。

除了就绪检测外，这个配置还包含活性检测。kubelet将在容器启动15秒后运行第一个活性检测器。像就绪检测一样，这将在8080端口上尝试连接goproxy容器。如果活性检测失败，容器将重新启动。

创建一个pod来尝试TCP活性检测：

```shell
kubectl apply -f https://k8s.io/examples/pods/probe/tcp-liveness-readiness.yaml
```

15秒后，查看pod事件来验证活性检测：

```shell
kubectl describe pod goproxy
```

#### 使用命名的端口（named port）

可以使用已命名的 [容器端口](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#containerport-v1-core) 进行HTTP或TCP活性检测：

```yaml
ports:
- name: liveness-port
  containerPort: 8080
  hostPort: 8080

livenessProbe:
  httpGet:
    path: /healthz
    port: liveness-port
```

#### 定义就绪检测

有时，应用程序会暂时无法为流量提供服务。例如，应用程序可能需要在启动期间加载大量数据或配置文件，或启动后依赖外部服务。在这种情况下，您不想杀死应用程序，但也不想把请求发送给他。Kubernetes提供了就绪检测来检测和缓解这些情况。带有容器报告的pod在没有准备就绪时不会通过Kubernetes服务接收流量。

>   **注意:** 就绪检测在容器的整个生命周期中运行。

就绪检测的配置类似于活动检测。惟一的区别是使用 `readinessProbe` 字段而不是 `livenessProbe` 字段。

```yaml
readinessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 5
```

HTTP 和TCP 就绪检测的配置也与活动检测相同。

就绪和活性探针可以并行地用于同一个容器。同时使用这两种方法可以确保流量不会到达没有就绪的容器，并且在容器失败时重新启动它们。

#### 配置探针

[探针(Probes)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#probe-v1-core) 有许多字段，您可以使用这些字段更精确地控制就绪和活动检测的行为：

-   `initialDelaySeconds`: 容器启动后多少秒开始进行就绪或活性检测
-   `periodSeconds`: 每隔多久执行一次探测(以秒为单位)。默认为10秒。最小值是1。
-   `timeoutSeconds`: :探针超时的秒数。默认为1秒。最小值是1。
-   `successThreshold`: 最小连续成功，就是检测失败后，成功多少次才会被认为是成功。默认为1。活性检测必须为1。最小值是1。
-   `failureThreshold`: 当Pod启动后，检测失败时，Kubernetes会在放弃它之前尝试的 `故障阈值(failureThreshold)` 次数。在活性检测中，放弃它意味着重新启动pod。在就绪检测中，pod将被标记为未就绪。默认为3。最小值是1。

[HTTP 探针](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#httpgetaction-v1-core) 有可以为 `httpGet` 设置的额外字段：

-   `host`: 要连接的主机名，默认是pod的IP。或许你更想在httpHeaders中设置Host而不是这里
-   `scheme`: 用于连接主机的模式(HTTP或HTTPS)。默认为HTTP
-   `path`: 访问HTTP服务的路径
-   `httpHeaders`: 要在请求中设置的自定义头。HTTP允许重复报头
-   `port`: 容器上要访问的端口的名称或编号。数字必须在1到65535之间。

对于HTTP检测，kubelet将HTTP请求发送到指定的路径和端口来执行检查。kubelet将探针发送到pod的IP地址，除非该地址被 `httpGet` 中可选的 `host` 字段覆盖。如果 `scheme` 字段设置为 `HTTPS` , kubelet将发送一个跳过证书验证的HTTPS请求。在大多数情况下，你不想设置 `host` 字段。这是一种设置它的场景。假设容器监听127.0.0.1，Pod的 `hostNetwork` 字段为true。则 `httpGet` 下的 `host` 应设置为127.0.0.1。如果您的pod依赖于虚拟主机(这可能是更常见的情况)，那么您不应该使用 `host`，而应该在 `httpHeaders` 中设置 `Host` 报头。

对于探针，kubelet在节点而不是pod中建立检测连接，这意味着您不能在主机参数中使用服务名，因为kubelet无法解析它。

#### 下一步

-   学习 [容器探针(Container Probes)](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes).

##### 参考

-   [Pod](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#pod-v1-core)
-   [容器(Container)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#container-v1-core)
-   [探针(Probe)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#probe-v1-core)





### 将 Pod 分配给节点

此页面显示如何将 Kubernetes Pod 分配给 Kubernetes 集群中的特定节点。



#### 准备开始

一个集群。



#### 给节点添加标签

1.  列出集群中的节点

    ```
    kubectl get nodes
    ```

    输出类似如下：

    ```
    NAME      STATUS    AGE     VERSION
    worker0   Ready     1d      v1.6.0+fff5156
    worker1   Ready     1d      v1.6.0+fff5156
    worker2   Ready     1d      v1.6.0+fff5156
    ```

1.  选择其中一个节点，为它添加标签：

    ```
    kubectl label nodes <your-node-name> disktype=ssd
    ```

    `<your-node-name>` 是你选择的节点的名称。

1.  验证你选择的节点是否有 `disktype=ssd` 标签：

    kubectl get nodes –show-labels

    输出类似如下：

    ```
    NAME      STATUS    AGE     VERSION            LABELS
    worker0   Ready     1d      v1.6.0+fff5156     ...,disktype=ssd,kubernetes.io/hostname=worker0
    worker1   Ready     1d      v1.6.0+fff5156     ...,kubernetes.io/hostname=worker1
    worker2   Ready     1d      v1.6.0+fff5156     ...,kubernetes.io/hostname=worker2
    ```

    在前面的输出中，你可以看到 `worker0` 节点有 `disktype=ssd` 标签。



#### 创建一个调度到你选择的节点的 pod

该 pod 配置文件描述了一个拥有节点选择器 `disktype: ssd` 的 pod。这表明该 pod 将被调度到 有 `disktype=ssd` 标签的节点。

| [`pods/pod-nginx.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/pods/pod-nginx.yaml) ![Copy pods/pod-nginx.yaml to clipboard](https://d33wubrfki0l68.cloudfront.net/951ae1fcc65e28202164b32c13fa7ae04fab4a0b/b77dc/images/copycode.svg) |
| ------------------------------------------------------------ |
| `apiVersion: v1 kind: Pod metadata:   name: nginx   labels:     env: test spec:   containers:   - name: nginx     image: nginx     imagePullPolicy: IfNotPresent   nodeSelector:     disktype: ssd ` |

1.  使用该配置文件去创建一个 pod，该 pod 将被调度到你选择的节点上：

    ```
    kubectl create -f https://k8s.io/examples/pods/pod-nginx.yaml
    ```

1.  验证 pod 是不是运行在你选择的节点上：

    ```
    kubectl get pods --output=wide
    ```

    输出类似如下：

    ```
    NAME     READY     STATUS    RESTARTS   AGE    IP           NODE
    nginx    1/1       Running   0          13s    10.200.0.4   worker0
    ```





### 配置容器初始化

本文介绍在应用容器运行前，怎样利用 Init 容器初始化 Pod。

#### 准备开始

一个集群。

#### 创建一个包含 Init 容器的 Pod

本例中您将创建一个包含一个应用容器和一个 Init 容器的 Pod。Init 容器在应用容器启动前运行完成。

下面是 Pod 的配置文件：

[`pods/init-containers.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/pods/init-containers.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # These containers are run during pod initialization
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
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

配置文件中，您可以看到应用容器和 Init 容器共享了一个卷。

Init 容器将共享卷挂载到了 `/work-dir` 目录，应用容器将共享卷挂载到了 `/usr/share/nginx/html` 目录。 Init 容器执行完下面的命令就终止：

```
wget -O /work-dir/index.html http://kubernetes.io
```

请注意 Init 容器在 nginx 服务器的根目录写入 `index.html`。

创建 Pod：

```
kubectl create -f https://k8s.io/examples/pods/init-containers.yaml
```

检查 nginx 容器运行正常：

```
kubectl get pod init-demo
```

结果表明 nginx 容器运行正常：

```
NAME        READY     STATUS    RESTARTS   AGE
init-demo   1/1       Running   0          1m
```

通过 shell 进入 init-demo Pod 中的 nginx 容器：

```
kubectl exec -it init-demo -- /bin/bash
```

在 shell 中，发送个 GET 请求到 nginx 服务器：

```
root@nginx:~# apt-get update
root@nginx:~# apt-get install curl
root@nginx:~# curl localhost
```

结果表明 nginx 正在为 Init 容器编写的 web 页面服务：

```
<!Doctype html>
<html id="home">

<head>
...
"url": "http://kubernetes.io/"}</script>
</head>
<body>
  ...
  <p>Kubernetes is open source giving you the freedom to take advantage ...</p>
  ...
```

#### 接下来

-   进一步了解 [相同 Pod 中的容器间的通信](https://v1-14.docs.kubernetes.io/docs/tasks/access-application-cluster/communicate-containers-same-pod-shared-volume/)。
-   进一步了解 [Init 容器](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/pods/init-containers/)。
-   进一步了解 [卷](https://v1-14.docs.kubernetes.io/docs/concepts/storage/volumes/)。
-   进一步了解 [Init 容器排错](https://v1-14.docs.kubernetes.io/docs/tasks/debug-application-cluster/debug-init-containers/)。





### 为容器的生命周期事件设置处理函数

这个页面将演示如何为容器的生命周期事件挂接处理函数。Kubernetes 支持 postStart 和 preStop 事件。 当一个容器启动后，Kubernetes 将立即发送 postStart 事件；在容器被终结之前， Kubernetes 将发送一个 preStop 事件。



#### 准备工作

一个集群



#### 定义 postStart 和 preStop 处理函数

在本练习中，你将创建一个包含一个容器的 Pod，该容器为 postStart 和 preStop 事件提供对应的处理函数。

下面是对应 Pod 的配置文件

[`pods/lifecycle-events.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/pods/lifecycle-events.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/bin/sh","-c","nginx -s quit; while killall -0 nginx; do sleep 1; done"]
```

在上述配置文件中，你可以看到 postStart 命令在容器的 `/usr/share` 目录下写入文件 `message`。 命令 preStop 负责优雅地终止 nginx 服务。当因为失效而导致容器终止时，这一处理方式很有用。```

创建 Pod：

```
kubectl apply -f https://k8s.io/examples/pods/lifecycle-events.yaml
```

验证 Pod 中的容器已经运行：

```
kubectl get pod lifecycle-demo
```

使用 shell 连接到你的 Pod 里的容器：

```
kubectl exec -it lifecycle-demo -- /bin/bash
```

在 shell 中，验证 `postStart` 处理函数创建了 `message` 文件：

```
root@lifecycle-demo:/# cat /usr/share/message
```

命令行输出的是 `postStart` 处理函数所写入的文本

```
Hello from the postStart handler
```



#### 讨论

Kubernetes 在容器创建后立即发送 postStart 事件。然而，postStart 处理函数的调用不保证早于容器的入口点（entrypoint） 的执行。postStart 处理函数与容器的代码是异步执行的，但 Kubernetes 的容器管理逻辑会一直阻塞等待 postStart 处理函数执行完毕。只有 postStart 处理函数执行完毕，容器的状态才会变成 RUNNING。

Kubernetes 在容器结束前立即发送 preStop 事件。除非 Pod 宽限期限超时，Kubernetes 的容器管理逻辑 会一直阻塞等待 preStop 处理函数执行完毕。更多的相关细节，可以参阅 [Pods 的结束](https://kubernetes.io/docs/user-guide/pods/#termination-of-pods)。

>   **Note:** Kubernetes 只有在 Pod *结束（Terminated）* 的时候才会发送 preStop 事件，这意味着在 Pod *完成（Completed）* 时 preStop 的事件处理逻辑不会被触发。这个限制在 [issue #55087](https://github.com/kubernetes/kubernetes/issues/55807) 中被追踪。



#### 接下来

-   进一步了解[容器生命周期回调](https://kubernetes.io/docs/concepts/containers/container-lifecycle-hooks/)。
-   进一步了解[Pod 的生命周期](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)。



#### 参考

-   [生命周期](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#lifecycle-v1-core)
-   [容器](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#container-v1-core)
-   参阅 [PodSpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#podspec-v1-core) 中关于`terminationGracePeriodSeconds` 的部分





### 将pod配置为使用ConfigMap

ConfigMap允许你将配置构件（configuration artifacts）从镜像内容（image content）中解耦，以保证容器化应用程序的可移植性。这一节提供了一系列使用示例来演示如何创建ConfigMap并使用ConfigMap中存储的数据来配置pod。

#### 准备工作

一个集群。

#### 创建ConfigMap

可以使用 `kustomization.yaml` 中的ConfigMap生成器或 `kubectl create configmap` 来创建ConfigMap。注意 `kubectl` 从1.14开始支持 `kustomization.yaml` 。

##### 使用 kubectl create configmap 命令来创建ConfigMap

使用 `kubectl create configmap` 命令通过[目录](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-directories), [文件](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files), 或 [常量值(literal value)]( https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-literal-values) 来创建ConfigMap。

```shell
kubectl create configmap <map-name> <data-source>
```

 <map-name> 是要分配给ConfigMap的名称， <data-source> 是要从中绘制数据的目录、文件或常量值。（顾名思义，数据源）

数据源（data source）在ConfigMap中以键值对形式存在

-   key(键) = 在命令行中提供的键或文件名
-   value(值) = 在命令行中提供的文件内容或常量值

可以使用 [`kubectl describe`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#describe) 或 [`kubectl get`](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands/#get) 来检索ConfigMap的信息。

###### 从目录创建ConfigMap

可以使用 `kubectl create configmap` 从同一目录下的多个文件来创建ConfigMap。

例如：

```shell
# Create the local directory
mkdir -p configure-pod-container/configmap/

# Download the sample files into `configure-pod-container/configmap/` directory
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties

# Create the configmap
kubectl create configmap game-config --from-file=configure-pod-container/configmap/
```

联合 `configure-pod-container/configmap/` 目录中的内容：

```shell
game.properties
ui.properties
```

到下面的ConfigMap：

```shell
kubectl describe configmaps game-config
```

输出大致是：

```
Name:           game-config
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
ui.properties:          83 bytes
```

 `configure-pod-container/configmap/` 目录下的 `game.properties` 和 `ui.properties` 文件在ConfigMap的 `data` 部分中表示。

```shell
kubectl get configmaps game-config -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:52:05Z
  name: game-config
  namespace: default
  resourceVersion: "516"
  selfLink: /api/v1/namespaces/default/configmaps/game-config
  uid: b4952dc3-d670-11e5-8cd0-68f728db1985
data:
  game.properties: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
```

###### 从文件创建ConfigMap

可以使用 `kubectl create configmap` 来从单个或多个文件创建ConfigMap。

例如：

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties
```

会生成下面的ConfigMap：

```shell
kubectl describe configmaps game-config-2
```

输出大致是：

```
Name:           game-config-2
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
```

可以多次传入 `--from-file` 参数来从多个数据源创建ConfigMap。

```shell
kubectl create configmap game-config-2 --from-file=configure-pod-container/configmap/game.properties --from-file=configure-pod-container/configmap/ui.properties
```

查看上面创建的 `game-config-2` ConfigMap：

```shell
kubectl describe configmaps game-config-2
```

输出大致是：

```
Name:           game-config-2
Namespace:      default
Labels:         <none>
Annotations:    <none>

Data
====
game.properties:        158 bytes
ui.properties:          83 bytes
```

使用 `--from-env-file` 选项来从env-file（环境文件）创建ConfigMap，比如：

```shell
# Env-files contain a list of environment variables.
# These syntax rules apply:
#   Each line in an env file has to be in VAR=VAL format.
#   Lines beginning with # (i.e. comments) are ignored.
#   Blank lines are ignored.
#   There is no special handling of quotation marks (i.e. they will be part of the ConfigMap value)).

# Env-files 包含了环境变量的列表
# 语法规则：
#  每一行的格式都要是 变量=值
#  忽略以#开头的行（注释）
#  忽略空行
#  没有对引号做特殊处理，或者说他们也是ConfigMap值的一部分

# Download the sample files into `configure-pod-container/configmap/` directory
wget https://kubernetes.io/examples/configmap/game-env-file.properties -O configure-pod-container/configmap/game-env-file.properties

# The env-file `game-env-file.properties` looks like below
cat configure-pod-container/configmap/game-env-file.properties
enemies=aliens
lives=3
allowed="true"

# This comment and the empty line above it are ignored
kubectl create configmap game-config-env-file \
       --from-env-file=configure-pod-container/configmap/game-env-file.properties
```

将生成下面的ConfigMap：

```shell
kubectl get configmap game-config-env-file -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2017-12-27T18:36:28Z
  name: game-config-env-file
  namespace: default
  resourceVersion: "809965"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-env-file
  uid: d9d1ca5b-eb34-11e7-887b-42010a8002b8
data:
  allowed: '"true"'
  enemies: aliens
  lives: "3"
```

多次使用 `--from-env-file` 来从多个数据源创建ConfigMap时，只是用最后一个env-file：

```shell
# Download the sample files into `configure-pod-container/configmap/` directory
wget https://k8s.io/examples/configmap/ui-env-file.properties -O configure-pod-container/configmap/ui-env-file.properties

# Create the configmap
kubectl create configmap config-multi-env-files \
        --from-env-file=configure-pod-container/configmap/game-env-file.properties \
        --from-env-file=configure-pod-container/configmap/ui-env-file.properties
```

将生成下面的ConfigMap：

```shell
kubectl get configmap config-multi-env-files -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2017-12-27T18:38:34Z
  name: config-multi-env-files
  namespace: default
  resourceVersion: "810136"
  selfLink: /api/v1/namespaces/default/configmaps/config-multi-env-files
  uid: 252c4572-eb35-11e7-887b-42010a8002b8
data:
  color: purple
  how: fairlyNice
  textmode: "true"
```

###### 定义从文件创建ConfigMap时使用的键

使用 `--from-file` 参数时，可以在ConfigMap中的 `data` 部分定义一个键，而不是文件名。

```shell
kubectl create configmap game-config-3 --from-file=<my-key-name>=<path-to-file>
```

 `<my-key-name>` 是希望在ConfigMap中使用的键， `<path-to-file>` 是希望key代表的数据源文件的位置

例如：

```shell
kubectl create configmap game-config-3 --from-file=game-special-key=configure-pod-container/configmap/game.properties
```

将会生成下面的ConfigMap：

```
kubectl get configmaps game-config-3 -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T18:54:22Z
  name: game-config-3
  namespace: default
  resourceVersion: "530"
  selfLink: /api/v1/namespaces/default/configmaps/game-config-3
  uid: 05f8da22-d671-11e5-8cd0-68f728db1985
data:
  game-special-key: |
    enemies=aliens
    lives=3
    enemies.cheat=true
    enemies.cheat.level=noGoodRotten
    secret.code.passphrase=UUDDLRLRBABAS
    secret.code.allowed=true
    secret.code.lives=30
```

###### 从常量值创建ConfigMap

可以为 `kubectl create configmap` 指定 `--from-literal` 参数从命令行定义常量值：

```shell
kubectl create configmap special-config --from-literal=special.how=very --from-literal=special.type=charm
```

您可以传入多个键值对。命令行中提供的每一对都表示为ConfigMap的 `data` 部分中的一个单独条目。

```shell
kubectl get configmaps special-config -o yaml
```

输出大致是：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: special-config
  namespace: default
  resourceVersion: "651"
  selfLink: /api/v1/namespaces/default/configmaps/special-config
  uid: dadce046-d673-11e5-8cd0-68f728db1985
data:
  special.how: very
  special.type: charm
```

##### 从生成器创建ConfigMap

`kubectl` 从1.14开始支持 `kustomization.yaml` 。也可以从生成器中创建一个ConfigMap然后使用它在Apiserver中创建对象。生成器应该在目录下的 `kustomization.yaml` 中指定。

###### 从文件生成ConfigMap

例如要用 `configure-pod-container/configmap/game.properties` 来生成ConfigMap

```shell
# Create a kustomization.yaml file with ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: game-config-4
  files:
  - configure-pod-container/configmap/game.properties
EOF
```

使用kustomization 所在的目录来创建ConfigMap对象。

```shell
kubectl apply -k .
configmap/game-config-4-m9dm2f92bt created
```

可以像这样检查ConfigMap已经创建：

```shell
kubectl get configmap
NAME                       DATA   AGE
game-config-4-m9dm2f92bt   1      37s


kubectl describe configmaps/game-config-4-m9dm2f92bt
Name:         game-config-4-m9dm2f92bt
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"v1","data":{"game.properties":"enemies=aliens\nlives=3\nenemies.cheat=true\nenemies.cheat.level=noGoodRotten\nsecret.code.p...

Data
====
game.properties:
----
enemies=aliens
lives=3
enemies.cheat=true
enemies.cheat.level=noGoodRotten
secret.code.passphrase=UUDDLRLRBABAS
secret.code.allowed=true
secret.code.lives=30
Events:  <none>
```

注意，生成的ConfigMap名称通过散列内容（hashing）来添加后缀。这确保每次修改内容时都会生成一个新的ConfigMap。

###### 定义从文件生成ConfigMap时使用的键

可以在ConfigMap生成器中定义一个键，而不是文件名。比如，从文件 `configure-pod-container/configmap/game.properties` 生成ConfigMap时使用键 `game-special-key`

```shell
# Create a kustomization.yaml file with ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: game-config-5
  files:
  - game-special-key=configure-pod-container/configmap/game.properties
EOF
```

使用 kustomization 所在目录来创建ConfigMap对象

```shell
kubectl apply -k .
configmap/game-config-5-m67dt67794 created
```

###### 使用常量（Literals）来生成ConfigMap

要使用常量 `special.type=charm` 和 `special.how=very` 来生成ConfigMap，可以像这样在 `kustomization.yaml` 中指定ConfigMap生成器：

```shell
# Create a kustomization.yaml file with ConfigMapGenerator
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: special-config-2
  literals:
  - special.how=very
  - special.type=charm
EOF
```

使用 kustomization 所在目录来创建ConfigMap对象

```shell
kubectl apply -k .
configmap/special-config-2-c92b5mmcf2 created
```

#### 使用ConfigMap中的数据来定义容器的环境变量

##### 从单个ConfigMap的data中为容器定义环境变量

1.  在ConfigMap中将环境变量定义为键值对：

    ```shell
    kubectl create configmap special-config --from-literal=special.how=very
    ```

2.  在pod配置文件中将Config Map中定义的 `special.how` 的值分配给 `SPECIAL_LEVEL_KEY` 环境变量

[`pods/pod-single-configmap-env-variable.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-single-configmap-env-variable.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # Define the environment variable
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # The ConfigMap containing the value you want to assign to SPECIAL_LEVEL_KEY
              name: special-config
              # Specify the key associated with the value
              key: special.how
  restartPolicy: Never
```

创建pod：

```shell
 kubectl create -f https://kubernetes.io/examples/pods/pod-single-configmap-env-variable.yaml
```

现在，pod的输出包含了 `SPECIAL_LEVEL_KEY=very` 环境变量。

##### 从多个ConfigMap的data中为容器定义环境变量

-   与前面的示例一样，先创建个ConfigMap

[`configmap/configmaps.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/configmap/configmaps.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  special.how: very
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: env-config
  namespace: default
data:
  log_level: INFO
```

创建ConfigMap：

```shell
 kubectl create -f https://kubernetes.io/examples/configmap/configmaps.yaml
```

-   在pod配置文件中定义环境变量

[`pods/pod-multiple-configmap-env-variable.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-multiple-configmap-env-variable.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: special.how
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: env-config
              key: log_level
  restartPolicy: Never
```

创建pod：

```shell
 kubectl create -f https://kubernetes.io/examples/pods/pod-multiple-configmap-env-variable.yaml
```

现在，pod的输出包含 `SPECIAL_LEVEL_KEY=very` 和 `LOG_LEVEL=INFO` 环境变量。

#### 将ConfigMap中的所有键值对配置为容器的环境变量

>   **注意：** 该功能在kubernetes v1.6和之后的版本中可用。

-   创建一个包含多个键值对的ConfigMap

[`configmap/configmap-multikeys.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/configmap/configmap-multikeys.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

创建ConfigMap

```shell
 kubectl create -f https://kubernetes.io/examples/configmap/configmap-multikeys.yaml
```

-   使用 `envFrom` 将ConfigMap中的所有data定义为容器的环境变量。ConfigMap中的键会成为pod中的环境变量名。

[`pods/pod-configmap-envFrom.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-configmap-envFrom.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: special-config
  restartPolicy: Never
```

创建pod：

```shell
 kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-envFrom.yaml
```

现在，pod的输出包含 `SPECIAL_LEVEL=very` 和 `SPECIAL_TYPE=charm` 环境变量。

#### 在pod命令中使用ConfigMap中定义的环境变量

可以使用 `$(VAR_NAME)` Kubernetes替换语法在Pod配置文件的 `command` 部分使用ConfigMap中定义的环境变量。

比如下面的pod配置文件：

[`pods/pod-configmap-env-var-valueFrom.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-configmap-env-var-valueFrom.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_LEVEL
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: special-config
              key: SPECIAL_TYPE
  restartPolicy: Never
```

使用以下命令创建：

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-env-var-valueFrom.yaml
```

在 `test-container` 容器中生成以下输出：

```shell
very charm
```

#### 将ConfigMap中的data添加到卷

正如 [从文件创建ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#create-configmaps-from-files) （就在上面）中解释的，当使用 `--from-file` 创建ConfigMap时，文件名变成了存储在ConfigMap的 `data` 部分的键。文件内容变成了键对应的值。

这一部分的这个示例引用了一个名为special-config的ConfigMap，如下所示。

[`configmap/configmap-multikeys.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/configmap/configmap-multikeys.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-config
  namespace: default
data:
  SPECIAL_LEVEL: very
  SPECIAL_TYPE: charm
```

创建ConfigMap：

```shell
kubectl create -f https://kubernetes.io/examples/configmap/configmap-multikeys.yaml
```

##### 用ConfigMap中存储的data填充一个卷

在Pod配置文件的 `volumes` 部分添加ConfigMap名。会将ConfigMap的data添加到 `volumeMounts.mountPath` 中指定的目录(在本例中为 `/etc/config`)。 `command` 部分引用存储在ConfigMap中的 `special.level` 项。

[`pods/pod-configmap-volume.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-configmap-volume.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # Provide the name of the ConfigMap containing the files you want
        # to add to the container
        name: special-config
  restartPolicy: Never
```

创建pod：

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-volume.yaml
```

当pod运行时， `ls /etc/config/` 命令生成以下输出：

```shell
SPECIAL_LEVEL
SPECIAL_TYPE
```

>   **小心：** 如果 `/etc/config/` 目录下存在的文件会被删除。

##### 将ConfigMap中存储的data添加到卷的指定路径

使用 `path` 字段为特定的ConfigMap项指定所需的文件路径。在本例中， `SPECIAL_LEVEL` 项将挂载到 `config-volume` 卷中的 `/etc/config/keys` 目录。

[`pods/pod-configmap-volume-specific-key.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/en/examples/pods/pod-configmap-volume-specific-key.yaml) 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: k8s.gcr.io/busybox
      command: [ "/bin/sh","-c","cat /etc/config/keys" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: special-config
        items:
        - key: SPECIAL_LEVEL
          path: keys
  restartPolicy: Never
```

创建pod：

```shell
kubectl create -f https://kubernetes.io/examples/pods/pod-configmap-volume-specific-key.yaml
```

pod运行时， `cat /etc/config/keys` 命令会生成以下输出：

```shell
very
```

>   **小心：** 跟前面那个一样，之前 `/etc/config/` 目录下所有的文件都会被删除。

##### 将键投射到特定路径并设置权限（Project keys to specific paths and file permissions）

可以根据每个文件将键投射到指定的文件和指定的路径。 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod) 用户指南解释了语法。

##### 挂载的ConfigMap会自动更新

当已经在卷中使用的ConfigMap更新时，投射的键最终也会更新。Kubelet会在每次定期同步时检查挂载的ConfigMap是否是最新的。但是，它使用基于ttl的本地缓存来获取ConfigMap的当前值。因此，从ConfigMap更新到将新键投射到pod的总延迟可能与kubelet同步周期(默认为1分钟) + kubelet中ConfigMaps缓存的ttl(默认为1分钟)一样长。

>   **注意：** 使用ConfigMap作为[子路径](https://kubernetes.io/docs/concepts/storage/volumes/#using-subpath) 卷(subPath volume)的容器将不会收到ConfigMap更新

#### 理解ConfigMap和pod

ConfigMap API资源将配置数据存储为键值对。数据可以在pod中使用，也可以为系统组件(如控制器)提供配置。ConfigMap类似于 [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) ，但是它提供了一种处理不包含敏感信息的字符串的方法。用户和系统组件都可以在ConfigMap中存储配置数据。

>   **注意：** ConfigMaps应该引用属性文件（properties files），而不是替换它们。可以将ConfigMap看作表示类似于Linux `/etc` 目录及其内容的东西。例如，如果您从ConfigMap创建一个 [Kubernetes 卷](https://kubernetes.io/docs/concepts/storage/volumes/) ，那么ConfigMap中的每个数据项都由卷中的一个单独的文件表示。

ConfigMap的 `data` 字段包含配置数据。如下面的示例所示，这可以是简单的——比如使用 `--from-literal` 定义的单个属性，也可以是复杂的——比如配置文件，或者是使用 `--from-file` 定义的JSON blob。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  # example of a simple property defined using --from-literal
  example.property.1: hello
  example.property.2: world
  # example of a complex property defined using --from-file
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3
```

##### 限制

-   在Pod配置文件中引用ConfigMap之前，必须创建一个ConfigMap(除非将ConfigMap标记为“可选”)。如果引用不存在的ConfigMap, Pod将不会启动。同样，引用ConfigMap中不存在的键将阻止pod启动。You must create a ConfigMap before referencing it in a Pod specification (unless you mark the ConfigMap as “optional”). If you reference a ConfigMap that doesn’t exist, the Pod won’t start. Likewise, references to keys that don’t exist in the ConfigMap will prevent the pod from starting.
-   如果使用 `envFrom` 从ConfigMaps定义环境变量，被认为无效的键会被跳过。pod会被允许启动，但是无效的名称将记录在事件日志中(`InvalidVariableNames`)。日志消息列出了每个跳过的键。例如：

```shell
   kubectl get events
```

输出大致是：

```console
   LASTSEEN FIRSTSEEN COUNT NAME          KIND  SUBOBJECT  TYPE      REASON                            SOURCE                MESSAGE
   0s       0s        1     dapi-test-pod Pod              Warning   InvalidEnvironmentVariableNames   {kubelet, 127.0.0.1}  Keys [1badkey, 2alsobad] from the EnvFrom configMap default/myconfig were skipped since they are considered invalid environment variable names.
```

-   ConfigMap住在指定的[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)中。ConfigMap只能由住在相同命名空间中的pod引用。
-   Kubelet不支持对API服务器上找不到的pod使用configmap。这包括通过Kubelet的 `--manifest-url` 标志、 `--config` 标志或Kubelet REST API创建的pod。



>   **注意：** 这些不是创建pod的常用方法。



#### 下一步

-   跟随实例： [使用ConfigMap来配置Redis](https://kubernetes.io/docs/tutorials/configuration/configure-redis-using-configmap/)





### 在pod的容器之间共享进程的命名空间

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/configure-pod-container/share-process-namespace/#)

此页面展示如何为 pod 配置进程命名空间共享。 当启用进程命名空间共享时，容器中的进程对该 pod 中的所有其他容器都是可见的。

您可以使用此功能来配置协作容器，比如日志处理 sidecar 容器，或者对那些不包含诸如 shell 等调试实用工具的镜像进行故障排查。

#### 准备工作

-   一个集群。
-   进程命名空间共享是默认启用的 **beta** 版功能。 您可以通过设置 `--feature-gates=PodShareProcessNamespace=false` 禁用此功能。

#### 配置 Pod

进程命名空间共享使用 `v1.PodSpec` 中的 `ShareProcessNamespace` 字段启用。例如：

[`pods/share-process-namespace.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/pods/share-process-namespace.yaml)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    securityContext:
      capabilities:
        add:
        - SYS_PTRACE
    stdin: true
    tty: true
```

1.  在集群中创建 `nginx` pod：

    ```
    kubectl create -f https://k8s.io/examples/pods/share-process-namespace.yaml
    ```

2.  获取容器 `shell`，执行 `ps`：

    ```
    kubectl attach -it nginx -c shell
    ```

    如果没有看到命令提示符，请按 enter 回车键。

    ```
    / # ps ax
    PID   USER     TIME  COMMAND
        1 root      0:00 /pause
        8 root      0:00 nginx: master process nginx -g daemon off;
       14 101       0:00 nginx: worker process
       15 root      0:00 sh
       21 root      0:00 ps ax
    ```

您可以在其他容器中对进程发出信号。例如，发送 `SIGHUP` 到 nginx 以重启工作进程。这需要 `SYS_PTRACE` 功能。

```
/ # kill -HUP 8
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   15 root      0:00 sh
   22 101       0:00 nginx: worker process
   23 root      0:00 ps ax
```

甚至可以使用 `/proc/$pid/root` 链接访问另一个容器镜像。

```
    / # head /proc/8/root/etc/nginx/nginx.conf

    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;


    events {
        worker_connections  1024;
```

#### 理解进程命名空间共享

Pod 共享许多资源，因此它们共享进程命名空间是很有意义的。 不过，有些容器镜像可能希望与其他容器隔离，因此了解这些差异很重要:

1.  **容器进程不再具有 PID 1。** 在没有 PID 1 的情况下，一些容器镜像拒绝启动（例如，使用 `systemd` 的容器)，或者拒绝执行 `kill -HUP 1` 之类的命令来通知容器进程。在具有共享进程命名空间的 pod 中，`kill -HUP 1` 将通知 pod 沙箱（在上面的例子中是 `/pause`）。
2.  **进程对 pod 中的其他容器可见。** 这包括 `/proc` 中可见的所有信息，例如作为参数或环境变量传递的密码。这些仅受常规 Unix 权限的保护。
3.  **容器文件系统通过 `/proc/$pid/root` 链接对 pod 中的其他容器可见。** 这使调试更加容易，但也意味着文件系统安全性只受文件系统权限的保护。





### 将Docker Compose文件转换为kubernetes资源

Kompose 是什么？它是个转换工具，可将 compose（即 Docker Compose）所组装的所有内容转换成容器编排器（Kubernetes 或 OpenShift）可识别的形式。

更多信息请参考 Kompose 官网 [http://kompose.io](http://kompose.io/)。

#### 准备开始

一个集群。

#### 安装 Kompose

我们有很多种方式安装 Kompose。首选方式是从最新的 GitHub 发布页面下载二进制文件。

#### GitHub 发行版本

Kompose 通过 GitHub 发行版本，发布周期为三星期。您可以在[GitHub 发布页面](https://github.com/kubernetes/kompose/releases)上看到所有当前版本。

```sh
# Linux
curl -L https://github.com/kubernetes/kompose/releases/download/v1.16.0/kompose-linux-amd64 -o kompose

# macOS
curl -L https://github.com/kubernetes/kompose/releases/download/v1.16.0/kompose-darwin-amd64 -o kompose

# Windows
curl -L https://github.com/kubernetes/kompose/releases/download/v1.16.0/kompose-windows-amd64.exe -o kompose.exe

chmod +x kompose
sudo mv ./kompose /usr/local/bin/kompose
```

或者，您可以下载 [tarball](https://github.com/kubernetes/kompose/releases)。

#### Go

用 `go get` 命令从主分支拉取最新的开发变更的方法安装 Kompose。

```sh
go get -u github.com/kubernetes/kompose
```

#### CentOS

Kompose 位于 [EPEL](https://fedoraproject.org/wiki/EPEL) CentOS 代码仓库。 如果您还没有安装启用 [EPEL](https://fedoraproject.org/wiki/EPEL) 代码仓库，请运行命令 `sudo yum install epel-release`。

如果您的系统中已经启用了 [EPEL](https://fedoraproject.org/wiki/EPEL)，您就可以像安装其他软件包一样安装 Kompose。

```bash
sudo yum -y install kompose
```

#### Fedora

Kompose 位于 Fedora 24、25 和 26 的代码仓库。您可以像安装其他软件包一样安装 Kompose。

```bash
sudo dnf -y install kompose
```

#### macOS

在 macOS 上您可以通过 [Homebrew](https://brew.sh/) 安装 Kompose 的最新版本：

```bash
brew install kompose
```

#### 使用 Kompose

再需几步，我们就把你从 Docker Compose 带到 Kubernetes。 您只需要一个现有的 `docker-compose.yml` 文件。

1.  进入 `docker-compose.yml` 文件所在的目录。如果没有，请使用下面这个进行测试。

    ```yaml
      version: "2"
    
      services:
    
        redis-master:
          image: k8s.gcr.io/redis:e2e
          ports:
            - "6379"
    
        redis-slave:
          image: gcr.io/google_samples/gb-redisslave:v1
          ports:
            - "6379"
          environment:
            - GET_HOSTS_FROM=dns
    
        frontend:
          image: gcr.io/google-samples/gb-frontend:v4
          ports:
            - "80:80"
          environment:
            - GET_HOSTS_FROM=dns
          labels:
            kompose.service.type: LoadBalancer
    ```

2.  运行 `kompose up` 命令直接部署到 Kubernetes，或者跳到下一步，生成 `kubectl` 使用的文件。

    ```bash
      $ kompose up
      We are going to create Kubernetes Deployments, Services and PersistentVolumeClaims for your Dockerized application.
      If you need different kind of resources, use the 'kompose convert' and 'kubectl create -f' commands instead.
    
      INFO Successfully created Service: redis          
      INFO Successfully created Service: web            
      INFO Successfully created Deployment: redis       
      INFO Successfully created Deployment: web         
    
      Your application has been deployed to Kubernetes. You can run 'kubectl get deployment,svc,pods,pvc' for details.
    ```

3.  要将 `docker-compose.yml` 转换为 `kubectl` 可用的文件，请运行 `kompose convert` 命令进行转换，然后运行 `kubectl create -f <output file>` 进行创建。

    ```bash
      $ kompose convert                           
      INFO Kubernetes file "frontend-service.yaml" created         
      INFO Kubernetes file "redis-master-service.yaml" created     
      INFO Kubernetes file "redis-slave-service.yaml" created      
      INFO Kubernetes file "frontend-deployment.yaml" created      
      INFO Kubernetes file "redis-master-deployment.yaml" created  
      INFO Kubernetes file "redis-slave-deployment.yaml" created   
    ```

    ```bash
      $ kubectl create -f frontend-service.yaml,redis-master-service.yaml,redis-slave-service.yaml,frontend-deployment.yaml,redis-master-deployment.yaml,redis-slave-deployment.yaml
      service/frontend created
      service/redis-master created
      service/redis-slave created
      deployment.apps/frontend created
      deployment.apps/redis-master created
      deployment.apps/redis-slave created
    ```

    您部署的应用在 Kubernetes 中运行起来了。

4.  访问您的应用。

    

    如果您在开发过程中使用 `minikube`，请执行：

    ```bash
      $ minikube service frontend
    ```

    否则，我们要查看一下您的服务使用了什么 IP！

    ```sh
      $ kubectl describe svc frontend
      Name:                   frontend
      Namespace:              default
      Labels:                 service=frontend
      Selector:               service=frontend
      Type:                   LoadBalancer
      IP:                     10.0.0.183
      LoadBalancer Ingress:   123.45.67.89
      Port:                   80      80/TCP
      NodePort:               80      31144/TCP
      Endpoints:              172.17.0.4:80
      Session Affinity:       None
      No events.
    ```

    如果您使用的是云提供商，您的 IP 将在 `LoadBalancer Ingress` 字段给出。

    ```sh
      $ curl http://123.45.67.89
    ```

#### 用户指南

-   CLI
    -   [`kompose convert`](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#kompose-convert)
    -   [`kompose up`](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#kompose-up)
    -   [`kompose down`](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#kompose-down)
-   文档
    -   [构建和推送 Docker 镜像](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#%E6%9E%84%E5%BB%BA%E5%92%8C%E6%8E%A8%E9%80%81-docker-%E9%95%9C%E5%83%8F)
    -   [其他转换方式](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#%E5%85%B6%E4%BB%96%E8%BD%AC%E6%8D%A2%E6%96%B9%E5%BC%8F)
    -   [标签](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#%E6%A0%87%E7%AD%BE)
    -   [重启](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#%E9%87%8D%E5%90%AF)
    -   [Docker Compose 版本](https://v1-14.docs.kubernetes.io/zh/docs/tasks/configure-pod-container/translate-compose-kubernetes/#docker-compose-%E7%89%88%E6%9C%AC)

Kompose 支持两种驱动：OpenShift 和 Kubernetes。 您可以通过全局选项 `--provider` 选择驱动方式。如果没有指定，会将 Kubernetes 作为默认驱动。

#### `kompose convert`

Kompose 支持将 V1、V2 和 V3 版本的 Docker Compose 文件转换为 Kubernetes 和 OpenShift 资源对象。

##### Kubernetes

```sh
$ kompose --file docker-voting.yml convert
WARN Unsupported key networks - ignoring
WARN Unsupported key build - ignoring
INFO Kubernetes file "worker-svc.yaml" created
INFO Kubernetes file "db-svc.yaml" created
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "result-svc.yaml" created
INFO Kubernetes file "vote-svc.yaml" created
INFO Kubernetes file "redis-deployment.yaml" created
INFO Kubernetes file "result-deployment.yaml" created
INFO Kubernetes file "vote-deployment.yaml" created
INFO Kubernetes file "worker-deployment.yaml" created
INFO Kubernetes file "db-deployment.yaml" created

$ ls
db-deployment.yaml  docker-compose.yml         docker-gitlab.yml  redis-deployment.yaml  result-deployment.yaml  vote-deployment.yaml  worker-deployment.yaml
db-svc.yaml         docker-voting.yml          redis-svc.yaml     result-svc.yaml        vote-svc.yaml           worker-svc.yaml
```

您也可以同时提供多个 docker-compose 文件进行转换：

```sh
$ kompose -f docker-compose.yml -f docker-guestbook.yml convert
INFO Kubernetes file "frontend-service.yaml" created         
INFO Kubernetes file "mlbparks-service.yaml" created         
INFO Kubernetes file "mongodb-service.yaml" created          
INFO Kubernetes file "redis-master-service.yaml" created     
INFO Kubernetes file "redis-slave-service.yaml" created      
INFO Kubernetes file "frontend-deployment.yaml" created      
INFO Kubernetes file "mlbparks-deployment.yaml" created      
INFO Kubernetes file "mongodb-deployment.yaml" created       
INFO Kubernetes file "mongodb-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "redis-master-deployment.yaml" created  
INFO Kubernetes file "redis-slave-deployment.yaml" created   

$ ls
mlbparks-deployment.yaml  mongodb-service.yaml                       redis-slave-service.jsonmlbparks-service.yaml  
frontend-deployment.yaml  mongodb-claim0-persistentvolumeclaim.yaml  redis-master-service.yaml
frontend-service.yaml     mongodb-deployment.yaml                    redis-slave-deployment.yaml
redis-master-deployment.yaml
```

当提供多个 docker-compose 文件时，配置将会合并。任何通用的配置都将被后续文件覆盖。

##### OpenShift

```sh
$ kompose --provider openshift --file docker-voting.yml convert
WARN [worker] Service cannot be created because of missing port.
INFO OpenShift file "vote-service.yaml" created             
INFO OpenShift file "db-service.yaml" created               
INFO OpenShift file "redis-service.yaml" created            
INFO OpenShift file "result-service.yaml" created           
INFO OpenShift file "vote-deploymentconfig.yaml" created    
INFO OpenShift file "vote-imagestream.yaml" created         
INFO OpenShift file "worker-deploymentconfig.yaml" created  
INFO OpenShift file "worker-imagestream.yaml" created       
INFO OpenShift file "db-deploymentconfig.yaml" created      
INFO OpenShift file "db-imagestream.yaml" created           
INFO OpenShift file "redis-deploymentconfig.yaml" created   
INFO OpenShift file "redis-imagestream.yaml" created        
INFO OpenShift file "result-deploymentconfig.yaml" created  
INFO OpenShift file "result-imagestream.yaml" created  
```

kompose 还支持为服务中的构建指令创建 buildconfig。 默认情况下，它使用当前 git 分支的 remote 仓库作为源仓库，使用当前分支作为构建的源分支。 您可以分别使用 `--build-repo`和 `--build-branch` 选项指定不同的源仓库和分支。

```sh
$ kompose --provider openshift --file buildconfig/docker-compose.yml convert
WARN [foo] Service cannot be created because of missing port.
INFO OpenShift Buildconfig using git@github.com:rtnpro/kompose.git::master as source.
INFO OpenShift file "foo-deploymentconfig.yaml" created     
INFO OpenShift file "foo-imagestream.yaml" created          
INFO OpenShift file "foo-buildconfig.yaml" created
```

>   Note:
>
>   如果使用 `oc create -f` 手动推送 Openshift 工件，则需要确保在构建配置工件之前推送 imagestream 工件，以解决 Openshift 的这个问题：<https://github.com/openshift/origin/issues/4518> 。

#### `kompose up`

Kompose 支持通过 `kompose up` 直接将您的”复合的（composed）” 应用程序部署到 Kubernetes 或 OpenShift。

##### Kubernetes

```sh
$ kompose --file ./examples/docker-guestbook.yml up
We are going to create Kubernetes deployments and services for your Dockerized application.
If you need different kind of resources, use the 'kompose convert' and 'kubectl create -f' commands instead.

INFO Successfully created service: redis-master   
INFO Successfully created service: redis-slave    
INFO Successfully created service: frontend       
INFO Successfully created deployment: redis-master
INFO Successfully created deployment: redis-slave
INFO Successfully created deployment: frontend    

Your application has been deployed to Kubernetes. You can run 'kubectl get deployment,svc,pods' for details.

$ kubectl get deployment,svc,pods
NAME                                              DESIRED       CURRENT       UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/frontend                    1             1             1            1           4m
deployment.extensions/redis-master                1             1             1            1           4m
deployment.extensions/redis-slave                 1             1             1            1           4m

NAME                         TYPE               CLUSTER-IP    EXTERNAL-IP   PORT(S)      AGE
service/frontend             ClusterIP          10.0.174.12   <none>        80/TCP       4m
service/kubernetes           ClusterIP          10.0.0.1      <none>        443/TCP      13d
service/redis-master         ClusterIP          10.0.202.43   <none>        6379/TCP     4m
service/redis-slave          ClusterIP          10.0.1.85     <none>        6379/TCP     4m

NAME                                READY         STATUS        RESTARTS     AGE
pod/frontend-2768218532-cs5t5       1/1           Running       0            4m
pod/redis-master-1432129712-63jn8   1/1           Running       0            4m
pod/redis-slave-2504961300-nve7b    1/1           Running       0            4m
```

**注意**：

-   您必须有一个运行正常的 Kubernetes 集群，该集群具有预先配置的 kubectl 上下文。
-   此操作仅生成 Deployment 和 Service 对象并将其部署到 Kubernetes。如果需要部署其他不同类型的资源，请使用 `kompose convert` 和 `kubectl create -f` 命令。

##### OpenShift

```sh
$ kompose --file ./examples/docker-guestbook.yml --provider openshift up
We are going to create OpenShift DeploymentConfigs and Services for your Dockerized application.
If you need different kind of resources, use the 'kompose convert' and 'oc create -f' commands instead.

INFO Successfully created service: redis-slave    
INFO Successfully created service: frontend       
INFO Successfully created service: redis-master   
INFO Successfully created deployment: redis-slave
INFO Successfully created ImageStream: redis-slave
INFO Successfully created deployment: frontend    
INFO Successfully created ImageStream: frontend   
INFO Successfully created deployment: redis-master
INFO Successfully created ImageStream: redis-master

Your application has been deployed to OpenShift. You can run 'oc get dc,svc,is' for details.

$ oc get dc,svc,is
NAME               REVISION                              DESIRED       CURRENT    TRIGGERED BY
dc/frontend        0                                     1             0          config,image(frontend:v4)
dc/redis-master    0                                     1             0          config,image(redis-master:e2e)
dc/redis-slave     0                                     1             0          config,image(redis-slave:v1)
NAME               CLUSTER-IP                            EXTERNAL-IP   PORT(S)    AGE
svc/frontend       172.30.46.64                          <none>        80/TCP     8s
svc/redis-master   172.30.144.56                         <none>        6379/TCP   8s
svc/redis-slave    172.30.75.245                         <none>        6379/TCP   8s
NAME               DOCKER REPO                           TAGS          UPDATED
is/frontend        172.30.12.200:5000/fff/frontend                     
is/redis-master    172.30.12.200:5000/fff/redis-master                 
is/redis-slave     172.30.12.200:5000/fff/redis-slave    v1  
```

**注意**：

-   您必须有一个运行正常的 OpenShift 集群，该集群具有预先配置的 `oc` 上下文 (`oc login`)。

#### `kompose down`

您一旦将”复合(composed)” 应用部署到 Kubernetes，`$ kompose down` 命令将能帮您通过删除 Deployment 和 Service 对象来删除应用。如果需要删除其他资源，请使用 ‘kubectl’ 命令。

```sh
$ kompose --file docker-guestbook.yml down
INFO Successfully deleted service: redis-master   
INFO Successfully deleted deployment: redis-master
INFO Successfully deleted service: redis-slave    
INFO Successfully deleted deployment: redis-slave
INFO Successfully deleted service: frontend       
INFO Successfully deleted deployment: frontend
```

**注意**：

-   您必须有一个运行正常的 Kubernetes 集群，该集群具有预先配置的 kubectl 上下文。

#### 构建和推送 Docker 镜像

Kompose 支持构建和推送 Docker 镜像。如果 Docker Compose 文件中使用了 `build` 关键字，您的镜像将会：

-   使用文档中指定的 `image` 键自动构建 Docker 镜像
-   使用本地凭据推送到正确的 Docker 仓库

使用 [Docker Compose 文件示例](https://raw.githubusercontent.com/kubernetes/kompose/master/examples/buildconfig/docker-compose.yml)

```yaml
version: "2"

services:
    foo:
        build: "./build"
        image: docker.io/foo/bar
```

使用带有 `build` 键的 `kompose up` 命令：

```none
$ kompose up
INFO Build key detected. Attempting to build and push image 'docker.io/foo/bar'
INFO Building image 'docker.io/foo/bar' from directory 'build'
INFO Image 'docker.io/foo/bar' from directory 'build' built successfully
INFO Pushing image 'foo/bar:latest' to registry 'docker.io'
INFO Attempting authentication credentials 'https://index.docker.io/v1/
INFO Successfully pushed image 'foo/bar:latest' to registry 'docker.io'
INFO We are going to create Kubernetes Deployments, Services and PersistentVolumeClaims for your Dockerized application. If you need different kind of resources, use the 'kompose convert' and 'kubectl create -f' commands instead.

INFO Deploying application in "default" namespace
INFO Successfully created Service: foo            
INFO Successfully created Deployment: foo         

Your application has been deployed to Kubernetes. You can run 'kubectl get deployment,svc,pods,pvc' for details.
```

要想禁用该功能，或者使用 BuildConfig 中的版本（在 OpenShift 中），可以通过传递 `--build (local|build-config|none)` 参数来实现。

```sh
# Disable building/pushing Docker images
$ kompose up --build none

# Generate Build Config artifacts for OpenShift
$ kompose up --provider openshift --build build-config
```

#### 其他转换方式

默认的 `kompose` 转换会生成 yaml 格式的 Kubernetes [Deployment](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/deployment/) 和 [Service](https://v1-14.docs.kubernetes.io/docs/concepts/services-networking/service/) 对象。 您可以选择通过 `-j` 参数生成 json 格式的对象。 您也可以替换生成 [Replication Controllers](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/) 对象、[Daemon Sets](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 或 [Helm](https://github.com/helm/helm) charts。

```sh
$ kompose convert -j
INFO Kubernetes file "redis-svc.json" created
INFO Kubernetes file "web-svc.json" created
INFO Kubernetes file "redis-deployment.json" created
INFO Kubernetes file "web-deployment.json" created
```

`*-deployment.json` 文件中包含 Deployment 对象。

```sh
$ kompose convert --replication-controller
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-replicationcontroller.yaml" created
INFO Kubernetes file "web-replicationcontroller.yaml" created
*-replicationcontroller.yaml` 文件包含 Replication Controller 对象。如果您想指定副本数（默认为 1），可以使用 `--replicas` 参数：`$ kompose convert --replication-controller --replicas 3
$ kompose convert --daemon-set
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-daemonset.yaml" created
INFO Kubernetes file "web-daemonset.yaml" created
```

`*-daemonset.yaml` 文件包含 Daemon Set 对象。

如果您想生成 [Helm](https://github.com/kubernetes/helm) 可用的 Chart，只需简单的执行下面的命令：

```sh
$ kompose convert -c
INFO Kubernetes file "web-svc.yaml" created
INFO Kubernetes file "redis-svc.yaml" created
INFO Kubernetes file "web-deployment.yaml" created
INFO Kubernetes file "redis-deployment.yaml" created
chart created in "./docker-compose/"

$ tree docker-compose/
docker-compose
├── Chart.yaml
├── README.md
└── templates
    ├── redis-deployment.yaml
    ├── redis-svc.yaml
    ├── web-deployment.yaml
    └── web-svc.yaml
```

这个图标结构旨在为构建 Helm Chart 提供框架。

#### 标签

`kompose` 支持 `docker-compose.yml` 文件中用于 Kompose 的标签，以便在转换时明确定义 Service 的行为。

-   `kompose.service.type` 定义要创建的 Service 类型。

```yaml
version: "2"
services:
  nginx:
    image: nginx
    dockerfile: foobar
    build: ./foobar
    cap_add:
      - ALL
    container_name: foobar
    labels:
      kompose.service.type: nodeport
```

-   ```
    kompose.service.expose
    ```

     

    定义 是否允许从集群外部访问 Service。如果该值被设置为 “true”，提供程序将自动设置端点，对于任何其他值，该值将被设置为主机名。如果在 Service 中定义了多个端口，则选择第一个端口作为公开端口。

    -   对于 Kubernetes 驱动程序，创建了一个 Ingress 资源，并且假定已经配置了相应的 Ingress 控制器。
    -   对于 OpenShift 驱动程序, 创建一个 route。

例如：

```yaml
version: "2"
services:
  web:
    image: tuna/docker-counter23
    ports:
     - "5000:5000"
    links:
     - redis
    labels:
      kompose.service.expose: "counter.example.com"
  redis:
    image: redis:3.0
    ports:
     - "6379"
```

当前支持的选项有:

| 键                     | 值                                  |
| :--------------------- | :---------------------------------- |
| kompose.service.type   | nodeport / clusterip / loadbalancer |
| kompose.service.expose | true / hostname                     |

>   Note:
>
>   `kompose.service.type` 标签应该只用`ports`来定义，否则 `kompose` 会失败。

#### 重启

如果你想创建没有控制器的普通 Pod，可以使用 docker-compose 的 `restart` 结构来定义它。请参考下表了解 `restart` 的不同参数。

| `docker-compose` `restart` | 创建的对象 | Pod `restartPolicy` |
| :------------------------- | :--------- | :------------------ |
| `""`                       | 控制器对象 | `Always`            |
| `always`                   | 控制器对象 | `Always`            |
| `on-failure`               | Pod        | `OnFailure`         |
| `no`                       | Pod        | `Never`             |

>   Note:
>
>   控制器对象可以是 `deployment` 或 `replicationcontroller` 等。

例如，`pival` Service 将在这里变成 Pod。这个容器的计算值为 `pi`。

```yaml
version: '2'

services:
  pival:
    image: perl
    command: ["perl",  "-Mbignum=bpi", "-wle", "print bpi(2000)"]
    restart: "on-failure"
```

##### 关于 Deployment Config 的提醒

如果 Docker Compose 文件中为服务声明了卷，Deployment (Kubernetes) 或 DeploymentConfig (OpenShift) 的策略会从 “RollingUpdate” (默认) 变为 “Recreate”。 这样做的目的是为了避免服务的多个实例同时访问卷。

如果 Docker Compose 文件中的服务名包含 `_` (例如 `web_service`)，那么将会被替换为 `-`，服务也相应的会重命名(例如 `web-service`)。 Kompose 这样做的原因是 “Kubernetes” 不允许对象名称中包含 `_`。

#### Docker Compose 版本

Kompose 支持的 Docker Compose 版本包括：1、2 和 3。有限支持 2.1 和 3.2 版本，因为它们还在实验阶段。

所有三个版本的兼容性列表请查看我们的 [转换文档](https://github.com/kubernetes/kompose/blob/master/docs/conversion.md)，文档中列出了所有不兼容的 Docker Compose 关键字。