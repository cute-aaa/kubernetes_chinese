# 运行 Job

## 使用CronJob运行自动化任务

你可以利用 [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs) 执行基于时间调度的任务。这些自动化任务和 Linux 或者 Unix 系统的 [Cron](https://en.wikipedia.org/wiki/Cron) 任务类似。

CronJobs 在创建周期性以及重复性的任务时很有帮助，例如执行备份操作或者发送邮件。CronJobs 也可以在特定时间调度单个任务，例如你想调度低活跃周期的任务。

>   Note:
>
>   从集群版本1.8开始，`batch/v2alpha1` API 组中的 CronJob 资源已经被废弃。 你应该切换到 API 服务器默认启用的 `batch/v1beta1` API 组。本文中的所有示例使用了`batch/v1beta1`。

CronJobs 有一些限制和特点。 例如，在特定状况下，同一个 CronJob 可以创建多个任务。 因此，任务应该是幂等的。 查看更多限制，请参考 [CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs)。

### 准备工作

-   一个集群

-   你需要一个版本 >=1.8 且工作正常的 Kubernetes 集群。对于更早的版本（ <1.8 ），你需要对 API 服务器设置 `--runtime-config=batch/v2alpha1=true` 来开启 `batch/v2alpha1` API，(更多信息请查看 [为你的集群开启或关闭 API 版本](https://kubernetes.io/docs/admin/cluster-management/#turn-on-or-off-an-api-version-for-your-cluster) ), 然后重启 API 服务器和控制管理器。

### 创建 CronJob

CronJob 需要一个配置文件。 本例中 CronJob 的`.spec` 配置文件每分钟打印出当前时间和一个问好信息：

[`application/job/cronjob.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/cronjob.yaml)

```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```

想要运行示例的 CronJob，可以下载示例文件并执行命令：

```shell
$ kubectl create -f ./cronjob.yaml
cronjob "hello" created
```

或者你也可以使用 `kubectl run` 来创建一个 CronJob 而不需要编写完整的配置：

```shell
$ kubectl run hello --schedule="*/1 * * * *" --restart=OnFailure --image=busybox -- /bin/sh -c "date; echo Hello from the Kubernetes cluster"
cronjob "hello" created
```

创建好 CronJob 后，使用下面的命令来获取其状态：

```shell
$ kubectl get cronjob hello
NAME      SCHEDULE      SUSPEND   ACTIVE    LAST-SCHEDULE
hello     */1 * * * *   False     0         <none>
```

就像你从命令返回结果看到的那样，CronJob 还没有调度或执行任何任务。大约需要一分钟任务才能创建好。

```shell
$ kubectl get jobs --watch
NAME               DESIRED   SUCCESSFUL   AGE
hello-4111706356   1         1         2s
```

现在你已经看到了一个运行中的任务被 “hello” CronJob 调度。你可以停止监视这个任务，然后再次查看 CronJob 就能看到它调度任务：

```shell
$ kubectl get cronjob hello
NAME      SCHEDULE      SUSPEND   ACTIVE    LAST-SCHEDULE
hello     */1 * * * *   False     0         Mon, 29 Aug 2016 14:34:00 -0700
```

你应该能看到 “hello” CronJob 在 `LAST-SCHEDULE` 声明的时间点成功的调度了一次任务。有0个活跃的任务意味着任务执行完毕或者执行失败。

现在，找到最后一次调度任务创建的 Pod 并查看一个 Pod 的标准输出。请注意任务名称和 Pod 名称是不同的。

```shell
# Replace "hello-4111706356" with the job name in your system
$ pods=$(kubectl get pods --selector=job-name=hello-4111706356 --output=jsonpath={.items..metadata.name})

$ echo $pods
hello-4111706356-o9qcm

$ kubectl logs $pods
Mon Aug 29 21:34:09 UTC 2016
Hello from the Kubernetes cluster
```

### 删除 CronJob

当你不再需要 CronJob 时，可以用 `kubectl delete cronjob` 删掉它：

```shell
$ kubectl delete cronjob hello
cronjob "hello" deleted
```

删除 CronJob 会清除它创建的所有任务和 Pod，并阻止它创建额外的任务。你可以查阅 [垃圾收集](https://kubernetes.io/docs/concepts/workloads/controllers/garbage-collection/)。

### 编写 CronJob 声明信息

像 Kubernetes 的其他配置一样，CronJob 需要 `apiVersion`、 `kind`、 和 `metadata` 域。配置文件的一般信息，请参考 [部署应用](https://kubernetes.io/docs/user-guide/deploying-applications) 和 [使用 kubectl 管理资源](https://kubernetes.io/docs/user-guide/working-with-resources).

CronJob 配置也需要包括[`.spec`](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status).

>   Note:
>
>   对 CronJob 的所有改动，特别是它的 `.spec`，只会影响将来的运行实例。

#### 时间安排

`.spec.schedule` 是 `.spec` 需要的域。它使用了 [Cron](https://en.wikipedia.org/wiki/Cron) 格式串，例如 `0 * * * *` or `@hourly` ，做为它的任务被创建和执行的调度时间。

该格式也包含了扩展的 `vixie cron` 步长值。[FreeBSD 手册](https://www.freebsd.org/cgi/man.cgi?crontab%285%29)中解释如下:

>   步长可被用于范围组合。范围后面带有 `/<数字>'' 可以声明范围内的步幅数值。 例如，`0-23/2” 可被用在小时域来声明命令在其他数值的小时数执行（ V7 标准中对应的方法是`0,2,4,6,8,10,12,14,16,18,20,22''）。 步长也可以放在通配符后面，因此如果你想表达`每两小时”，就用 ``*/2” 。

>   Note:
>
>   调度中的问号 (`?`) 和星号 `*` 含义相同，表示给定域的任何可用值。

#### 任务模版

`.spec.jobTemplate`是任务的模版，它是必须的。它和 [Job](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/)的语法完全一样，除了它是嵌套的没有 `apiVersion` 和 `kind`。 编写任务的 `.spec` ，请参考 [编写任务的Spec](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/#writing-a-job-spec)。

#### 开始的最后期限

`.spec.startingDeadlineSeconds` 域是可选的。 它表示任务如果由于某种原因错过了调度时间，开始该任务的截止时间的秒数。过了截止时间，CronJob 就不会开始任务。 不满足这种最后期限的任务会被统计为失败任务。如果该域没有声明，那任务就没有最后期限。

CronJob 控制器会统计错过了多少次调度。如果错过了100次以上的调度，CronJob 就不再调度了。当没有设置 `.spec.startingDeadlineSeconds` 时，CronJob 控制器统计从`status.lastScheduleTime`到当前的调度错过次数。 例如一个 CronJob 期望每分钟执行一次，`status.lastScheduleTime`是 5:00am，但现在是 7:00am。那意味着120次调度被错过了，所以 CronJob 将不再被调度。 如果设置了 `.spec.startingDeadlineSeconds` 域(非空)，CronJob 控制器统计从 `.spec.startingDeadlineSeconds` 到当前时间错过了多少次任务。 例如设置了 `200`，它会统计过去200秒内错过了多少次调度。在那种情况下，如果过去200秒内错过了超过100次的调度，CronJob 就不再调度。

#### 并发性规则

`.spec.concurrencyPolicy` 也是可选的。它声明了 CronJob 创建的任务执行时发生重叠如何处理。spec 仅能声明下列规则中的一种：

-   `Allow` (默认)：CronJob 允许并发任务执行。
-   `Forbid`： CronJob 不允许并发任务执行；如果新任务的执行时间到了而老任务没有执行完，CronJob 会忽略新任务的执行。
-   `Replace`：如果新任务的执行时间到了而老任务没有执行完，CronJob 会用新任务替换当前正在运行的任务。

请注意，并发性规则仅适用于相同 CronJob 创建的任务。如果有多个 CronJob，它们相应的任务总是允许并发执行的。

#### 挂起

`.spec.suspend`域也是可选的。如果设置为 `true` ，后续发生的执行都会挂起。这个设置对已经开始的执行不起作用。默认是关闭的。

>   Caution:
>
>   在调度时间内挂起的执行都会被统计为错过的任务。当 `.spec.suspend` 从 `true` 改为 `false` 时，且没有 [开始的最后期限](https://kubernetes.io/zh/docs/tasks/job/#starting-deadline)，错过的任务会被立即调度。

#### 任务历史限制

`.spec.successfulJobsHistoryLimit` 和 `.spec.failedJobsHistoryLimit`是可选的。 这两个域声明了有多少执行完成和失败的任务会被保留。 默认设置为3和1。限制设置为0代表相应类型的任务完成后不会保留。





## 使用扩展进行并行处理

>   这个译者把一些命令跟输出结果合并了

在这个示例中，我们将运行从一个公共模板创建的多个 Kubernetes 作业。您可能希望熟悉[Jobs](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) 的基本、非并行使用。

### 基本模板扩展

首先，将以下作业模板下载到名为 `job-tmpl.yaml` 的文件中

[`application/job/job-tmpl.yaml` ](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/job-tmpl.yaml)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```

与 *pod 模板*不同，我们的 *job 模板*不是 Kubernetes API 类型。它只是作业对象的 yaml 表示， YAML 文件有一些占位符，在使用它之前需要填充这些占位符。`$ITEM` 语法对 Kubernetes 没有意义。

在这个例子中，容器所做的唯一处理是 `echo` 一个字符串并休眠一段时间。 在真实的用例中，处理将是一些重要的计算，例如呈现电影的帧，或者处理数据库中的一系列行。例如，`$ITEM` 参数将指定帧号或行范围。

这个作业及其 Pod 模板有一个标签: `jobgroup=jobexample`。这个标签在系统中没有什么特别之处。 这个标签使得我们可以方便地同时操作组中的所有作业。 我们还将相同的标签放在 pod 模板上，这样我们就可以用一个命令检查这些作业的所有 pod。 创建作业之后，系统将添加更多的标签来区分一个作业的 pod 和另一个作业的 pod。 注意，标签键 `jobgroup` 对 Kubernetes 并无特殊含义。您可以选择自己的标签方案。

下一步，将模板展开到多个文件中，每个文件对应要处理的项。

```shell
# Expand files into a temporary directory
$ mkdir ./jobs
$ for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
```

检查是否工作正常：

```shell
$ ls jobs/
job-apple.yaml
job-banana.yaml
job-cherry.yaml
```

在这里，我们使用 `sed` 将字符串 `$ITEM` 替换为循环变量。 您可以使用任何类型的模板语言(jinja2, erb) 或编写程序来生成作业对象。

接下来，使用 kubectl 命令创建所有作业：

```shell
$ kubectl create -f ./jobs
job "process-item-apple" created
job "process-item-banana" created
job "process-item-cherry" created
```

现在，检查这些作业：

```shell
$ kubectl get jobs -l jobgroup=jobexample
NAME                  DESIRED   SUCCESSFUL   AGE
process-item-apple    1         1            31s
process-item-banana   1         1            31s
process-item-cherry   1         1            31s
```

在这里，我们使用 `-l` 选项选择属于这组作业的所有作业。(系统中可能还有其他不相关的工作，我们不想看到。)

我们可以检查 pod 以及使用同样地标签选择器：

```shell
$ kubectl get pods -l jobgroup=jobexample
NAME                        READY     STATUS      RESTARTS   AGE
process-item-apple-kixwv    0/1       Completed   0          4m
process-item-banana-wrsf7   0/1       Completed   0          4m
process-item-cherry-dnfu9   0/1       Completed   0          4m
```

没有一个命令可以一次检查所有作业的输出，但是循环遍历所有 pod 非常简单：

```shell
$ for p in $(kubectl get pods -l jobgroup=jobexample -o name)
do
  kubectl logs $p
done
Processing item apple
Processing item banana
Processing item cherry
```

### 多个模板参数

在第一个示例中，模板的每个实例都有一个参数，该参数也用作标签。 但是标签的键名在[可包含的字符](https://v1-14.docs.kubernetes.io/docs/concepts/overview/working-with-objects/labels/#syntax-and-character-set)方面有一定的约束。

这个稍微复杂一点的示例使用 jinja2 板语言来生成我们的对象。 我们将使用一行 python 脚本将模板转换为文件。

首先，粘贴作业对象的以下模板到一个名为 `job.yaml.jinja2` 的文件中：

```liquid
{%- set params = [{ "name": "apple", "url": "http://www.orangepippin.com/apples", },
                  { "name": "banana", "url": "https://en.wikipedia.org/wiki/Banana", },
                  { "name": "raspberry", "url": "https://www.raspberrypi.org/" }]
%}
{%- for p in params %}
{%- set name = p["name"] %}
{%- set url = p["url"] %}
apiVersion: batch/v1
kind: Job
metadata:
  name: jobexample-{{ name }}
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing URL {{ url }} && sleep 5"]
      restartPolicy: Never
---
{%- endfor %}
```

上面的模板使用 python dicts 列表（第1-4行）定义每个作业对象的参数。 然后 for 循环为每组参数（剩余行）生成一个作业 yaml 对象。 我们利用了多个 yaml 文档可以与 `---` 分隔符连接的事实（倒数第二行）。 我们可以将输出直接传递给 kubectl 来创建对象。

如果您还没有 jinja2 包则需要安装它: `pip install --user jinja2`。 现在，使用这个一行 python 程序来展开模板:

```shell
alias render_template='python -c "from jinja2 import Template; import sys; print(Template(sys.stdin.read()).render());"'
```

输出可以保存到一个文件，像这样：

```shell
cat job.yaml.jinja2 | render_template > jobs.yaml
```

或直接发送到 kubectl，如下所示：

```shell
cat job.yaml.jinja2 | render_template | kubectl create -f -
```

### 替代方案

如果您有大量作业对象，您可能会发现：

-   即使使用标签，管理这么多作业对象也很麻烦。
-   在一次创建所有作业时，您超过了资源配额，可是您也不希望以递增方式创建作业并等待其完成。
-   同时创建的大量作业会使 Kubernetes apiserver、控制器或调度程序过载。 

在这种情况下，您可以考虑 其他[工作模式](https://v1-14.docs.kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/#job-patterns)。





## 使用工作队列进行粗略并行处理

本例中，我们会运行包含多个并行工作进程的 Kubernetes Job。

本例中，每个 Pod 一旦被创建，会立即从任务队列中取走一个工作单元并完成它，然后将工作单元从队列中删除后再退出。

下面是本次示例的主要步骤：

1.  **启动一个消息队列服务** 本例中，我们使用 RabbitMQ，你也可以用其他的消息队列服务。在实际工作环境中，你可以创建一次消息队列服务然后在多个任务中重复使用。
2.  **创建一个队列，放上消息数据** 每个消息表示一个要执行的任务。本例中，每个消息是一个整数值。我们将基于这个整数值执行很长的计算操作。
3.  **启动一个在队列中执行这些任务的 Job**。该 Job 启动多个 Pod。每个 Pod 从消息队列中取走一个任务，处理它，然后重复执行，直到队列的队尾。

### 准备开始

要熟悉 Job 基本用法（非并行的），请参考 [Job](https://v1-14.docs.kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/)。

你必须拥有一个 Kubernetes 的集群，同时你的 Kubernetes 集群必须带有 kubectl 命令行工具。 如果你还没有集群，你可以通过 [Minikube](https://v1-14.docs.kubernetes.io/docs/getting-started-guides/minikube) 构建一 个你自己的集群，或者你可以使用下面任意一个 Kubernetes 工具构建：

-   [Katacoda](https://www.katacoda.com/courses/kubernetes/playground)
-   [Play with Kubernetes](http://labs.play-with-k8s.com/) –>

To check the version, enter `kubectl version`.

### 启动消息队列服务

本例使用了 RabbitMQ，使用其他 AMQP 类型的消息服务应该比较容易。

在实际工作中，在集群中一次性部署某个消息队列服务，之后在很多 Job 中复用，包括需要长期运行的服务。

按下面的方法启动 RabbitMQ：

```shell
$ kubectl create -f examples/celery-rabbitmq/rabbitmq-service.yaml
service "rabbitmq-service" created
$ kubectl create -f examples/celery-rabbitmq/rabbitmq-controller.yaml
replicationcontroller "rabbitmq-controller" created
```

我们仅用到 [celery-rabbitmq 示例](https://github.com/kubernetes/kubernetes/tree/release-1.3/examples/celery-rabbitmq) 中描述的部分功能。

### 测试消息队列服务

现在，我们可以试着访问消息队列。我们将会创建一个临时的可交互的 Pod，在它上面安装一些工具，然后用队列做实验。

首先创建一个临时的可交互的 Pod：

```shell
# 创建一个临时的可交互的 Pod
$ kubectl run -i --tty temp --image ubuntu:14.04
Waiting for pod default/temp-loe07 to be running, status is Pending, pod ready: false
... [ previous line repeats several times .. hit return when it stops ] ...
```

请注意你的 Pod 名称和命令提示符将会不同。

接下来安装 `amqp-tools` ，这样我们就能用消息队列了。

```shell
# 安装一些工具
root@temp-loe07:/# apt-get update
.... [ lots of output ] ....
root@temp-loe07:/# apt-get install -y curl ca-certificates amqp-tools python dnsutils
.... [ lots of output ] ....
```

后续，我们将制作一个包含这些包的 Docker 镜像。

接着，我们将要验证我们发现 RabbitMQ 服务：

```
# 请注意 rabbitmq-service 有Kubernetes 提供的 DNS 名称，

root@temp-loe07:/# nslookup rabbitmq-service
Server:        10.0.0.10
Address:    10.0.0.10#53

Name:    rabbitmq-service.default.svc.cluster.local
Address: 10.0.147.152

# 你的 IP 地址将会发生变化。
```

如果 Kube-DNS 没有正确安装，上一步可能会出错。 你也可以在环境变量中找到服务 IP。

```
# env | grep RABBIT | grep HOST
RABBITMQ_SERVICE_SERVICE_HOST=10.0.147.152

# 你的 IP 地址将会发生变化。
```

接着我们将要确认可以创建队列，并能发布消息和消费消息。

```shell
# 下一行，rabbitmq-service 是访问 rabbitmq-service 的主机名。5672是 rabbitmq 的标准端口。

root@temp-loe07:/# export BROKER_URL=amqp://guest:guest@rabbitmq-service:5672

# 如果上一步中你不能解析 "rabbitmq-service"，可以用下面的命令替换：
# root@temp-loe07:/# BROKER_URL=amqp://guest:guest@$RABBITMQ_SERVICE_SERVICE_HOST:5672

# 现在创建队列：

root@temp-loe07:/# /usr/bin/amqp-declare-queue --url=$BROKER_URL -q foo -d foo

# 向它推送一条消息:

root@temp-loe07:/# /usr/bin/amqp-publish --url=$BROKER_URL -r foo -p -b Hello

# 然后取回它.

root@temp-loe07:/# /usr/bin/amqp-consume --url=$BROKER_URL -q foo -c 1 cat && echo
Hello
root@temp-loe07:/#
```

最后一个命令中， `amqp-consume` 工具从队列中取走了一个消息，并把该消息传递给了随机命令的标准输出。在这种情况下，`cat` 只会打印它从标准输入或得的内容，echo 只会添加回车符以便示例可读。

### 为队列增加任务

现在让我们给队列增加一些任务。在我们的示例中，任务是多个待打印的字符串。

实践中，消息的内容可以是：

-   待处理的文件名
-   程序额外的参数
-   数据库表的关键字范围
-   模拟任务的配置参数
-   待渲染的场景的帧序列号

本例中，如果有大量的数据需要被 Job 的所有 Pod 读取，典型的做法是把它们放在一个共享文件系统中，如NFS，并以只读的方式挂载到所有 Pod，或者 Pod 中的程序从类似 HDFS 的集群文件系统中读取。

例如，我们创建队列并使用 amqp 命令行工具向队列中填充消息。实践中，你可以写个程序来利用 amqp 客户端库来填充这些队列。

```shell
$ /usr/bin/amqp-declare-queue --url=$BROKER_URL -q job1  -d job1
$ for f in apple banana cherry date fig grape lemon melon 

do
  /usr/bin/amqp-publish --url=$BROKER_URL -r job1 -p -b $f
done
```

这样，我们给队列中填充了8个消息。

### 创建镜像

现在我们可以创建一个做为 Job 来运行的镜像。

我们将用 `amqp-consume` 来从队列中读取消息并实际运行我们的程序。这里给出一个非常简单的示例程序：

[`application/job/rabbitmq/worker.py`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/rabbitmq/worker.py)

```python
#!/usr/bin/env python

# Just prints standard out and sleeps for 10 seconds.
import sys
import time
print("Processing " + sys.stdin.lines())
time.sleep(10)
```

现在，编译镜像。如果你在用源代码树，那么切换到目录 `examples/job/work-queue-1`。否则的话，创建一个临时目录，切换到这个目录。下载 [Dockerfile](https://v1-14.docs.kubernetes.io/examples/application/job/rabbitmq/Dockerfile)，和 [worker.py](https://v1-14.docs.kubernetes.io/examples/application/job/rabbitmq/worker.py)。无论哪种情况，都可以用下面的命令编译镜像

```shell
$ docker build -t job-wq-1 .
```

对于 [Docker Hub](https://hub.docker.com/), 给你的应用镜像打上标签，标签为你的用户名，然后用下面的命令推送到 Hub。用你的 Hub 用户名替换 `<username>`。

```shell
docker tag job-wq-1 <username>/job-wq-1
docker push <username>/job-wq-1
```

如果你在用[谷歌容器仓库](https://cloud.google.com/tools/container-registry/)，用你的项目 ID 作为标签打到你的应用镜像上，然后推送到 GCR。用你的项目 ID 替换 `<project>`。

```shell
docker tag job-wq-1 gcr.io/<project>/job-wq-1
gcloud docker -- push gcr.io/<project>/job-wq-1
```

### 定义 Job

这里给出一个 Job 定义 yaml文件。你需要拷贝一份并编辑镜像以匹配你使用的名称，保存为 `./job.yaml`。

[`application/job/rabbitmq/job.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/rabbitmq/job.yaml)

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-1
spec:
  completions: 8
  parallelism: 2
  template:
    metadata:
      name: job-wq-1
    spec:
      containers:
      - name: c
        image: gcr.io/<project>/job-wq-1
        env:
        - name: BROKER_URL
          value: amqp://guest:guest@rabbitmq-service:5672
        - name: QUEUE
          value: job1
      restartPolicy: OnFailure
```

本例中，每个 Pod 使用队列中的一个消息然后退出。这样，Job 的完成计数就代表了完成的工作项的数量。本例中我们设置 `.spec.completions: 8`，因为我们放了8项内容在队列中。

### 运行 Job

现在我们运行 Job：

```shell
kubectl create -f ./job.yaml
```

稍等片刻，然后检查 Job。

```shell
$ kubectl describe jobs/job-wq-1
Name:             job-wq-1
Namespace:        default
Selector:         controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
Labels:           controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
                  job-name=job-wq-1
Annotations:      <none>
Parallelism:      2
Completions:      8
Start Time:       Wed, 06 Sep 2017 16:42:02 +0800
Pods Statuses:    0 Running / 8 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=41d75705-92df-11e7-b85e-fa163ee3c11f
                job-name=job-wq-1
  Containers:
   c:
    Image:      gcr.io/causal-jigsaw-637/job-wq-1
    Port:
    Environment:
      BROKER_URL:       amqp://guest:guest@rabbitmq-service:5672
      QUEUE:            job1
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen  LastSeen   Count    From    SubobjectPath    Type      Reason              Message
  ─────────  ────────   ─────    ────    ─────────────    ──────    ──────              ───────
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-hcobb
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-weytj
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-qaam5
  27s        27s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-b67sr
  26s        26s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-xe5hj
  15s        15s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-w2zqe
  14s        14s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-d6ppa
  14s        14s        1        {job }                   Normal    SuccessfulCreate    Created pod: job-wq-1-p17e0
```

我们所有的 Pod 都成功了。耶！

### 替代方案

本文所讲述的处理方法的好处是你不需要修改你的 “worker” 程序使其知道工作队列的存在。

本文所描述的方法需要你运行一个消息队列服务。如果不方便运行消息队列服务，你也许会考虑另外一种[任务模式](https://v1-14.docs.kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/#job-patterns)。

本文所述的方法为每个工作项创建了一个 Pod。如果你的工作项仅需数秒钟，为每个工作项创建 Pod会增加很多的常规消耗。可以考虑另外的方案请参考[示例](https://v1-14.docs.kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/)，这种方案可以实现每个 Pod 执行多个工作项。

示例中，我们使用 `amqp-consume` 从消息队列读取消息并执行我们真正的程序。这样的好处是你不需要修改你的程序使其知道队列的存在。要了解怎样使用客户端库和工作队列通信，请参考[不同的示例](https://v1-14.docs.kubernetes.io/docs/tasks/job/fine-parallel-processing-work-queue/)。

### 友情提醒

如果设置的完成数量小于队列中的消息数量，会导致一部分消息项不会被执行。

如果设置的完成数量大于队列中的消息数量，当队列中所有的消息都处理完成后，Job 也会显示为未完成。Job 将创建 Pod 并阻塞等待消息输入。

当发生下面两种情况时，即使队列中所有的消息都处理完了，Job 也不会显示为完成状态： * 在 amqp-consume 命令拿到消息和容器成功退出之间的时间段内，执行杀死容器操作； * 在 kubelet 向 api-server 传回 Pod 成功运行之前，发生节点崩溃。





## 使用工作队列进行精细并行处理

在这个例子中，我们会运行一个Kubernetes Job，其中的 Pod 会运行多个并行工作进程。

在这个例子中，当每个pod被创建时，它会从一个任务队列中获取一个工作单元，处理它，然后重复，直到到达队列的尾部。

下面是这个示例的步骤概述

1.  **启动存储服务用于保存工作队列。** 在这个例子中，我们使用 Redis 来存储工作项。在上一个例子中，我们使用了 RabbitMQ。在这个例子中，由于 AMQP 不能为客户端提供一个良好的方法来检测一个有限长度的工作队列是否为空，我们使用了 Redis 和一个自定义的工作队列客户端库。在实践中，您可能会设置一个类似于 Redis 的存储库，并将其同时用于多项任务或其他事务的工作队列。
3.  **创建一个队列，然后向其中填充消息。** 每个消息表示一个将要被处理的工作任务。在这个例子中，消息只是一个我们将用于进行长度计算的整数。
5.  **启动一个 Job 对队列中的任务进行处理**。这个 Job 启动了若干个 Pod 。每个 Pod 从消息队列中取出一个工作任务，处理它，然后重复，直到到达队列的尾部。

### 准备工作

一个集群。

### 启动 Redis

对于这个例子，为了简单起见，我们将启动一个单实例的 Redis。 了解如何部署一个可伸缩、高可用的 Redis 例子，请查看 [Redis 样例](https://github.com/kubernetes/examples/tree/master/guestbook)

如果您在使用本文档库的源代码目录，您可以进入如下目录，然后启动一个临时的 Pod 用于运行 Redis 和 一个临时的 service 以便我们能够找到这个 Pod

```shell
$ cd content/en/examples/application/job/redis
$ kubectl create -f ./redis-pod.yaml
pod/redis-master created
$ kubectl create -f ./redis-service.yaml
service/redis created
```

如果您没有使用本文档库的源代码目录，您可以直接下载如下文件：

-   [`redis-pod.yaml`](https://kubernetes.io/examples/application/job/redis/redis-pod.yaml)
-   [`redis-service.yaml`](https://kubernetes.io/examples/application/job/redis/redis-service.yaml)
-   [`Dockerfile`](https://kubernetes.io/examples/application/job/redis/Dockerfile)
-   [`job.yaml`](https://kubernetes.io/examples/application/job/redis/job.yaml)
-   [`rediswq.py`](https://kubernetes.io/examples/application/job/redis/rediswq.py)
-   [`worker.py`](https://kubernetes.io/examples/application/job/redis/worker.py)

### 使用任务填充队列

现在，让我们往队列里添加一些“任务”。在这个例子中，我们的任务只是一些将被打印出来的字符串。

启动一个临时的可交互的 pod 用于运行 Redis 命令行界面。

```shell
$ kubectl run -i --tty temp --image redis --command "/bin/sh"
Waiting for pod default/redis2-c7h78 to be running, status is Pending, pod ready: false
Hit enter for command prompt
```

现在按回车键，启动 redis 命令行界面，然后创建一个存在若干个工作项的列表。

```
# redis-cli -h redis
redis:6379> rpush job2 "apple"
(integer) 1
redis:6379> rpush job2 "banana"
(integer) 2
redis:6379> rpush job2 "cherry"
(integer) 3
redis:6379> rpush job2 "date"
(integer) 4
redis:6379> rpush job2 "fig"
(integer) 5
redis:6379> rpush job2 "grape"
(integer) 6
redis:6379> rpush job2 "lemon"
(integer) 7
redis:6379> rpush job2 "melon"
(integer) 8
redis:6379> rpush job2 "orange"
(integer) 9
redis:6379> lrange job2 0 -1
1) "apple"
2) "banana"
3) "cherry"
4) "date"
5) "fig"
6) "grape"
7) "lemon"
8) "melon"
9) "orange"
```

因此，这个键为 `job2` 的列表就是我们的工作队列。

注意：如果您还没有正确地配置 Kube DNS，您可能需要将上面的第一步改为 `redis-cli -h $REDIS_SERVICE_HOST`。

创建镜像

现在我们已经准备好创建一个我们要运行的镜像

我们会使用一个带有 redis 客户端的 python 工作程序从消息队列中读出消息。

这里提供了一个简单的 Redis 工作队列客户端库，叫 rediswq.py ([下载](https://kubernetes.io/examples/application/job/redis/rediswq.py))。

Job 中每个 Pod 内的 “工作程序” 使用工作队列客户端库获取工作。如下：

[`application/job/redis/worker.py`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/redis/worker.py)

```python
#!/usr/bin/env python

import time
import rediswq

host="redis"
# Uncomment next two lines if you do not have Kube-DNS working.
# import os
# host = os.getenv("REDIS_SERVICE_HOST")

q = rediswq.RedisWQ(name="job2", host="redis")
print("Worker with sessionID: " +  q.sessionID())
print("Initial queue state: empty=" + str(q.empty()))
while not q.empty():
  item = q.lease(lease_secs=10, block=True, timeout=2) 
  if item is not None:
    itemstr = item.decode("utf=8")
    print("Working on " + itemstr)
    time.sleep(10) # Put your actual work here instead of sleep.
    q.complete(item)
  else:
    print("Waiting for work")
print("Queue empty, exiting")
```

如果您在使用本文档库的源代码目录，请将当前目录切换到 `content/en/examples/application/job/redis/`。否则，请点击链接下载 [`worker.py`](https://kubernetes.io/examples/application/job/redis/worker.py)、 [`rediswq.py`](https://kubernetes.io/examples/application/job/redis/rediswq.py) 和 [`Dockerfile`](https://kubernetes.io/examples/application/job/redis/Dockerfile)。然后构建镜像：

```shell
docker build -t job-wq-2 .
```

### Push 镜像

对于 [Docker Hub](https://hub.docker.com/)，请先用您的用户名给镜像打上标签，然后使用下面的命令 push 您的镜像到仓库。请将 `<username>` 替换为您自己的用户名。

```shell
docker tag job-wq-2 <username>/job-wq-2
docker push <username>/job-wq-2
```

您需要将镜像 push 到一个公共仓库或者 [配置集群访问您的私有仓库](https://kubernetes.io/docs/concepts/containers/images/)。

如果您使用的是 [Google Container Registry](https://cloud.google.com/tools/container-registry/)，请先用您的 project ID 给您的镜像打上标签，然后 push 到 GCR 。请将 `<project>` 替换为您自己的 project ID

```shell
docker tag job-wq-2 gcr.io/<project>/job-wq-2
gcloud docker -- push gcr.io/<project>/job-wq-2
```

### 定义一个 Job

这是 job 定义：

[`application/job/redis/job.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/application/job/redis/job.yaml) 

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-wq-2
spec:
  parallelism: 2
  template:
    metadata:
      name: job-wq-2
    spec:
      containers:
      - name: c
        image: gcr.io/myproject/job-wq-2
      restartPolicy: OnFailure
```

请确保将 job 模板中的 `gcr.io/myproject` 更改为您自己的路径。

在这个例子中，每个 pod 处理了队列中的多个项目，直到队列中没有项目时便退出。因为是由工作程序自行检测工作队列是否为空，并且 Job 控制器不知道工作队列的存在，所以依赖于工作程序在完成工作时发出信号。工作程序以成功退出的形式发出信号表示工作队列已经为空。所以，只要有任意一个工作程序成功退出，控制器就知道工作已经完成了，所有的 Pod 将很快会退出。因此，我们将 Job 的 completion count 设置为 1 。尽管如此，Job 控制器还是会等待其它 Pod 完成。

### 运行 Job

现在运行这个 Job ：

```shell
kubectl create -f ./job.yaml
```

稍等片刻，然后检查这个 Job。

```shell
$ kubectl describe jobs/job-wq-2
Name:             job-wq-2
Namespace:        default
Selector:         controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
Labels:           controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
                  job-name=job-wq-2
Annotations:      <none>
Parallelism:      2
Completions:      <unset>
Start Time:       Mon, 11 Jan 2016 17:07:59 -0800
Pods Statuses:    1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       controller-uid=b1c7e4e3-92e1-11e7-b85e-fa163ee3c11f
                job-name=job-wq-2
  Containers:
   c:
    Image:              gcr.io/exampleproject/job-wq-2
    Port:
    Environment:        <none>
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From            SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----            -------------    --------    ------            -------
  33s          33s         1        {job-controller }                Normal      SuccessfulCreate  Created pod: job-wq-2-lglf8


$ kubectl logs pods/job-wq-2-7r7b2
Worker with sessionID: bbd72d0a-9e5c-4dd6-abf6-416cc267991f
Initial queue state: empty=False
Working on banana
Working on date
Working on lemon
```

您可以看到，其中的一个 pod 处理了若干个工作单元。

### 其它

如果您不方便运行一个队列服务或者修改您的容器用于运行一个工作队列，您可以考虑其它的 [job 模式](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/#job-patterns)。

如果您有连续的后台处理业务，那么可以考虑使用 `replicationController` 来运行您的后台业务，和运行一个类似 <https://github.com/resque/resque> 的后台处理库。

