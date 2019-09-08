# Federation

>   #### Deprecated
>
>   强烈建议不要使用`联合 v1 版本`，`联合 v1 版本`从未达到 GA 状态，且不再处于积极开发阶段。文档仅作为历史参考。
>
>   有关更多信息，请参阅预期的替代品 [Kubernetes 联合 v2 版本](https://github.com/kubernetes-sigs/federation-v2)。

## 使用Kubefed建立联合集群

Kubernetes version 1.5 and above includes a new command line tool called [`kubefed`](https://kubernetes.io/docs/admin/kubefed/) to help you administrate your federated clusters. `kubefed` helps you to deploy a new Kubernetes cluster federation control plane, and to add clusters to or remove clusters from an existing federation control plane.

This guide explains how to administer a Kubernetes Cluster Federation using `kubefed`.

>   Note: `kubefed` is a beta feature in Kubernetes 1.6.

### Before you begin

You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://kubernetes.io/docs/setup/minikube), or you can use one of these Kubernetes playgrounds:

-   [Katacoda](https://www.katacoda.com/courses/kubernetes/playground)
-   [Play with Kubernetes](http://labs.play-with-k8s.com/)

To check the version, enter `kubectl version`.

### Prerequisites

This guide assumes that you have a running Kubernetes cluster. Please see one of the [getting started](https://kubernetes.io/docs/setup/) guides for installation instructions for your platform.

### Getting `kubefed`

Download the client tarball corresponding to the particular release and extract the binaries in the tarball:

>   **Note:** Until Kubernetes version `1.8.x` the federation project was maintained as part of the [core kubernetes repo](https://github.com/kubernetes/kubernetes). Between Kubernetes releases `1.8` and `1.9`, the federation project moved into a separate [federation repo](https://github.com/kubernetes/federation), where it is now maintained. Consequently, the federation release information is available on the [release page](https://github.com/kubernetes/federation/releases).

#### For Kubernetes versions 1.8.x and earlier:

```shell
curl -LO https://storage.googleapis.com/kubernetes-release/release/${RELEASE-VERSION}/kubernetes-client-linux-amd64.tar.gz
tar -xzvf kubernetes-client-linux-amd64.tar.gz
```

>   **Note:** The `RELEASE-VERSION` variable should either be set to or replaced with the actual version needed.

Copy the extracted binary to one of the directories in your `$PATH` and set the executable permission on the binary.

```shell
sudo cp kubernetes/client/bin/kubefed /usr/local/bin
sudo chmod +x /usr/local/bin/kubefed
```

#### For Kubernetes versions 1.9.x and above:

```shell
curl -LO https://storage.cloud.google.com/kubernetes-federation-release/release/${RELEASE-VERSION}/federation-client-linux-amd64.tar.gz
tar -xzvf federation-client-linux-amd64.tar.gz
```

>   **Note:** The `RELEASE-VERSION` variable should be replaced with one of the release versions available at [federation release page](https://github.com/kubernetes/federation/releases).

Copy the extracted binary to one of the directories in your `$PATH` and set the executable permission on the binary.

```shell
sudo cp federation/client/bin/kubefed /usr/local/bin
sudo chmod +x /usr/local/bin/kubefed
```

#### Install kubectl

You can install a matching version of kubectl using the instructions on the [kubectl install page](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Choosing a host cluster.

You’ll need to choose one of your Kubernetes clusters to be the *host cluster*. The host cluster hosts the components that make up your federation control plane. Ensure that you have a `kubeconfig` entry in your local `kubeconfig` that corresponds to the host cluster. You can verify that you have the required `kubeconfig` entry by running:

```shell
kubectl config get-contexts
```

The output should contain an entry corresponding to your host cluster, similar to the following:

```
CURRENT   NAME                                          CLUSTER                                       AUTHINFO                                      NAMESPACE
*         gke_myproject_asia-east1-b_gce-asia-east1     gke_myproject_asia-east1-b_gce-asia-east1     gke_myproject_asia-east1-b_gce-asia-east1
```

You’ll need to provide the `kubeconfig` context (called name in the entry above) for your host cluster when you deploy your federation control plane.

### Deploying a federation control plane

To deploy a federation control plane on your host cluster, run [`kubefed init`](https://kubernetes.io/docs/admin/kubefed_init/) command. When you use `kubefed init`, you must provide the following:

-   Federation name
-   `--host-cluster-context`, the `kubeconfig` context for the host cluster
-   `--dns-provider`, one of `'google-clouddns'`, `aws-route53` or `coredns`
-   `--dns-zone-name`, a domain name suffix for your federated services

If your host cluster is running in a non-cloud environment or an environment that doesn’t support common cloud primitives such as load balancers, you might need additional flags. Please see the [on-premises host clusters](https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/#on-premises-host-clusters) section below.

The following example command deploys a federation control plane with the name `fellowship`, a host cluster context `rivendell`, and the domain suffix `example.com.`:

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com."
```

The domain suffix specified in `--dns-zone-name` must be an existing domain that you control, and that is programmable by your DNS provider. It must also end with a trailing dot.

Once the federation control plane is initialized, query the namespaces:

```shell
kubectl get namespace --context=fellowship
```

If you do not see the `default` namespace listed (this is due to a [bug](https://github.com/kubernetes/kubernetes/issues/33292)). Create it yourself with the following command:

```shell
kubectl create namespace default --context=fellowship
```

The machines in your host cluster must have the appropriate permissions to program the DNS service that you are using. For example, if your cluster is running on Google Compute Engine, you must enable the Google Cloud DNS API for your project.

The machines in Google Kubernetes Engine clusters are created without the Google Cloud DNS API scope by default. If you want to use a Google Kubernetes Engine cluster as a Federation host, you must create it using the `gcloud` command with the appropriate value in the `--scopes` field. You cannot modify a Google Kubernetes Engine cluster directly to add this scope, but you can create a new node pool for your cluster and delete the old one.

>   **Note:** This will cause pods in the cluster to be rescheduled.

To add the new node pool, run:

```shell
scopes="$(gcloud container node-pools describe --cluster=gke-cluster default-pool --format='value[delimiter=","](config.oauthScopes)')"
gcloud container node-pools create new-np \
    --cluster=gke-cluster \
    --scopes="${scopes},https://www.googleapis.com/auth/ndev.clouddns.readwrite"
```

To delete the old node pool, run:

```shell
gcloud container node-pools delete default-pool --cluster gke-cluster
```

`kubefed init` sets up the federation control plane in the host cluster and also adds an entry for the federation API server in your local kubeconfig.

>   **Note:**
>
>   In the beta release of Kubernetes 1.6, `kubefed init` does not automatically set the current context to the newly deployed federation. You can set the current context manually by running:
>
>   ```shell
>   kubectl config use-context fellowship
>   ```
>
>   where `fellowship` is the name of your federation.

#### Basic and token authentication support

`kubefed init` by default only generates TLS certificates and keys to authenticate with the federation API server and writes them to your local kubeconfig file. If you wish to enable basic authentication or token authentication for debugging purposes, you can enable them by passing the `--apiserver-enable-basic-auth` flag or the `--apiserver-enable-token-auth` flag.

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com." \
    --apiserver-enable-basic-auth=true \
    --apiserver-enable-token-auth=true
```

#### Passing command line arguments to federation components

`kubefed init` bootstraps a federation control plane with default arguments to federation API server and federation controller manager. Some of these arguments are derived from `kubefed init`’s flags. However, you can override these command line arguments by passing them via the appropriate override flags.

You can override the federation API server arguments by passing them to `--apiserver-arg-overrides` and override the federation controller manager arguments by passing them to `--controllermanager-arg-overrides`.

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com." \
    --apiserver-arg-overrides="--anonymous-auth=false,--v=4" \
    --controllermanager-arg-overrides="--controllers=services=false"
```

#### Configuring a DNS provider

The Federated service controller programs a DNS provider to expose federated services via DNS names. Certain cloud providers automatically provide the configuration required to program the DNS provider if the host cluster’s cloud provider is same as the DNS provider. In all other cases, you have to provide the DNS provider configuration to your federation controller manager which will in-turn be passed to the federated service controller. You can provide this configuration to federation controller manager by storing it in a file and passing the file’s local filesystem path to `kubefed init`’s `--dns-provider-config` flag. For example, save the config below in `$HOME/coredns-provider.conf`.

```ini
[Global]
etcd-endpoints = http://etcd-cluster.ns:2379
zones = example.com.
```

And then pass this file to `kubefed init`:

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="coredns" \
    --dns-zone-name="example.com." \
    --dns-provider-config="$HOME/coredns-provider.conf"
```

#### On-premises host clusters

##### API server service type

`kubefed init` exposes the federation API server as a Kubernetes [service](https://kubernetes.io/docs/concepts/services-networking/service/) on the host cluster. By default, this service is exposed as a [load balanced service](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer). Most on-premises and bare-metal environments, and some cloud environments lack support for load balanced services. `kubefed init` allows exposing the federation API server as a [`NodePort` service](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport)on such environments. This can be accomplished by passing the `--api-server-service-type=NodePort` flag. You can also specify the preferred address to advertise the federation API server by passing the `--api-server-advertise-address=<IP-address>` flag. Otherwise, one of the host cluster’s node address is chosen as the default.

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com." \
    --api-server-service-type="NodePort" \
    --api-server-advertise-address="10.0.10.20"
```

##### Provisioning storage for etcd

Federation control plane stores its state in [`etcd`](https://coreos.com/etcd/docs/latest/). [`etcd`](https://coreos.com/etcd/docs/latest/) data must be stored in a persistent storage volume to ensure correct operation across federation control plane restarts. On host clusters that support [dynamic provisioning of storage volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#dynamic), `kubefed init` dynamically provisions a [`PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes) and binds it to a [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) to store [`etcd`](https://coreos.com/etcd/docs/latest/)data. If your host cluster doesn’t support dynamic provisioning, you can also statically provision a [`PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes). `kubefed init` creates a [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) that has the following configuration:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.alpha.kubernetes.io/storage-class: "yes"
  labels:
    app: federated-cluster
  name: fellowship-federation-apiserver-etcd-claim
  namespace: federation-system
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

To statically provision a [`PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes), you must ensure that the [`PersistentVolume`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes)that you create has the matching storage class, access mode and at least as much capacity as the requested [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims).

Alternatively, you can disable persistent storage completely by passing `--etcd-persistent-storage=false` to `kubefed init`. However, we do not recommended this because your federation control plane cannot survive restarts in this mode.

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="google-clouddns" \
    --dns-zone-name="example.com." \
    --etcd-persistent-storage=false
```

`kubefed init` still doesn’t support attaching an existing [`PersistentVolumeClaim`](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) to the federation control plane that it bootstraps. We are planning to support this in a future version of `kubefed`.

##### CoreDNS support

Federated services now support [CoreDNS](https://coredns.io/) as one of the DNS providers. If you are running your clusters and federation in an environment that does not have access to cloud-based DNS providers, then you can run your own [CoreDNS](https://coredns.io/) instance and publish the federated service DNS names to that server.

You can configure your federation to use [CoreDNS](https://coredns.io/), by passing appropriate values to `kubefed init`’s `--dns-provider` and `--dns-provider-config` flags.

```shell
kubefed init fellowship \
    --host-cluster-context=rivendell \
    --dns-provider="coredns" \
    --dns-zone-name="example.com." \
    --dns-provider-config="$HOME/coredns-provider.conf"
```

For more information see [Setting up CoreDNS as DNS provider for Cluster Federation](https://kubernetes.io/docs/tasks/federation/set-up-coredns-provider-federation/).

##### AWS Route53 support

It is possible to utilize AWS Route53 as a cloud DNS provider when the federation controller-manager is run on-premise. The controller-manager Deployment must be configured with AWS credentials since it cannot implicitly gather them from a VM running on AWS.

Currently, `kubefed init` does not read AWS Route53 credentials from the `--dns-provider-config` flag, so a patch must be applied.

Specify AWS Route53 as your DNS provider when initializing your on-premise federation controller-manager by passing the flag `--dns-provider="aws-route53"` to `kubefed init`.

Create a patch file with your AWS credentials:

```yaml
spec:
  template:
    spec:
      containers:
      - name: controller-manager
        env:
        - name: AWS_ACCESS_KEY_ID
          value: "ABCDEFG1234567890"
        - name: AWS_SECRET_ACCESS_KEY
          value: "ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890"
```

Patch the Deployment:

```shell
kubectl -n federation-system patch deployment controller-manager --patch "$(cat <patch-file-name>.yml)"
```

Where `<patch-file-name>` is the name of the file you created above.

### Adding a cluster to a federation

After you’ve deployed a federation control plane, you’ll need to make that control plane aware of the clusters it should manage.

To join clusters into the federation:

1.  Change the context:

    ```shell
    kubectl config use-context fellowship
    ```

2.  If you are using a managed cluster service, allow the service to access the cluster. To do this, create a `clusterrolebinding` for the account associated with your cluster service:

    ```shell
    kubectl create clusterrolebinding <your_user>-cluster-admin-binding --clusterrole=cluster-admin --user=<your_user>@example.org --context=<joining_cluster_context>
    ```

3.  Join the cluster to the federation, using `kubefed join`, and make sure you provide the following:

    -   The name of the cluster that you are joining to the federation
    -   `--host-cluster-context`, the kubeconfig context for the host cluster

    For example, this command adds the cluster `gondor` to the federation running on host cluster `rivendell`:

    ```shell
    kubefed join gondor --host-cluster-context=rivendell
    ```

A new context has now been added to your kubeconfig named `fellowship` (after the name of your federation).

>   **Note:** The name that you provide to the `join` command is used as the joining cluster’s identity in federation. This name should adhere to the rules described in the [identifiers doc](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/). If the context corresponding to your joining cluster conforms to these rules, you can use the same name in the join command. Otherwise, you must choose a different name for your cluster’s identity.

#### Naming rules and customization

The cluster name you supply to `kubefed join` must be a valid [RFC 1035](https://www.ietf.org/rfc/rfc1035.txt) label and are enumerated in the [Identifiers doc](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/).

Furthermore, federation control plane requires credentials of the joined clusters to operate on them. These credentials are obtained from the local kubeconfig. `kubefed join` uses the cluster name specified as the argument to look for the cluster’s context in the local kubeconfig. If it fails to find a matching context, it exits with an error.

This might cause issues in cases where context names for each cluster in the federation don’t follow [RFC 1035](https://www.ietf.org/rfc/rfc1035.txt) label naming rules. In such cases, you can specify a cluster name that conforms to the [RFC 1035](https://www.ietf.org/rfc/rfc1035.txt) label naming rules and specify the cluster context using the `--cluster-context` flag. For example, if context of the cluster you are joining is `gondor_needs-no_king`, then you can join the cluster by running:

```shell
kubefed join gondor --host-cluster-context=rivendell --cluster-context=gondor_needs-no_king
```

##### Secret name

Cluster credentials required by the federation control plane as described above are stored as a secret in the host cluster. The name of the secret is also derived from the cluster name.

However, the name of a secret object in Kubernetes should conform to the DNS subdomain name specification described in [RFC 1123](https://tools.ietf.org/html/rfc1123). If this isn’t the case, you can pass the secret name to `kubefed join` using the `--secret-name` flag. For example, if the cluster name is `noldor` and the secret name is `11kingdom`, you can join the cluster by running:

```shell
kubefed join noldor --host-cluster-context=rivendell --secret-name=11kingdom
```

>   **Note:** If your cluster name does not conform to the DNS subdomain name specification, all you need to do is supply the secret name using the `--secret-name` flag. `kubefed join`automatically creates the secret for you.

#### `kube-dns` configuration

`kube-dns` configuration must be updated in each joining cluster to enable federated service discovery. If the joining Kubernetes cluster is version 1.5 or newer and your `kubefed` is version 1.6 or newer, then this configuration is automatically managed for you when the clusters are joined or unjoined using `kubefed join` or `unjoin` commands.

In all other cases, you must update `kube-dns` configuration manually as described in the[Updating KubeDNS section of the admin guide](https://kubernetes.io/docs/admin/federation/).

### Removing a cluster from a federation

To remove a cluster from a federation, run the [`kubefed unjoin`](https://kubernetes.io/docs/reference/setup-tools/kubefed/kubefed_unjoin/) command with the cluster name and the federation’s `--host-cluster-context`:

```shell
kubefed unjoin gondor --host-cluster-context=rivendell
```

### Turning down the federation control plane

Proper cleanup of federation control plane is not fully implemented in this beta release of `kubefed`. However, for the time being, deleting the federation system namespace should remove all the resources except the persistent storage volume dynamically provisioned for the federation control plane’s etcd. You can delete the federation namespace by running the following command:

```shell
kubectl delete ns federation-system --context=rivendell
```

>   **Note:** `rivendell` is the host cluster name. Replace that name with the appropriate name in your configuration.







## （未翻译）将CoreDNS设置为集群联邦的DNS提供者

本文展示了如何配置并部署 CoreDNS 作为联合集群的 DNS 提供商。

### 动机

-   配置并部署 CoreDNS 服务器
-   以 CoreDNS 作为 dns 提供商创建联邦
-   在名称服务器查找链中配置 CoreDNS 服务器

### 准备工作

-   您需要拥有一个运行的 Kubernetes 集群 （它被引用为 host 集群）。 请查看 [入门](https://kubernetes.io/docs/setup/)指导，根据您的平台选择其中一种安装说明。
-   为启用 `CoreDNS` 来实现跨联邦集群的服务发现，联邦的成员集群中必须支持 `LoadBalancer` 服务。

### 部署CoreDNS 和etcd charts

我们可以按照不同的配置部署 CoreDNS。 下面是一种参考，您可以对其进行调整，以适应平台和集群联邦的需求：

我们可以利用 helm charts 来部署 CoreDNS。 CoreDNS 部署时会以 [etcd](https://coreos.com/etcd) 作为后端，并且 etcd 应预先安装。 etcd 也可以利用 helm charts 进行部署。 下面展示了部署 etcd 的指导。

```
helm install --namespace my-namespace --name etcd-operator stable/etcd-operator
helm upgrade --namespace my-namespace --set cluster.enabled=true etcd-operator stable/etcd-operator
```

*注意：可以覆盖默认的部署配置，来适应 host 集群。*

部署成功后，可以在 host 集群内通过 [http://etcd-cluster.my-namespace:2379](http://etcd-cluster.my-namespace:2379/) 端点访问 etcd。

您需要定制 CoreDNS 默认配置，以适应联邦。 下面展示的是 Values.yaml，它覆盖了 CoreDNS chart 的默认配置参数。

```yaml
isClusterService: false
serviceType: "LoadBalancer"
plugins:
  kubernetes:
    enabled: false
  etcd:
    enabled: true
    zones:
    - "example.com."
    endpoint: "http://etcd-cluster.my-namespace:2379"
```

上面的配置文件需要一些说明：

-   `isClusterService` 指定 CoreDNS 是否以集群服务的形式部署（默认为是）。 您需要将其设置为 “false”，以使 CoreDNS 以 Kubernetes 应用服务的形式部署。
-   `serviceType` 指定为 CoreDNS 创建的 Kubernetes 服务类型。 您需要选择 “LoadBalancer” 或 “NodePort”，以使得 CoreDNS 服务能够从 Kubernetes 集群外部访问。
-   `middleware.kubernetes` 默认是启用的，通过将 `middleware.kubernetes.enabled` 设置为 “false” 来禁用 `middleware.kubernetes`。
-   通过将 `middleware.etcd.enabled` 设置为 “true” 来启用 `middleware.etcd`。
-   按照上面展示的，通过设置 `middleware.etcd.zones` 来配置 CoreDNS 被授权的 DNS 区域（联邦区域）。
-   通过设置 `middleware.etcd.endpoint` 来设置先前部署的 etcd 的端点。

现在通过运行以下命令来部署 CoreDNS：

```
helm install --namespace my-namespace --name coredns -f Values.yaml stable/coredns
```

验证 etcd 和 CoreDNS pod 按照预期在运行。

### 使用 CoreDNS 作为 DNS 提供商来部署联合集群

可以使用 `kubefed init` 来部署联邦控制平面。 可以通过指定两个附加参数来选择 CoreDNS 作为 DNS 提供商。

```
--dns-provider=coredns
--dns-provider-config=coredns-provider.conf
```

coredns-provider.conf 格式如下：

```
[Global]
etcd-endpoints = http://etcd-cluster.my-namespace:2379
zones = example.com.
coredns-endpoints = <coredns-server-ip>:<port>
```

-   `etcd-endpoints` 是访问 etcd 的端点。
-   `zones` 是 CoreDNS 被授权的联邦区域，其值与 `kubefed init` 的 –dns-zone-name 参数相同。
-   `coredns-endpoints` 是访问 CoreDNS 服务器的端点。 这是一个 1.7 版本开始引入的可选参数。

>   注意：CoreDNS 配置中的  `plugins.etcd.zones`  与  `kubefed init`  的  `--dns-zone-name`  参数应匹配。

### 在名称服务器 resolv.conf 链中设置 CoreDNS 服务器

>   注意：以下章节只适应于 1.7 以前的版本，且如上面章节所描述的，如果在 `coredns-provider.conf` 中配置了 `coredns-endpoints` 参数，会自动将 CoreDNS 服务器配置到名称服务器 resolv.conf 链中。

由于这种自主的 CoreDNS 服务器不能够公开发现，一旦联邦控制平面部署完成，且要联合的集群加入了联邦， 您需要将 CoreDNS 服务器加入到所有联邦集群的 pod 名称服务器 resolv.conf 链中。 这可以通过将下面的内容加入到 `kube-dns` deployment 中的 `dnsmasq` 容器参数来实现。

```
--server=/example.com./<CoreDNS endpoint>
```

用联邦的域替换上面的 `example.com` 。

现在联邦集群已经准备好跨集群服务发现了！







## 在联合集群建立放置决策

>   **Note:** `Federation V1`， 是当前的 Kubernetes 联合 API， 它“原样”重用 Kubernetes API 资源， 其许多特性目前被认为是 alpha。 没有明确的途径将 API 发展成 GA； 然而， 除了 Kubernetes API 之外， 还有一个 `Federation V2` 正在努力实现专用的联合 API。详细信息可在 [sig-multicluster 社区页面](https://github.com/kubernetes/community/tree/master/sig-multicluster) 获得。

此页面显示如何使用外部策略引擎对联合资源强制执行基于策略的放置决策。

### 准备开始

您需要一个正在运行的 Kubernetes 集群(它被引用为主机集群)。有关您的平台的安装说明，请参阅[入门](https://v1-14.docs.kubernetes.io/docs/setup/)指南。

### 部署联合集群并配置外部策略引擎

可以使用 `kubefed init` 部署联合控制面板。

部署联合控制面板之后，必须在联合 API 服务器中配置一个准入控制器，该控制器强制执行从外部策略引擎接收到的放置决策。

```
kubectl create -f scheduling-policy-admission.yaml
```

下图是准入控制器的 ConfigMap 示例：

[`federation/scheduling-policy-admission.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/federation/scheduling-policy-admission.yaml)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: admission
  namespace: federation-system
data:
  config.yml: |
    apiVersion: apiserver.k8s.io/v1alpha1
    kind: AdmissionConfiguration
    plugins:
    - name: SchedulingPolicy
      path: /etc/kubernetes/admission/scheduling-policy-config.yml
  scheduling-policy-config.yml: |
    kubeconfig: /etc/kubernetes/admission/opa-kubeconfig
  opa-kubeconfig: |
    clusters:
      - name: opa-api
        cluster:
          server: http://opa.federation-system.svc.cluster.local:8181/v0/data/kubernetes/placement
    users:
      - name: scheduling-policy
        user:
          token: deadbeefsecret
    contexts:
      - name: default
        context:
          cluster: opa-api
          user: scheduling-policy
    current-context: default
```

ConfigMap 包含三个文件：

-   `config.yml` 指定 `调度策略` 准入控制器配置文件的位置。
-   `scheduling-policy-config.yml` 指定与外部策略引擎联系所需的 kubeconfig 文件的位置。 该文件还可以包含一个 `retryBackoff` 值，该值以毫秒为单位控制初始重试 backoff 延迟。
-   `opa-kubeconfig` 是一个标准的 kubeconfig，包含联系外部策略引擎所需的 URL 和凭证。

编辑联合 API 服务器部署以启用 `SchedulingPolicy` 准入控制器。

```
kubectl -n federation-system edit deployment federation-apiserver
```

更新 Federation API 服务器命令行参数以启用准入控制器， 并将 ConfigMap 挂载到容器中。如果存在现有的 `-enable-admissionplugins` 参数，则追加 `SchedulingPolicy` 而不是添加另一行。

```
--enable-admission-plugins=SchedulingPolicy
--admission-control-config-file=/etc/kubernetes/admission/config.yml
```

将以下卷添加到联合 API 服务器 pod：

```
- name: admission-config
  configMap:
    name: admission
```

添加以下卷挂载联合 API 服务器的 `apiserver` 容器：

```
volumeMounts:
- name: admission-config
  mountPath: /etc/kubernetes/admission
```

### 部署外部策略引擎

[Open Policy Agent (OPA)](http://openpolicyagent.org/) 是一个开源的通用策略引擎， 您可以使用它在联合控制面板中执行基于策略的放置决策。

在主机群集中创建服务以联系外部策略引擎：

```
kubectl create -f policy-engine-service.yaml
```

下面显示的是 OPA 的示例服务。

[`federation/policy-engine-service.yaml` ](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/federation/policy-engine-service.yaml)

```yaml
kind: Service
apiVersion: v1
metadata:
  name: opa
  namespace: federation-system
spec:
  selector:
    app: opa
  ports:
  - name: http
    protocol: TCP
    port: 8181
    targetPort: 8181
```

使用联合控制面板在主机群集中创建部署：

```
kubectl create -f policy-engine-deployment.yaml
```

下面显示的是 OPA 的部署示例。

[`federation/policy-engine-deployment.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/federation/policy-engine-deployment.yaml)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: opa
  name: opa
  namespace: federation-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: opa
  template:
    metadata:
      labels:
        app: opa
      name: opa
    spec:
      containers:
        - name: opa
          image: openpolicyagent/opa:0.4.10
          args:
          - "run"
          - "--server"
        - name: kube-mgmt
          image: openpolicyagent/kube-mgmt:0.2
          args:
          - "-kubeconfig=/srv/kubernetes/kubeconfig"
          - "-cluster=federation/v1beta1/clusters"
          volumeMounts:
           - name: federation-kubeconfig
             mountPath: /srv/kubernetes
             readOnly: true
      volumes:
      - name: federation-kubeconfig
        secret:
          secretName: federation-controller-manager-kubeconfig
```

### 通过 ConfigMaps 配置放置策略

外部策略引擎将发现在 Federation API 服务器的 `kube-federation-scheduling-policy` 命名空间中创建的放置策略。

如果命名空间尚不存在，请创建它：

```
kubectl --context=federation create namespace kube-federation-scheduling-policy
```

配置一个示例策略来测试外部策略引擎:

[`policy.rego docs/tasks/federation`](https://github.com/kubernetes/website/blob/master/content/zh/docs/tasks/federation/policy.rego)

```rego
# OPA supports a high-level declarative language named Rego for authoring and
# enforcing policies. For more information on Rego, visit
# http://openpolicyagent.org.

# Rego policies are namespaced by the "package" directive.
package kubernetes.placement

# Imports provide aliases for data inside the policy engine. In this case, the
# policy simply refers to "clusters" below.
import data.kubernetes.clusters

# The "annotations" rule generates a JSON object containing the key
# "federation.kubernetes.io/replica-set-preferences" mapped to <preferences>.
# The preferences values is generated dynamically by OPA when it evaluates the
# rule.
#
# The SchedulingPolicy Admission Controller running inside the Federation API
# server will merge these annotations into incoming Federated resources. By
# setting replica-set-preferences, we can control the placement of Federated
# ReplicaSets.
#
# Rules are defined to generate JSON values (booleans, strings, objects, etc.)
# When OPA evaluates a rule, it generates a value IF all of the expressions in
# the body evaluate successfully. All rules can be understood intuitively as
# <head> if <body> where <body> is true if <expr-1> AND <expr-2> AND ...
# <expr-N> is true (for some set of data.)
annotations["federation.kubernetes.io/replica-set-preferences"] = preferences {
    input.kind = "ReplicaSet"
    value = {"clusters": cluster_map, "rebalance": true}
    json.marshal(value, preferences)
}

# This "annotations" rule generates a value for the "federation.alpha.kubernetes.io/cluster-selector"
# annotation.
#
# In English, the policy asserts that resources in the "production" namespace
# that are not annotated with "criticality=low" MUST be placed on clusters
# labelled with "on-premises=true".
annotations["federation.alpha.kubernetes.io/cluster-selector"] = selector {
    input.metadata.namespace = "production"
    not input.metadata.annotations.criticality = "low"
    json.marshal([{
        "operator": "=",
        "key": "on-premises",
        "values": "[true]",
    }], selector)
}

# Generates a set of cluster names that satisfy the incoming Federated
# ReplicaSet's requirements. In this case, just PCI compliance.
replica_set_clusters[cluster_name] {
    clusters[cluster_name]
    not insufficient_pci[cluster_name]
}

# Generates a set of clusters that must not be used for Federated ReplicaSets
# that request PCI compliance.
insufficient_pci[cluster_name] {
    clusters[cluster_name]
    input.metadata.annotations["requires-pci"] = "true"
    not pci_clusters[cluster_name]
}

# Generates a set of clusters that are PCI certified. In this case, we assume
# clusters are annotated to indicate if they have passed PCI compliance audits.
pci_clusters[cluster_name] {
    clusters[cluster_name].metadata.annotations["pci-certified"] = "true"
}

# Helper rule to generate a mapping of desired clusters to weights. In this
# case, weights are static.
cluster_map[cluster_name] = {"weight": 1} {
    replica_set_clusters[cluster_name]
}
```

下面显示的是创建示例策略的命令：

```shell
kubectl --context=federation -n kube-federation-scheduling-policy create configmap scheduling-policy --from-file=policy.rego
```

这个示例策略说明了一些关键思想：

-   放置策略可以引用联合资源中的任何字段。
-   放置策略可以利用外部上下文(例如，集群元数据)来做出决策。
-   管理策略可以集中管理。
-   策略可以定义简单的接口(例如 `requirements -pci` 注解)，以避免在清单中重复逻辑。

### 测试放置政策

注释其中一个集群以表明它是经过 PCI 认证的。

```
kubectl --context=federation annotate clusters cluster-name-1 pci-certified=true
```

部署联合副本来测试放置策略。

[`federation/replicaset-example-policy.yaml`](https://raw.githubusercontent.com/kubernetes/website/master/content/zh/examples/federation/replicaset-example-policy.yaml)

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  labels:
    app: nginx-pci
  name: nginx-pci
  annotations:
    requires-pci: "true"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pci
  template:
    metadata:
      labels:
        app: nginx-pci
    spec:
      containers:
      - image: nginx
        name: nginx-pci
```

下面显示的命令用于部署与策略匹配的副本集。

```shell
kubectl --context=federation create -f replicaset-example-policy.yaml
```

检查副本集以确认已应用适当的注解：

```shell
kubectl --context=federation get rs nginx-pci -o jsonpath='{.metadata.annotations}'
```





## 使用联合集群服务进行跨集群服务发现

本文介绍了如何使用 Kubernetes 联邦服务跨多个 Kubernetes 集群部署通用服务。 这使您可以轻松地为 Kubernetes 应用程序实现跨集群服务发现和可用区容错。

联邦服务的创建方式和传统[Kubernetes Services](https://kubernetes.io/docs/concepts/services-networking/service/)非常相似， 只需进行一次 API 调用即可指定所需的服务属性。在联邦服务环境下，此API调用将定向到联邦API端点， 而不是 Kubernetes 集群 API 端点，联邦服务的API与传统 Kubernetes Services 的API 100％兼容

一旦创建，联邦服务自动:

1.  在集群联邦底层的每个集群中创建对应的 Kubernetes 服务，
2.  监视那些服务 “shards” (以及它们所在的集群)的健康状况，
3.  在公共 DNS 提供商(例如 Google Cloud DNS 或 AWS Route 53 )中管理一组 DNS 记录， 从而确保您的联邦服务的客户端可以随时无缝地找到合适的健康服务端点,即使在集群可用区或区域中断的情况下也是如此。

在集群联邦内部的客户端(即 Pod )，如果集群联邦存在且健康，会自动在所在集群中找到集群联邦的本地分片， 如果不存在，将会寻找最接近健康的分片。

### 准备工作

一个集群

### 概览

本指南假定您有一个已安装运行的 Kubernetes 集群联邦，如果没有，请转到[集群联邦管理指南](https://kubernetes.io/docs/admin/federation/) 以了解如何启动一个集群联邦(或让您的集群管理员安装一个)。其他教程， 例如 Kelsey Hightower 的[这个教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)也可以帮助您。

一般来说，你应对[ Kubernetes 的工作原理](https://kubernetes.io/docs/tutorials/kubernetes-basics/)有一个基本了解，特别是[服务](https://kubernetes.io/docs/concepts/services-networking/service/)。

### 混合云功能

Kubernetes 集群联邦可以包含运行在不同云提供商(例如 Google Cloud ，AWS )中的集群,以及本地私有云(例如在 OpenStack 上)。 只需在适当的云提供商 和/或 地区创建您需要的所有集群，并且向联邦 API Server 注册每个集群的API端点和凭证(参考[集群联邦管理指南](https://kubernetes.io/docs/admin/federation/)了解详细信息)。

之后，您的应用程序和服务可以跨越不同的集群和云提供商，详情如下所述。

### 创建联邦服务

通常这样完成，例如:

```shell
kubectl --context=federation-cluster create -f services/nginx.yaml
```

’–context=federation-cluster’参数告诉 kubectl 使用合适的凭证将请求提交给联邦 API 端点。 如果你还没有配置这样的上下文，访问[集群联邦管理指南](https://kubernetes.io/docs/admin/federation/)或[管理教程](https://github.com/kelseyhightower/kubernetes-cluster-federation) 其中之一了解如何做到这一点。

如上所述，联邦服务将自动在联邦底层的所有集群中创建和维护对应的 Kubernetes 服务。

您可以通过检查每个基础集群来验证这一点，例如:

```shell
kubectl --context=gce-asia-east1a get services nginx
NAME     TYPE        CLUSTER-IP     EXTERNAL-IP      PORT(S)   AGE
nginx    ClusterIP   10.63.250.98   104.199.136.89   80/TCP    9m
```

上面假定您的客户端中为您的集群在该区域中配置了一个名为’gce-asia-east1a’的上下文, 底层服务的名称和命名空间将自动匹配上面创建的联邦服务的名称和命名空间( 如果碰巧在这些集群中有已经存在的相同名称和命名空间的服务， 它们将被联邦自动采用并更新以符合您联邦服务的规范 - 无论哪种方式，最终结果都将一样)。

联邦服务的状态将自动反映底层 Kubernetes 服务的实时状态，例如:

```shell
kubectl --context=federation-cluster describe services nginx
Name:                   nginx
Namespace:              default
Labels:                 run=nginx
Annotations:            <none>
Selector:               run=nginx
Type:                   LoadBalancer
IP:                     10.63.250.98
LoadBalancer Ingress:   104.197.246.190, 130.211.57.243, 104.196.14.231, 104.199.136.89, ...
Port:                   http    80/TCP
Endpoints:              <none>
Session Affinity:       None
Events:                 <none>
```

>   **注意:** 联邦服务的’LoadBalancer Ingress’地址与所有基础 Kubernetes 服务的’LoadBalancer Ingress’地址相对应( 一旦这些被分配 - 这可能需要几秒钟)。为了集群间和提供商间服务分片的网络正常工作, 您的服务需要有一个外部可见的IP地址。[Service Type:Loadbalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) 通常用于此, 尽管也有其他选项(例如 [External IP’s](https://kubernetes.io/docs/concepts/services-networking/service/#external-ips))存在。

还要注意，我们还没有配置任何后端 Pod 来接收指向这些地址的网络流量(即’Service Endpoints’)， 所以联邦服务还没有认为这些服务是健康的服务分片，因此还没有将这些服务添加到这个联邦服务的 DNS 记录中(后面更多关于这方面的内容)。

### 添加后端pod

为了使底层服务分片健康，我们需要在服务后面添加后端 Pod 。 这目前是直接针对底层集群的API端点完成的(尽管将来联邦服务器将能够使用单个命令为您完成所有这些工作，以节省您的麻烦)。 例如，要在13个基础集群中创建后端 Pod ：

```shell
for CLUSTER in asia-east1-c asia-east1-a asia-east1-b \
                        europe-west1-d europe-west1-c europe-west1-b \
                        us-central1-f us-central1-a us-central1-b us-central1-c \
                        us-east1-d us-east1-c us-east1-b
do
  kubectl --context=$CLUSTER run nginx --image=nginx:1.11.1-alpine --port=80
done
```

请注意，`kubectl run`会自动添加`run = nginx`标签，从而将后端Pod与对应的service相关联。

### 验证公共DNS记录

一旦上面的 Pod 成功启动，并开始监听连接，Kubernetes 将报告 Pod 作为在该集群服务的健康端点(通过自动健康检查)。 集群联邦将依次考虑这些服务“分片”中的每一个都是健康的，并通过自动配置相应的公共 DNS 记录以将其放置在服务中。 您可以使用您的首选接口到您配置的 DNS 提供商来验证这一点。例如，如果您的联邦配置为使用Google Cloud DNS， 并且托管 DNS 域名为’example.com’:

```shell
gcloud dns managed-zones describe example-dot-com
creationTime: '2016-06-26T18:18:39.229Z'
description: Example domain for Kubernetes Cluster Federation
dnsName: example.com.
id: '3229332181334243121'
kind: dns#managedZone
name: example-dot-com
nameServers:
- ns-cloud-a1.googledomains.com.
- ns-cloud-a2.googledomains.com.
- ns-cloud-a3.googledomains.com.
- ns-cloud-a4.googledomains.com.
gcloud dns record-sets list --zone example-dot-com
NAME                                                            TYPE      TTL     DATA
example.com.                                                    NS        21600   ns-cloud-e1.googledomains.com., ns-cloud-e2.googledomains.com.
example.com.                                                    OA        21600   ns-cloud-e1.googledomains.com. cloud-dns-hostmaster.google.com. 1 21600 3600 1209600 300
nginx.mynamespace.myfederation.svc.example.com.                 A         180     104.197.246.190, 130.211.57.243, 104.196.14.231, 104.199.136.89,...
nginx.mynamespace.myfederation.svc.us-central1-a.example.com.   A         180     104.197.247.191
nginx.mynamespace.myfederation.svc.us-central1-b.example.com.   A         180     104.197.244.180
nginx.mynamespace.myfederation.svc.us-central1-c.example.com.   A         180     104.197.245.170
nginx.mynamespace.myfederation.svc.us-central1-f.example.com.   CNAME     180     nginx.mynamespace.myfederation.svc.us-central1.example.com.
nginx.mynamespace.myfederation.svc.us-central1.example.com.     A         180     104.197.247.191, 104.197.244.180, 104.197.245.170
nginx.mynamespace.myfederation.svc.asia-east1-a.example.com.    A         180     130.211.57.243
nginx.mynamespace.myfederation.svc.asia-east1-b.example.com.    CNAME     180     nginx.mynamespace.myfederation.svc.asia-east1.example.com.
nginx.mynamespace.myfederation.svc.asia-east1-c.example.com.    A         180     130.211.56.221
nginx.mynamespace.myfederation.svc.asia-east1.example.com.      A         180     130.211.57.243, 130.211.56.221
nginx.mynamespace.myfederation.svc.europe-west1.example.com.    CNAME     180     nginx.mynamespace.myfederation.svc.example.com.
nginx.mynamespace.myfederation.svc.europe-west1-d.example.com.  CNAME     180     nginx.mynamespace.myfederation.svc.europe-west1.example.com.
... etc.
```

>   **注意：**
>
>   如果您的联邦配置为使用 AWS Route53，则可以使用其中一个等效的AWS工具，例如：
>
>   ```shell
>   aws route53 list-hosted-zones
>   ```
>
>   或
>
>   ```shell
>   aws route53 list-resource-record-sets --hosted-zone-id Z3ECL0L9QLOVBX
>   ```

无论您使用哪种 DNS 服务程序,任何 DNS 查询工具(例如’dig’或’nslookup’)允许您查看联邦为您创建的记录。 请注意，您应该直接在您的 DNS 提供商处指出这些工具(例如`dig @ns-cloud-e1.googledomains.com ...`), 或者根据您配置的 TTL 预计延迟(默认为180秒)看到更新,由于中间 DNS 服务器的缓存.

#### 关于上面例子的一些说明

1.  请注意，每个至少有一个健康后端端点的服务分片有一个正常(‘A’)记录。 例如，在 us-central1-a 中，104.197.247.191是该区域中服务分片的外部IP地址， 而在 asia-east1-a 中，地址是130.211.56.221。
2.  同样，还有区域’A’记录，包括该地区所有健康的分片。 例如，’us-central1’。这些区域记录对于不具有特定区域首选项的客户端以及下述局部自动化和故障转移机制的构建块非常有用。
3.  对于当前没有健康后端端点的区域，使用CNAME(‘Canonical Name’)记录将这些查询别名(自动重定向)到下一个最接近的健康区域。 在本例中，us-central1-f 中的服务分片当前没有健康的后端端点(即 Pod )， 因此已创建CNAME记录自动将查询重定向到该区域中的其他分片(本例中为 us-central1 )。
4.  同样，如果封闭区域内没有健康的碎片， 搜索就会进一步扩展。在 europe-west1-d 可用性区域，没有健康的后端， 所以查询被重定向到更广泛的 europe-west1 (其也没有健康的后端Pod)， 并且继续到全局健康的地址(‘nginx.mynamespace .myfederation.svc.example.com。’)。

上述 DNS 记录集自动保持与联邦服务系统全局所有服务分片的当前健康状态同步。 DNS 解析器库(由所有客户端调用)自动遍历 ‘CNAME’ 和 ‘A’ 记录的层次结构， 以返回正确的健康 IP 地址集。然后， 客户端可以选择任何一个返回的地址来启动网络连接(如果需要，可以自动切换到其他等效地址之一)。

### 发现联邦服务

#### From pods inside your federated clusters

#### 从联邦集群内部的 pod

默认情况下，Kubernetes 集群预先配置了集群本地DNS服务器(‘KubeDNS’),以及智能构建的DNS搜索路径， 这些路径一起确保您在 Pod 内运行的软件发出的“myservice”，“myservice.mynamespace”，“bobsservice.othernamespace”等DNS查询会自动扩展并正确解析到在本地集群中运行的服务相应的服务IP。

随着联邦服务和跨集群服务发现的引入，此概念将扩展到涵盖在全局范围内跨集群联邦的任何其他集群中运行的 Kubernetes 服务。 要充分利用此扩展范围，请使用稍微不同的 DNS 名称(形式`"<服务名称>.<命名空间>.<联邦名称>"`， 例如 `myservice.mynamespace.myfederation` )来解析联邦服务。 你没有明确地选择这种行为时使用不同的 DNS 名称还可避免现有应用程序意外地穿越跨区域或跨区域网络， 并且您因此可能会收到不必要的网络费用或延迟。

因此，使用我们的NGINX演示上面的服务，以及刚刚描述的联邦服务DNS名称表单，我们来看一个例子： `us-central1-f`可用区中的集群中的Pod需要连接我们的 NGINX 服务。现在可以使用服务的联邦 DNS 名称 `"nginx.mynamespace.myfederation"` 而不是使用服务在传统集群本地的 DNS (`"nginx.mynamespace"`, 将自动扩展到`"nginx.mynamespace.svc.cluster.local"`)名称，这将自动扩展，并解析到我的 NGINX 服务的最接近健康的分片，无论在世界任何地方。
如果在本地集群中存在健康的分片，该服务的集群本地(通常 10.x.y.z)IP地址将被返回(由集群本地 KubeDNS )，
这几乎完全等同于非联邦服务解决方案(接近因为 KubeDNS 实际上为本地联邦服务返回一个 CNAME 和一个 A 记录，但是应用程序会忽略这个小的技术差异)。

但是如果这个服务在本地集群中不存在的话(或者服务存在,但是没有健康的后端 pod )，那么 DNS 查询会自动扩展为`"nginx.mynamespace.myfederation.svc.us-central1-f.example.com”`(即逻辑上"找到离我的可用区域最近的其中一个分片的外部 IP "),这个扩展是由 KubeDNS 自动执行的，它返回相关的 CNAME 记录。
在上面的例子中，这导致了自动遍历 DNS 记录的层次结构，并且在本地 us-central1 区域的联邦服务的外部IP之一处结束(即 104.197.247.191,104.197.244.180 or 104.197.245.170)。 

当然，可以通过明确指定适当的DNS名称，而不是依靠DNS的自动扩展，
明确地定位可用区域和Pod以外的区域中的服务分片，而不是依靠自动 DNS 扩展。例如，
"nginx.mynamespace.myfederation.svc.europe-west1.example.com" 将解析欧洲所有目前健康的服务分片，
即使发布查询的 Pod 位于美国，也不管在美国是否有健康的服务碎片。这对于远程监控和其他类似的应用是有用的。

#### 来自集群联邦之外的其他客户端

上述讨论大部分同样适用于外部客户端，只是所述的自动 DNS 扩展已不再可用。
因此，外部客户端需要指定联邦服务的完全限定的 DNS 名称之一，是区域名称，
地区名称或全球名称。出于方便的原因，在服务中手动配置其他静态 CNAME 记录通常是一个好主意，例如：

```shell
eu.nginx.acme.com        CNAME nginx.mynamespace.myfederation.svc.europe-west1.example.com.
us.nginx.acme.com        CNAME nginx.mynamespace.myfederation.svc.us-central1.example.com.
nginx.acme.com           CNAME nginx.mynamespace.myfederation.svc.example.com.
```

这样，您的客户就可以使用左侧的简短表格，并且可以自动将其路由到本国最近的健康分片。 Kubernetes 集群联邦自动处理所有必需的故障转移。未来的版本将进一步改善。

### 处理后端 pod 和整个集群的失败

标准 Kubernetes 服务 cluster-IP 已经确定无响应的单独Pod端点自动以低延迟(几秒钟)退出服务。 此外，如上所述，Kubernetes 集群联邦系统会自动监控集群的健康状况， 以及您的联邦服务的所有分片背后的端点，根据需要采取分片进入和退出服务(例如当服务后面的所有端点， 或者整个集群或可用区域都停止运行时，或者相反，从停机状态恢复)。由于 DNS 缓存固有的延迟( 默认情况下，联邦服务 DNS 记录的缓存超时或 TTL 配置为3分钟，但可以进行调整)。 在灾难性失败的情况下，所有客户可能需要花费很长时间才能完全故障切换到另一个集群。 但是，考虑到每个区域服务端点可以返回的离散IP地址的数量(例如上面 us-central1，有三个选择) 许多客户端将自动故障切换到其中一个替代IP的时间少于给定的适当配置。

### 故障排查

#### 我无法连接到我的集群联邦 API

检查你的:

1.  客户端(通常为 kubectl )配置正确(包括 API 端点和登录证书)。
2.  集群联邦 API 服务器正在运行并且网络可达。

请参阅[集群联邦管理指南](https://kubernetes.io/docs/admin/federation/)了解如何正确启动集群联邦身份验证 (或让集群管理员为您执行此操作),以及如何正确配置客户端。

#### 我可以通过集群联邦 API 成功创建一个联邦服务，但在我的底层集群中没有创建对应的服务

检查：

1.  您的集群在集群联邦 API (`kubectl describe clusters`)中正确注册。
2.  你的集群都是 ‘Active’ 的。这意味着集群联邦系统能够连接和验证集群端点。如果不是，请查阅 federation-controller-manager pod 的日志以确定可能发生的故障。 `kubectl --namespace=federation logs $(kubectl get pods --namespace=federation -l module=federation-controller-manager -o name)`
3.  该集群提供给集群联邦 API 的登录凭证具有正确的授权和配额，以在集群中的相关命名空间中创建服务。如果不是这种情况，您应该再次看到相关的错误消息提供了上述日志文件中的更多细节。
4.  是否有其他错误阻止了服务创建操作的成功(在`kubectl logs federation-controller-manager --namespace federation`的输出中查找`service-controller`错误)。

#### 我可以成功创建联邦服务，但是在我的 DNS 提供程序中没有创建匹配的DNS记录

检查:

1.您的联邦名称，DNS 提供程序，DNS 域名配置正确。请参阅[联邦集群管理指南](https://kubernetes.io/docs/admin/federation/)或[教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)以了解 如何配置集群联邦身份验证系统的 DNS 提供程序(或让您的集群管理员为您执行此操作)。 2.确认集群联邦的服务控制器已经成功连接到所选的 DNS 提供程序并进行身份验证(在`kubectl logs federation-controller-manager --namespace federation`的输出中查找`service-controller` errors 或 successes)。

#### 匹配到我在 DNS 提供商中创建的 DNS 记录，但客户端无法解析这些记录

检查:

1.  管理您的联邦 DNS 域的 DNS 注册管理器已正确配置到指向您配置的DNS提供商的域名服务器。 例如，请参阅 [Google Domains文档](https://support.google.com/domains/answer/3290309?hl=zh_CN&ref_topic=3251230) 和 [Google云端DNS文档](https://cloud.google.com/dns/update-name-servers) ，或从您的域名注册商或 DNS 提供商获得同样的指导。

#### 此故障排除指南并没有帮助我解决我的问题

1.  请使用我们的[支持渠道](https://kubernetes.io/docs/tasks/debug-application-cluster/troubleshooting/)寻求帮助。

### 更多信息

-   [Federation proposal](https://git.k8s.io/community/contributors/design-proposals/multicluster/federation.md) 详细介绍了激发集群联邦工作的用例.







## 管理联合集群控制面板

### 联合集群

>   **Note:** `Federation V1`， 是当前的 Kubernetes 联邦 API， 它“原样”重用 Kubernetes API 资源， 其许多特性目前被认为是 alpha。 没有明确的途径将 API 发展成 GA； 然而， 除了 Kubernetes API 之外， 还有一个 `Federation V2` 正在努力实现专用的联邦 API。详细信息可在 [sig-multicluster 社区页面](https://github.com/kubernetes/community/tree/master/sig-multicluster) 获得。

本指南介绍了如何在联邦控制平面中使用集群 API 资源。

与 Deployment、Service 和 ConfigMap 等 Kubernetes 资源不同，cluster 只存在于联邦上下文中，即这些请求必须提交给联邦 api-server。

#### 准备开始

-   本指南假设您已安装有一个正在运行的 Kubernetes 集群联邦。如果没有，那么请转到 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)，了解如何启动联邦集群(或者让集群管理员为您执行此操作)。 其他教程，例如 Kelsey Hightower 的[联邦 Kubernetes 教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)， 也可能帮助您创建联邦 Kubernetes 集群。

-   你需要具备基本的 [Kubernetes 工作知识](https://v1-14.docs.kubernetes.io/docs/setup/pick-right-solution/)。

#### 集群列表

要列出联邦中可用的 cluster，可以使用 [kubectl](https://v1-14.docs.kubernetes.io/docs/user-guide/kubectl/) 运行:

```shell
kubectl --context=federation get clusters
```

`--context=federation` 参数告诉 kubectl 将请求提交给联邦 apiserver， 而不是将其发送给 Kubernetes 集群。如果您将其提交给 k8s 集群，则会收到错误消息

```
the server doesn't have a resource type "clusters"
```

如果您传递了正确的联邦上下文，但是收到了一条消息错误

```
No resources found.
```

这表示着没有向联邦添加任何集群。

#### 创建一个联邦集群

在联邦中创建`集群`资源意味着将其加入到联邦中。因此，您可以使用 `kubefed join`。基本上，您需要为新群集指定一个名称， 并说明与承载联邦的集群相对应的上下文的名称。下面的示例命令将集群 `gondor` 添加到运行在主机集群 `rivendell` 上的联邦：

```shell
kubefed join gondor --host-cluster-context=rivendell
```

您可以在 [kubefed 指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/#adding-a-cluster-to-a-federation)的相关章节中找到更多关于如何实现这一点的详细信息。

#### 删除一个联邦集群

与创建集群相反，删除集群意味着从联邦中取消加入这个集群。这可以通过 `kubefed unjoin` 命令完成。要删除 `gondor` 群集，需要执行以下操作：

```shell
kubefed unjoin gondor --host-cluster-context=rivendell
```

你可以在 [kubefed 指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/#removing-a-cluster-from-a-federation)中找到更多关于取消加入的详细信息。

##### 标记集群

您可以使用与其他任何 Kubernetes 对象相同的方法标记集群，这有助于对集群进行分组，也可以配合使用 ClusterSelector。

```shell
kubectl --context=rivendell label cluster gondor key1=value1 key2=value2
```

##### ClusterSelector 注解

从 Kubernetes 1.7 开始，alpha 支持通过注解 `federation.alpha.kubernetes.io/cluster-selector` 在联邦集群中引导对象。。*ClusterSelector* 在概念上类似于 `nodeSelector`，但是它不是针对节点上的标签进行选择，而是针对联邦集群上的标签进行选择。

注解值必须是 JSON 格式并且必须可解析为 [ClusterSelector API 类型](https://v1-14.docs.kubernetes.io/docs/reference/federation/v1beta1/definitions/#_v1beta1_clusterselector)。 例如：`[{"key": "load", "operator": "Lt", "values": ["10"]}]`，不能正确解析的内容将抛出一个错误，并阻止将对象分发到任何联邦集群。alpha 实现包含 ConfigMap、Secret、Daemonset、Service 和 Ingress 类型的对象。

下面是一个 ClusterSelector 注释示例，它只会选择带有标签 `pci=true` 不选择标签为 `environment=test` 的集群：

```yaml
  metadata:
    annotations:
      federation.alpha.kubernetes.io/cluster-selector: '[{"key": "pci", "operator":
        "In", "values": ["true"]}, {"key": "environment", "operator": "NotIn", "values":
        ["test"]}]'
```

*key* 与联邦集群上的标签名称匹配。

*values* 与联邦集群上的标签值匹配。

可能的*操作符*有：`In`、`NotIn`、`Exists`、`DoesNotExist`、`Gt`、`Lt`。

*values* 字段在指定 `Exists` 或 `DoesNotExist` 时为空，在使用 `In` 或 `NotIn` 时可能包含多个字符串。

目前，`Gt` 和 `Lt` 操作符只支持整数。

#### 集群 API 参考

完整的集群 API 参考目前在 `federation/v1beta1` 中，更多细节可以在[联邦 API 参考页面](https://v1-14.docs.kubernetes.io/docs/reference/federation/)中找到。





### 联合 ConfigMap

这个指南介绍了如何在联邦控制平面中使用 ConfigMaps。

联邦 ConfigMaps 与传统的 [Kubernetes ConfigMaps](https://v1-14.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/) 十分相似，并且提供相同的功能。 在联邦控制平面中创建它们可确保它们在联邦集群间的同步。

#### 准备开始

-   本指南假设您已安装有一个正在运行的 Kubernetes 集群联邦。如果没有，那么请转到 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)，了解如何启动联邦集群(或者让集群管理员为您执行此操作)。 其他教程，例如 Kelsey Hightower 的[联邦 Kubernetes 教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)， 也可能帮助您创建联邦 Kubernetes 集群。
-   你还需要有基本的[Kubernetes 工作常识](https://v1-14.docs.kubernetes.io/docs/setup/pick-right-solution/)，尤其是[ConfigMaps](https://v1-14.docs.kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/)。

#### 创建一个 Federated ConfigMap

Federated ConfigMap 的 API 100% 兼容传统的 Kubernetes ConfigMap。你可以通过发送一个请求到联邦 apiserver 来创建一个 ConfigMap。

你可以使用[kubectl](https://v1-14.docs.kubernetes.io/docs/user-guide/kubectl/)执行：

```shell
kubectl --context=federation-cluster create -f myconfigmap.yaml
```

`--context=federation-cluster` 参数告诉 kubectl 发送请求至联邦 apiserver 而不是 Kubernetes 集群。

一旦联邦 ConfigMap 创建成功，联邦控制平面会在所有底层 Kubernetes 集群中创建匹配的 ConfigMap。你可以通过检查每个底层的集群来验证这点，例如：

```shell
kubectl --context=gce-asia-east1a get configmap myconfigmap
```

以上是假设你在客户端中为该区域中的集群配置了一个名为 ‘gce-asia-east1a’ 的上下文。

这些在底层集群中的 ConfigMaps 会自动匹配联邦 ConfigMap。

#### 更新一个联邦 ConfigMap

你可以像更新 Kubernetes ConfigMap 一样更新一个联邦 ConfigMap。然而，对于联邦 ConfigMap，你必须发送请求至联邦 apiserver 而不是指定的 Kubernetes 集群。 联邦控制平面会确保无论何时联邦 ConfigMap 被更新，它都会更新所有底层集群中想相匹配的 ConfigMaps。

#### 删除一个联邦 ConfigMap

你可以像删除一个 Kubernetes ConfigMap 一样删除一个联邦 ConfigMap。然而，对于联邦 ConfigMap，你必须发送请求至联邦 apiserver 而不是指定的 Kubernetes 集群。

例如，你可以使用 kubectl 执行：

```shell
kubectl --context=federation-cluster delete configmap
```

>   Note:
>
>   删除一个联邦 ConfigMap 不会从底层集群中删除相应的 ConfigMaps。你必须手动地删除底层 ConfigMaps。





### 联合守护进程集(DaemonSet)

本指南介绍了如何在联邦控制平面中使用 DaemonSet。

联邦控制平面中的 DaemonSet （即本指南中的联邦 DaemonSet）与传统的 Kubernetes [DaemonSet](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/) 非常相似，并提供相同的功能。

在联邦控制平面中创建它们可以确保它们跨联邦中的所有集群保持同步。

#### 准备开始

-   本指南假设您已安装有一个正在运行的 Kubernetes 集群联邦。如果没有，那么请转到 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)，了解如何启动联邦集群(或者让集群管理员为您执行此操作)。 其他教程，例如 Kelsey Hightower 的[联邦 Kubernetes 教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)， 也可能帮助您创建联邦 Kubernetes 集群。
-   您还需要具备基本的 [Kubernetes 工作知识](https://v1-14.docs.kubernetes.io/docs/setup/pick-right-solution/)，特别是关于 [DaemonSet](https://v1-14.docs.kubernetes.io/docs/concepts/workloads/controllers/daemonset/)。

#### 创建一个联邦 DaemonSet

用于联邦 DaemonSet 的 API 与用于传统 Kubernetes DaemonSet 的 API 100% 兼容。 您可以通过向联邦 apiserver 发送请求来创建 DaemonSet。

您可以通过使用 [kubectl](https://v1-14.docs.kubernetes.io/docs/user-guide/kubectl/) 运行以下命令来执行此操作：

```shell
kubectl --context=federation-cluster create -f mydaemonset.yaml
```

`--context=federation-cluster` 参数告诉 kubectl 将请求提交给联邦 apiserver， 而不是发送给 Kubernetes 集群。

一旦创建了联邦 DaemonSet，联邦控制平面将在所有底层 Kubernetes 集群中创建匹配的 DaemonSet。 您可以通过检查每个基础集群来验证这一点，例如：

```shell
kubectl --context=gce-asia-east1a get daemonset mydaemonset
```

上面假设您的客户机中为该区域中的集群配置了一个名为 ‘gce-asia-east1a’ 的上下文。

#### 更新联邦 DaemonSet

您可以像更新 Kubernetes DaemonSet 一样更新联邦 DaemonSet； 但是，对于联邦 DaemonSet，您必须将请求发送到 联邦 apiserver 而不是将其发送到特定的 Kubernetes 集群。 联邦控制平面确保每当更新联邦 DaemonSet 时，它都会更新所有底层集群中的相应 DaemonSet 以匹配它。

#### 删除联邦 DaemonSet

您可以删除联邦 DaemonSet，就像删除 Kubernetes DaemonSet 一样； 但是，对于联邦 DaemonSet，您必须将请求发送到 联邦 apiserver 而不是将其发送到特定的 Kubernetes 集群。

例如，您可以通过使用 kubectl 运行以下命令执行此操作：

```shell
kubectl --context=federation-cluster delete daemonset mydaemonset
```





### 联合部署

This guide explains how to use Deployments in the Federation control plane.

Deployments in the federation control plane (referred to as “Federated Deployments” in this guide) are very similar to the traditional [Kubernetes Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and provide the same functionality. Creating them in the federation control plane ensures that the desired number of replicas exist across the registered clusters.

**FEATURE STATE:** `Kubernetes 1.5` [alpha](https://kubernetes.io/docs/tasks/federation/administer-federation/deployment/#)

Some features (such as full rollout compatibility) are still in development.

#### Before you begin

-   This guide assumes that you have a running Kubernetes Cluster Federation installation. If not, then head over to the [federation admin guide](https://kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/) to learn how to bring up a cluster federation (or have your cluster administrator do this for you). Other tutorials, such as Kelsey Hightower’s [Federated Kubernetes Tutorial](https://github.com/kelseyhightower/kubernetes-cluster-federation), might also help you create a Federated Kubernetes cluster.
-   You should also have a basic [working knowledge of Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) in general and [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) in particular.

#### Creating a Federated Deployment

The API for Federated Deployment is compatible with the API for traditional Kubernetes Deployment. You can create a Deployment by sending a request to the federation apiserver.

You can do that using [kubectl](https://kubernetes.io/docs/user-guide/kubectl/) by running:

```shell
kubectl --context=federation-cluster create -f mydeployment.yaml
```

The `--context=federation-cluster` flag tells kubectl to submit the request to the Federation apiserver instead of sending it to a Kubernetes cluster.

Once a Federated Deployment is created, the federation control plane will create a Deployment in all underlying Kubernetes clusters. You can verify this by checking each of the underlying clusters, for example:

```shell
kubectl --context=gce-asia-east1a get deployment mydep
```

The above assumes that you have a context named ‘gce-asia-east1a’ configured in your client for your cluster in that zone.

These Deployments in underlying clusters will match the federation Deployment *except* in the number of replicas and revision-related annotations. Federation control plane ensures that the sum of replicas in each cluster combined matches the desired number of replicas in the Federated Deployment.

##### Spreading Replicas in Underlying Clusters

By default, replicas are spread equally in all the underlying clusters. For example: if you have 3 registered clusters and you create a Federated Deployment with `spec.replicas = 9`, then each Deployment in the 3 clusters will have `spec.replicas=3`. To modify the number of replicas in each cluster, you can specify [FederatedReplicaSetPreference](https://github.com/kubernetes/federation/blob/master/apis/federation/types.go) as an annotation with key `federation.kubernetes.io/deployment-preferences` on Federated Deployment.

#### Updating a Federated Deployment

You can update a Federated Deployment as you would update a Kubernetes Deployment; however, for a Federated Deployment, you must send the request to the federation apiserver instead of sending it to a specific Kubernetes cluster. The federation control plane ensures that whenever the Federated Deployment is updated, it updates the corresponding Deployments in all underlying clusters to match it. So if the rolling update strategy was chosen then the underlying cluster will do the rolling update independently and `maxSurge`and `maxUnavailable` will apply only to individual clusters. This behavior may change in the future.

If your update includes a change in number of replicas, the federation control plane will change the number of replicas in underlying clusters to ensure that their sum remains equal to the number of desired replicas in Federated Deployment.

#### Deleting a Federated Deployment

You can delete a Federated Deployment as you would delete a Kubernetes Deployment; however, for a Federated Deployment, you must send the request to the federation apiserver instead of sending it to a specific Kubernetes cluster.

For example, you can do that using kubectl by running:

```shell
kubectl --context=federation-cluster delete deployment mydep
```





### 联合事件(Events)

本指南介绍如何在联邦控制平面中使用事件来帮助调试。

#### 前提条件

本指南假设您已安装好了 Kubernetes 集群联邦。如果没有，请先查看 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/admin/federation/) 来学习怎样安装集群联邦（也可以让您的管理员替您完成安装）。 其他的教程，例如 Kelsey Hightower 编写的 [这个教程](https://github.com/kelseyhightower/kubernetes-cluster-federation) 也会帮到你。

您还应该具备 [Kubernetes 的基本操作](https://v1-14.docs.kubernetes.io/docs/setup) 的知识。

#### 查看联邦事件

联邦控制平面的事件（本指南中称为“联邦事件”）与传统的 Kubernetes 事件非常相似，提供相同功能。 联邦事件仅存储在联邦控制平面中，不会传递给下层 Kubernetes 集群。

联邦控制器在处理 API 资源时创建事件，以向用户显示它们所处的状态。 您可以通过运行以下命令从联邦 API 服务器获取所有事件：

```shell
kubectl --context=federation-cluster get events
```

标准的 kubectl get、update、delete 命令都可以正常工作。





### 联合水平pod缩放器(HPA)

**FEATURE STATE:** `Kubernetes v1.15` [alpha](https://kubernetes.io/docs/tasks/federation/administer-federation/hpa/#)

本指南介绍了如何在联邦控制平面中使用联邦横向 pod 自动伸缩器 （HPAs）。

联邦控制平面中的 HPA 与传统的 [Kubernetes HPA](https://v1-14.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 非常相似，提供相同的功能。 在联邦控制平面中针对联邦对象创建的 HPA 将保证目标对象的期望副本在所有注册集群上进行伸缩，而不是在单个集群上。此外，控制平面持续监控联邦集群中单个 HPA 的状态，通过操作联邦集群中 HPA 对象的最小和最大限制值来确保工作副本被移动到最需要的地方。

#### 准备开始

-   本指南假设您已安装有一个正在运行的 Kubernetes 集群联邦。如果没有，那么请转到 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)，了解如何启动联邦集群(或者让集群管理员为您执行此操作)。 其他教程，例如 Kelsey Hightower 的[联邦 Kubernetes 教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)， 也可能帮助您创建联邦 Kubernetes 集群。
-   通常您还应当拥有基本的 [Kubernetes 应用知识](https://v1-14.docs.kubernetes.io/docs/setup/)，特别是 [HPA](https://v1-14.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 。

联邦 HPA 是一个 alpha 特性。默认情况下该 API 没有在联邦 API server 上启用。要使用此特性，部署联邦控制平面的用户或管理员需要使用 `--runtime-config=api/all=true` 选项运行联邦 API server 以启用所有 API（包括 alpha API）。此外，联邦 HPA 只能和 CPU 使用率度量一起使用。

#### 创建联邦 HPA

联邦 HPA 的 API 100% 兼容传统 Kubernetes HPA 的 API。您可以通过向联邦 API server 发送请求来创建 HPA。

您可以使用 [kubectl](https://v1-14.docs.kubernetes.io/docs/user-guide/kubectl/) 运行命令：

```shell
cat <<EOF | kubectl --context=federation-cluster create -f -
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
EOF
```

`--context=federation-cluster` 参数告诉 `kubectl` 将请求提交到联邦 API server 而不是发送给某一个 Kubernetes 集群。

一旦创建了联邦 HPA，联邦控制平面就会对其进行分区，并在所有基础集群中创建 HPA。从 Kubernetes v1.7 开始，[cluster selectors](https://v1-14.docs.kubernetes.io/docs/tasks/administer-federation/cluster/#clusterselector-annotation) 同样可以用来限制任何联邦对象，包括集群子集中的 HPA。

您可以通过检查每个基础集群来验证创建是否成功。例如，您的客户端配置了名为 `gce-asia-east1a` 的上下文，集群处于该区域中：

```shell
kubectl --context=gce-asia-east1a get HPA php-apache
```

除了最小和最大副本的数量之外，基础集群中的 HPA 将与联邦 HPA 相匹配。联邦控制平面将保证每个集群的最大副本数总和等于联邦 HPA 对象的最大副本数，并且最小副本总和大于等于联邦 HPA 对象的最小副本数。

>   **Note:** 集群的最小副本总和不能为 0。

##### 在基础集群中分发 HPA 的最小和最大副本数

默认情况下，首先将最大副本数在所有基础集群中平均分布，然后再将最小副本数分发给已接收了最大副本数的集群。这意味着，如果指定的最大副本数大于联邦的集群数，则每个集群都将获得一个 HPA。反之，如果指定的最大副本数小于联邦的集群数，则会跳过某些集群。

举例说明：如果您有3个注册集群，并且您使用参数 `spec.maxReplicas = 9` 和 `spec.minReplicas = 2` 创建了一个联邦 HPA，那么3个集群中的每个 HPA 都将获得 `spec.maxReplicas=3` 和 `spec.minReplicas = 1` 的参数。

目前，联邦 HPA 仅可以使用默认的分发机制，但在将来，用户将可以设置偏好来控制和/或限制分发过程。

#### 更新联邦 HPA

您可以像更新 Kubernetes HPA 一样更新联邦 HPA；但是，对于联邦 HPA 您必须将请求发送给联邦 API server 而不是发送到一个特定的 Kubernetes 集群。联邦控制平面将保证在联邦 HPA 被更新后，它会对所有基础集群中与之对应的 HPA 进行更新。

如果您的更新修改了副本数量，联邦控制平面将修改基础集群中的副本数量，保证最小副本总数和最大副本总数仍然符合要求正如前文所述。

#### 删除联邦 HPA

您可以像删除 Kubernetes HPA 一样删除联邦 HPA；但是，对于联邦 HPA， 您必须将请求发送给联邦 API server 而不是发送到一个特定的 Kubernetes 集群。

>   **Note:** 如果要删除所有基础集群中的联邦资源，应该使用 [级联删除](https://v1-14.docs.kubernetes.io/docs/concepts/cluster-administration/federation/#cascading-deletion)。

例如，您可以使用 `kubectl` 运行命令：

```shell
kubectl --context=federation-cluster delete HPA php-apache
```

#### 使用联邦 HPA 的其他方法

对于和联邦控制平面（或简称联邦）交互的用户，这种交互几乎和与一个普通 Kubernetes 集群的交互完全相同（但只能使用有限的联邦 API 集合）。由于目前 Deployment 和 HorizontalPodAutoscaler 都有联邦版本，类似 `kubectl run` 和 `kubectl autoscale` 的 `kubectl` 命令都可以在联邦上运行。鉴于这个事实，[horizontal pod autoscaler walkthrough](https://v1-14.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/)中指定的机制同样可以在联邦中运行。但也需要注意，当[在目标 deployment 上生成负载](https://v1-14.docs.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/#step-three-increase-load) 时，应该针对一个特定的联邦集群（或多个集群）而不是整个联邦。

#### 结论

使用联邦 HPA 是为了保证工作副本移动到最需要它们的地方，换句话说，是移动到负载超过期望阈值的地方。联邦 HPA 功能通过操作其在联邦集群中创建的 HPA 的最小和最大副本数来实现此目的。它并不直接监控联邦集群中目标对象的度量值，实际上依赖于集群内部的 HPA 控制器来监控目标 pod 的度量值并更新相关字段。集群内部的 HPA 控制器监控目标 pod 的度量值并更新相关字段，如期望的副本数（在基于度量的计算之后）和当前副本数（通过观察集群中 pod 的当前状态）。另一方面，联邦 HPA 控制器只监控集群特定的 HPA 对象字段并更新集群中 HPA 对象的最小和最大副本数字段，这些对象的副本数和阈值匹配。

例如，如果一个集群同时拥有期望副本数和当前副本数，其值与最大副本数相同，但当前 CPU 的平均利用率仍然高于目标使用率（它们都是本地 HPA 对象上的字段）时，集群中的目标应用程序就需要更多的副本，但是扩容动作被本地 HPA 对象上设置的最大副本数限制。在这种场景下，联邦 HPA 控制器将会扫描所有集群并尝试查找没有这种条件的集群（意思是期望副本数小于最大值且当前 CPU 平均使用率低于阈值）。如果找到了这样的集群，它会减少该集群中 HPA 的最大副本数，并增加最需要副本的集群上的 HPA 的最大副本数。

还存在许多类似的情况导致联邦 HPA 控制器检查并移动联邦集群中本地 HPA 的最大和最小副本数，以确保副本最终移动（或保留）到最需要它们的集群中。

更多相关信息请参考[“联邦 HPA 设计提议”](https://github.com/kubernetes/community/pull/593)。





### 联合Ingress

This page explains how to use Kubernetes Federated Ingress to deploy a common HTTP(S) virtual IP load balancer across a federated service running in multiple Kubernetes clusters. As of v1.4, clusters hosted in Google Cloud (both Google Kubernetes Engine and GCE, or both) are supported. This makes it easy to deploy a service that reliably serves HTTP(S) traffic originating from web clients around the globe on a single, static IP address. Low network latency, high fault tolerance and easy administration are ensured through intelligent request routing and automatic replica relocation (using [Federated ReplicaSets](https://kubernetes.io/docs/tasks/administer-federation/replicaset/). Clients are automatically routed, via the shortest network path, to the cluster closest to them with available capacity (despite the fact that all clients use exactly the same static IP address). The load balancer automatically checks the health of the pods comprising the service, and avoids sending requests to unresponsive or slow pods (or entire unresponsive clusters).

Federated Ingress is released as an alpha feature, and supports Google Cloud Platform (Google Kubernetes Engine, GCE and hybrid scenarios involving both) in Kubernetes v1.4. Work is under way to support other cloud providers such as AWS, and other hybrid cloud scenarios (e.g. services spanning private on-premises as well as public cloud Kubernetes clusters).

You create Federated Ingresses in much that same way as traditional [Kubernetes Ingresses](https://kubernetes.io/docs/concepts/services-networking/ingress/): by making an API call which specifies the desired properties of your logical ingress point. In the case of Federated Ingress, this API call is directed to the Federation API endpoint, rather than a Kubernetes cluster API endpoint. The API for Federated Ingress is 100% compatible with the API for traditional Kubernetes Services.

Once created, the Federated Ingress automatically:

-   Creates matching Kubernetes Ingress objects in every cluster underlying your Cluster Federation
-   Ensures that all of these in-cluster ingress objects share the same logical global L7 (that is, HTTP(S)) load balancer and IP address
-   Monitors the health and capacity of the service shards (that is, your pods) behind this ingress in each cluster
-   Ensures that all client connections are routed to an appropriate healthy backend service endpoint at all times, even in the event of pod, cluster, availability zone or regional outages

Note that in the case of Google Cloud, the logical L7 load balancer is not a single physical device (which would present both a single point of failure, and a single global network routing choke point), but rather a [truly global, highly available load balancing managed service](https://cloud.google.com/load-balancing/), globally reachable via a single, static IP address.

Clients inside your federated Kubernetes clusters (Pods) will be automatically routed to the cluster-local shard of the Federated Service backing the Ingress in their cluster if it exists and is healthy, or the closest healthy shard in a different cluster if it does not. Note that this involves a network trip to the HTTP(s) load balancer, which resides outside your local Kubernetes cluster but inside the same GCP region.

#### Before you begin

This document assumes that you have a running Kubernetes Cluster Federation installation. If not, then see the [federation admin guide](https://kubernetes.io/docs/tasks/federation/set-up-cluster-federation-kubefed/) to learn how to bring up a cluster federation (or have your cluster administrator do this for you). Other tutorials, for example [this one](https://github.com/kelseyhightower/kubernetes-cluster-federation) by Kelsey Hightower, are also available to help you.

You should also have a basic [working knowledge of Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) in general, and [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) in particular.

#### Creating a federated ingress

You can create a federated ingress in any of the usual ways, for example, using kubectl:

```shell
kubectl --context=federation-cluster create -f myingress.yaml
```

For example ingress YAML configurations, see the [Ingress User Guide](https://kubernetes.io/docs/concepts/services-networking/ingress/). The `--context=federation-cluster` flag tells kubectl to submit the request to the Federation API endpoint, with the appropriate credentials. If you have not yet configured such a context, see the [federation admin guide](https://kubernetes.io/docs/admin/federation/) or one of the [administration tutorials](https://github.com/kelseyhightower/kubernetes-cluster-federation) to find out how to do so.

The Federated Ingress automatically creates and maintains matching Kubernetes ingresses in all of the clusters underlying your federation. These cluster-specific ingresses (and their associated ingress controllers) configure and manage the load balancing and health checking infrastructure that ensures that traffic is load balanced to each cluster appropriately.

You can verify this by checking in each of the underlying clusters. For example:

```shell
kubectl --context=gce-asia-east1a get ingress myingress
NAME        HOSTS     ADDRESS           PORTS     AGE
myingress   *         130.211.5.194     80, 443   1m
```

The above assumes that you have a context named ‘gce-asia-east1a’ configured in your client for your cluster in that zone. The name and namespace of the underlying ingress automatically matches those of the Federated Ingress that you created above (and if you happen to have had ingresses of the same name and namespace already existing in any of those clusters, they will be automatically adopted by the Federation and updated to conform with the specification of your Federated Ingress. Either way, the end result will be the same).

The status of your Federated Ingress automatically reflects the real-time status of the underlying Kubernetes ingresses. For example:

```shell
kubectl --context=federation-cluster describe ingress myingress

Name:           myingress
Namespace:      default
Address:        130.211.5.194
TLS:
  tls-secret terminates
Rules:
  Host  Path    Backends
  ----  ----    --------
  * *   echoheaders-https:80 (10.152.1.3:8080,10.152.2.4:8080)
Annotations:
  https-target-proxy:       k8s-tps-default-myingress--ff1107f83ed600c0
  target-proxy:         k8s-tp-default-myingress--ff1107f83ed600c0
  url-map:          k8s-um-default-myingress--ff1107f83ed600c0
  backends:         {"k8s-be-30301--ff1107f83ed600c0":"Unknown"}
  forwarding-rule:      k8s-fw-default-myingress--ff1107f83ed600c0
  https-forwarding-rule:    k8s-fws-default-myingress--ff1107f83ed600c0
Events:
  FirstSeen LastSeen    Count   From                SubobjectPath   Type        Reason  Message
  --------- --------    -----   ----                -------------   --------    ------  -------
  3m        3m      1   {loadbalancer-controller }          Normal      ADD default/myingress
  2m        2m      1   {loadbalancer-controller }          Normal      CREATE  ip: 130.211.5.194
```

Note that:

-   The address of your Federated Ingress corresponds with the address of all of the underlying Kubernetes ingresses (once these have been allocated - this may take up to a few minutes).
-   You have not yet provisioned any backend Pods to receive the network traffic directed to this ingress (that is, ‘Service Endpoints’ behind the service backing the Ingress), so the Federated Ingress does not yet consider these to be healthy shards and will not direct traffic to any of these clusters.
-   The federation control system automatically reconfigures the load balancer controllers in all of the clusters in your federation to make them consistent, and allows them to share global load balancers. But this reconfiguration can only complete successfully if there are no pre-existing Ingresses in those clusters (this is a safety feature to prevent accidental breakage of existing ingresses). So, to ensure that your federated ingresses function correctly, either start with new, empty clusters, or make sure that you delete (and recreate if necessary) all pre-existing Ingresses in the clusters comprising your federation.

#### Adding backend services and pods

To render the underlying ingress shards healthy, you need to add backend Pods behind the service upon which the Ingress is based. There are several ways to achieve this, but the easiest is to create a Federated Service and Federated ReplicaSet. To create appropriately labelled pods and services in the 13 underlying clusters of your federation:

```shell
kubectl --context=federation-cluster create -f services/nginx.yaml
kubectl --context=federation-cluster create -f myreplicaset.yaml
```

Note that in order for your federated ingress to work correctly on Google Cloud, the node ports of all of the underlying cluster-local services need to be identical. If you’re using a federated service this is easy to do. Simply pick a node port that is not already being used in any of your clusters, and add that to the spec of your federated service. If you do not specify a node port for your federated service, each cluster will choose its own node port for its cluster-local shard of the service, and these will probably end up being different, which is not what you want.

You can verify this by checking in each of the underlying clusters. For example:

```shell
kubectl --context=gce-asia-east1a get services nginx
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP      PORT(S)   AGE
nginx     ClusterIP   10.63.250.98   104.199.136.89   80/TCP    9m
```

#### Hybrid cloud capabilities

Federations of Kubernetes Clusters can include clusters running in different cloud providers (for example, Google Cloud, AWS), and on-premises (for example, on OpenStack). However, in Kubernetes v1.4, Federated Ingress is only supported across Google Cloud clusters.

#### Discovering a federated ingress

Ingress objects (in both plain Kubernetes clusters, and in federations of clusters) expose one or more IP addresses (via the Status.Loadbalancer.Ingress field) that remains static for the lifetime of the Ingress object (in future, automatically managed DNS names might also be added). All clients (whether internal to your cluster, or on the external network or internet) should connect to one of these IP or DNS addresses. All client requests are automatically routed, via the shortest network path, to a healthy pod in the closest cluster to the origin of the request. So for example, HTTP(S) requests from internet users in Europe will be routed directly to the closest cluster in Europe that has available capacity. If there are no such clusters in Europe, the request will be routed to the next closest cluster (typically in the U.S.).

#### Handling failures of backend pods and whole clusters

Ingresses are backed by Services, which are typically (but not always) backed by one or more ReplicaSets. For Federated Ingresses, it is common practise to use the federated variants of Services and ReplicaSets for this purpose.

In particular, Federated ReplicaSets ensure that the desired number of pods are kept running in each cluster, even in the event of node failures. In the event of entire cluster or availability zone failures, Federated ReplicaSets automatically place additional replicas in the other available clusters in the federation to accommodate the traffic which was previously being served by the now unavailable cluster. While the Federated ReplicaSet ensures that sufficient replicas are kept running, the Federated Ingress ensures that user traffic is automatically redirected away from the failed cluster to other available clusters.

#### Troubleshooting

##### I cannot connect to my cluster federation API.

Check that your:

1.  Client (typically `kubectl`) is correctly configured (including API endpoints and login credentials).
2.  Cluster Federation API server is running and network-reachable.

See the [federation admin guide](https://kubernetes.io/docs/admin/federation/) to learn how to bring up a cluster federation correctly (or have your cluster administrator do this for you), and how to correctly configure your client.

##### I can create a Federated Ingress/service/replicaset successfully against the cluster federation API, but no matching ingresses/services/replicasets are created in my underlying clusters.

Check that:

1.  Your clusters are correctly registered in the Cluster Federation API. (`kubectl describe clusters`)
2.  Your clusters are all ‘Active’. This means that the cluster Federation system was able to connect and authenticate against the clusters’ endpoints. If not, consult the event logs of the federation-controller-manager pod to ascertain what the failure might be. (`kubectl --namespace=federation logs $(kubectl get pods --namespace=federation -l module=federation-controller-manager -o name`)
3.  That the login credentials provided to the Cluster Federation API for the clusters have the correct authorization and quota to create ingresses/services/replicasets in the relevant namespace in the clusters. Again you should see associated error messages providing more detail in the above event log file if this is not the case.
4.  Whether any other error is preventing the service creation operation from succeeding (look for `ingress-controller`, `service-controller` or `replicaset-controller`, errors in the output of `kubectl logs federation-controller-manager --namespace federation`).

##### I can create a federated ingress successfully, but request load is not correctly distributed across the underlying clusters.

Check that:

1.  The services underlying your federated ingress in each cluster have identical node ports. See [above](https://kubernetes.io/docs/tasks/federation/administer-federation/ingress/#creating_a_federated_ingress) for further explanation.
2.  The load balancer controllers in each of your clusters are of the correct type (“GLBC”) and have been correctly reconfigured by the federation control plane to share a global GCE load balancer (this should happen automatically). If they are of the correct type, and have been correctly reconfigured, the UID data item in the GLBC configmap in each cluster will be identical across all clusters. See [the GLBC docs](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md) for further details. If this is not the case, check the logs of your federation controller manager to determine why this automated reconfiguration might be failing.
3.  No ingresses have been manually created in any of your clusters before the above reconfiguration of the load balancer controller completed successfully. Ingresses created before the reconfiguration of your GLBC will interfere with the behavior of your federated ingresses created after the reconfiguration (see [the GLBC docs](https://github.com/kubernetes/ingress/blob/7dcb4ae17d5def23d3e9c878f3146ac6df61b09d/controllers/gce/README.md) for further information). To remedy this, delete any ingresses created before the cluster joined the federation (and had its GLBC reconfigured), and recreate them if necessary.

#### What's next

-   If you need assistance, use one of the

     

    support channels

     

    to seek assistance.

    -   For details about use cases that motivated this work, see [Federation proposal](https://git.k8s.io/community/contributors/design-proposals/multicluster/federation.md).





### 联合Job

This guide explains how to use jobs in the federation control plane.

Jobs in the federation control plane (referred to as “federated jobs” in this guide) are similar to the traditional [Kubernetes jobs](https://kubernetes.io/docs/concepts/workloads/controllers/job/), and provide the same functionality. Creating jobs in the federation control plane ensures that the desired number of parallelism and completions exist across the registered clusters.

#### Before you begin

-   This guide assumes that you have a running Kubernetes Cluster Federation installation. If not, then head over to the [federation admin guide](https://kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/) to learn how to bring up a cluster federation (or have your cluster administrator do this for you). Other tutorials, such as Kelsey Hightower’s [Federated Kubernetes Tutorial](https://github.com/kelseyhightower/kubernetes-cluster-federation), might also help you create a Federated Kubernetes cluster.
-   You should also have a basic [working knowledge of Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) in general and [jobs](https://kubernetes.io/docs/concepts/workloads/controllers/jobs-run-to-completion/) in particular.

#### Creating a federated job

The API for federated jobs is fully compatible with the API for traditional Kubernetes jobs. You can create a job by sending a request to the federation apiserver.

You can do that using [kubectl](https://kubernetes.io/docs/user-guide/kubectl/) by running:

```shell
kubectl --context=federation-cluster create -f myjob.yaml
```

The `--context=federation-cluster` flag tells kubectl to submit the request to the federation API server instead of sending it to a Kubernetes cluster.

Once a federated job is created, the federation control plane creates a job in all underlying Kubernetes clusters. You can verify this by checking each of the underlying clusters, for example:

```shell
kubectl --context=gce-asia-east1a get job myjob
```

The previous example assumes that you have a context named `gce-asia-east1a`configured in your client for your cluster in that zone.

The jobs in the underlying clusters match the federated job except in the number of parallelism and completions. The federation control plane ensures that the sum of the parallelism and completions in each cluster matches the desired number of parallelism and completions in the federated job.

##### Spreading job tasks in underlying clusters

By default, parallelism and completions are spread equally in all underlying clusters. For example: if you have 3 registered clusters and you create a federated job with `spec.parallelism = 9` and `spec.completions = 18`, then each job in the 3 clusters has `spec.parallelism = 3` and `spec.completions = 6`. To modify the number of parallelism and completions in each cluster, you can specify [ReplicaAllocationPreferences](https://github.com/kubernetes/federation/blob/master/apis/federation/types.go) as an annotation with key `federation.kubernetes.io/job-preferences` on the federated job.

#### Updating a federated job

You can update a federated job as you would update a Kubernetes job; however, for a federated job, you must send the request to the federation API server instead of sending it to a specific Kubernetes cluster. The federation control plane ensures that whenever the federated job is updated, it updates the corresponding job in all underlying clusters to match it.

If your update includes a change in number of parallelism and completions, the federation control plane changes the number of parallelism and completions in underlying clusters to ensure that their sum remains equal to the number of desired parallelism and completions in federated job.

#### Deleting a federated job

You can delete a federated job as you would delete a Kubernetes job; however, for a federated job, you must send the request to the federation API server instead of sending it to a specific Kubernetes cluster.

For example, with kubectl:

```shell
kubectl --context=federation-cluster delete job myjob
```

>   **Note:** Deleting a federated job will not delete the corresponding jobs from underlying clusters. You must delete the underlying jobs manually.





### 联合命名空间

本指南介绍如何在联邦控制平面中使用命名空间。

联邦控制平面中的命名空间（在本指南中称为“联邦命名空间”）与提供相同功能的传统 [Kubernetes 命名空间](https://v1-14.docs.kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)非常相似。在联邦控制平面中创建它们可确保它们在联邦中的所有集群之间同步。

#### 准备开始

-   本指南假设您已安装有一个正在运行的 Kubernetes 集群联邦。如果没有，那么请转到 [联邦管理指南](https://v1-14.docs.kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)，了解如何启动联邦集群(或者让集群管理员为您执行此操作)。 其他教程，例如 Kelsey Hightower 的[联邦 Kubernetes 教程](https://github.com/kelseyhightower/kubernetes-cluster-federation)， 也可能帮助您创建联邦 Kubernetes 集群。
-   一般来说您还应具备 [Kubernetes 的基本工作知识](https://v1-14.docs.kubernetes.io/docs/setup/pick-right-solution/)，尤其是[命名空间](https://v1-14.docs.kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)。

#### 创建联邦命名空间

联邦命名空间的 API 与传统 Kubernetes 命名空间的 API 100％兼容。您可以通过给联邦 API 服务器发送请求来创建一个命名空间。

您可以通过运行以下 kubectl 命令来执行此操作：

```shell
kubectl --context=federation-cluster create -f myns.yaml
```

参数 `--context=federation-cluster` 用于告知 kubectl 要向联邦 API 服务器提交请求而不是 Kubernetes 集群。

一旦联邦命名空间被创建，联邦控制平面将在所有基础的 Kubernetes 集群中创建与之相匹配的命名空间。 您可以通过检查每个基础集群来验证这一点，例如：

```shell
kubectl --context=gce-asia-east1a get namespaces myns
```

以上假设您在客户机中为该区域的集群配置了一个名为 ‘gce-asia-east1a’ 的上下文。基础命名空间的名称和 spec 将与您在上面创建的联合命名空间的名称和 spec 相匹配。

#### 更新联邦命名空间

您可以像更新 Kubernetes 命名空间一样更新联邦命名空间，只需要将请求发送给联邦的 API 服务器，而不是发送给特定的 Kubernetes 集群。 联邦控制平面将确保每当更新联邦命名空间时，它都会更新所有基础集群中的相应命名空间以与其匹配。

#### 删除联邦命名空间

您可以删除联邦命名空间就像删除 Kubernetes 命名空间一样，只需要将请求发送给联邦的 API 服务器，而不是发送给特定的 Kubernetes 集群。

例如，您可以通过运行以下 kubectl 命令来执行此操作：

```shell
kubectl --context=federation-cluster delete ns myns
```

与在 Kubernetes 中一样，删除联邦命名空间将从联邦控制平面中删除该命名空间中的所有资源。

>   **Note:** 就此，删除联邦命名空间不会从基础集群中删除相应的命名空间或这些命名空间中的资源。用户必须手动删除它们。我们打算在将来解决这个问题。





### 联合副本集(ReplicaSets)

This guide explains how to use ReplicaSets in the Federation control plane.

ReplicaSets in the federation control plane (referred to as “federated ReplicaSets” in this guide) are very similar to the traditional [Kubernetes ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/), and provide the same functionality. Creating them in the federation control plane ensures that the desired number of replicas exist across the registered clusters.

#### Before you begin

-   This guide assumes that you have a running Kubernetes Cluster Federation installation. If not, then head over to the [federation admin guide](https://kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/) to learn how to bring up a cluster federation (or have your cluster administrator do this for you). Other tutorials, such as Kelsey Hightower’s [Federated Kubernetes Tutorial](https://github.com/kelseyhightower/kubernetes-cluster-federation), might also help you create a Federated Kubernetes cluster.
-   You should also have a basic [working knowledge of Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/) in general and [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) in particular.

#### Creating a Federated ReplicaSet

The API for Federated ReplicaSet is 100% compatible with the API for traditional Kubernetes ReplicaSet. You can create a ReplicaSet by sending a request to the federation apiserver.

You can do that using [kubectl](https://kubernetes.io/docs/user-guide/kubectl/) by running:

```shell
kubectl --context=federation-cluster create -f myrs.yaml
```

The `--context=federation-cluster` flag tells kubectl to submit the request to the Federation apiserver instead of sending it to a Kubernetes cluster.

Once a federated ReplicaSet is created, the federation control plane will create a ReplicaSet in all underlying Kubernetes clusters. You can verify this by checking each of the underlying clusters, for example:

```shell
kubectl --context=gce-asia-east1a get rs myrs
```

The above assumes that you have a context named ‘gce-asia-east1a’ configured in your client for your cluster in that zone.

The ReplicaSets in the underlying clusters will match the federation ReplicaSet except in the number of replicas. The federation control plane will ensure that the sum of the replicas in each cluster match the desired number of replicas in the federation ReplicaSet.

##### Spreading Replicas in Underlying Clusters

By default, replicas are spread equally in all the underlying clusters. For example: if you have 3 registered clusters and you create a federated ReplicaSet with `spec.replicas = 9`, then each ReplicaSet in the 3 clusters will have `spec.replicas=3`. To modify the number of replicas in each cluster, you can add an annotation with key `federation.kubernetes.io/replica-set-preferences` to the federated ReplicaSet. The value of the annoation is a serialized JSON that contains fields shown in the following example:

```
{
  "rebalance": true,
  "clusters": {
    "foo": {
      "minReplicas": 10,
      "maxReplicas": 50,
      "weight": 100
    },
    "bar": {
      "minReplicas": 10,
      "maxReplicas": 100,
      "weight": 200
    }
  }
}
```

The `rebalance` boolean field specifies whether replicas already scheduled and running may be moved in order to match current state to the specified preferences. The `clusters` object field contains a map where users can specify the constraints for replica placement across the clusters (`foo` and `bar` in the example). For each cluster, you can specify the minimum number of replicas that should be assigned to it (default is zero), the maximum number of replicas the cluster can accept (default is unbounded) and a number expressing the relative weight of preferences to place additional replicas to that cluster.

#### Updating a Federated ReplicaSet

You can update a federated ReplicaSet as you would update a Kubernetes ReplicaSet; however, for a federated ReplicaSet, you must send the request to the federation apiserver instead of sending it to a specific Kubernetes cluster. The Federation control plane ensures that whenever the federated ReplicaSet is updated, it updates the corresponding ReplicaSet in all underlying clusters to match it. If your update includes a change in number of replicas, the federation control plane will change the number of replicas in underlying clusters to ensure that their sum remains equal to the number of desired replicas in federated ReplicaSet.

#### Deleting a Federated ReplicaSet

You can delete a federated ReplicaSet as you would delete a Kubernetes ReplicaSet; however, for a federated ReplicaSet, you must send the request to the federation apiserver instead of sending it to a specific Kubernetes cluster.

For example, you can do that using kubectl by running:

```shell
kubectl --context=federation-cluster delete rs myrs
```

>   **Note:** At this point, deleting a federated ReplicaSet will not delete the corresponding ReplicaSets from underlying clusters. You must delete the underlying ReplicaSets manually. We intend to fix this in the future.





### 联合Secrets(加密)

本指南解释了怎样使用联邦控制平面中的 secret。

联邦控制平面中的 secret (请参考本文的 “联邦 secrets” 章节) 和传统的[Kubernetes Secrets](https://v1-14.docs.kubernetes.io/docs/concepts/configuration/secret/)非常类似并提供相同的功能。 在联邦控制平面中创建它们可以确保它们在联邦中的所有集群之间同步。

#### 前提条件

本指南假设您安装好了 Kubernetes 集群联邦。如果没有，请先查看 [federation admin guide](https://v1-14.docs.kubernetes.io/docs/admin/federation/)来学习怎样安装集群联邦（也可以让您的管理员替您完成安装）。 其他的教程，例如 Kelsey Hightower 编写的 [这个教程](https://github.com/kelseyhightower/kubernetes-cluster-federation) 也会帮到你。

也希望您大概了解一些基本的 [Kubernetes 知识](https://v1-14.docs.kubernetes.io/docs/setup/) 特别是和 [Secrets](https://v1-14.docs.kubernetes.io/docs/concepts/configuration/secret/) 相关的知识。

#### 创建联邦 Secret

联邦 Secret 的 API 与传统的 Kubernetes Secret 的 API 100% 兼容。 你可以通过向联邦的 ApiServer 发出请求来创建 Secret。

```shell
kubectl --context=federation-cluster create -f mysecret.yaml
```

`--context=federation-cluster` 参数告诉 kubectl 将请求提交给联邦 ApiServer 而不是将其发送到 Kubernetes 集群上。

在创建完联邦 secret 后，联邦控制平面将在所有底层 Kubernetes 集群中创建匹配的 secret。 您可以通过检查每个底层集群来验证这一点，例如：

```shell
kubectl --context=gce-asia-east1a get secret mysecret
```

以上假设您在客户端上为该区域的集群配置了一个名为 ‘gce-asia-east1a’ 的上下文。

底层集群中的这些 secret 将与联邦 secret 相匹配。

#### 更新联邦 secret

您可以像更新 Kubernetes secret 那样更新联邦 secret；但是，对于联邦 secret，必须将请求发送到联邦 ApiServer，而非某个 Kubernetes 集群上。 联邦控制平面确保每当联邦 secret 被更新时，它来更新所有底层集群中的相应 secret ，以便相互匹配。

#### 删除联邦 Secret

您可以像删除 Kubernetes secret 那样删除联邦 secret；但是，对于联邦 secret，必须将请求发送到联邦 ApiServer，而不是将其发送到特定的 Kubernetes 集群。

例如，您可以使用 kubectl 来进行删除：

```shell
kubectl --context=federation-cluster delete secret mysecret
```

>   Note:
>
>   目前，删除联邦 secret 并不会从底层集群中删除相应的 secret。您必须手动删除底层 secret。我们考虑在将来解决这个问题。
