# TLS

## 证书轮换

本文展示如何在 kubelet 中启用并配置证书轮换。

### 准备工作

-   要求 Kubernetes 1.8.0 或更高的版本
-   Kubelet 证书轮换在 1.8.0 版本中处于 beta 阶段, 这意味着该特性可能在没有通知的情况下发生变化。

### 概述

Kubelet 使用证书进行 Kubernetes API 的认证。 默认情况下，这些证书的签发期限为一年，所以不需要太频繁地进行更新。

Kubernetes 1.8 版本中包含 beta 特性 [kubelet 证书轮换](https://kubernetes.io/docs/tasks/administer-cluster/certificate-rotation/)， 在当前证书即将过期时， 将自动生成新的秘钥，并从 Kubernetes API 申请新的证书。 一旦新的证书可用，它将被用于与 Kubernetes API 间的连接认证。

### 启用客户端证书轮换

`kubelet` 进程接收 `--rotate-certificates` 参数，该参数决定 kubelet 在当前使用的证书即将到期时， 是否会自动申请新的证书。 由于证书轮换是 beta 特性，必须通过参数 `--feature-gates=RotateKubeletClientCertificate=true` 进行启用。

`kube-controller-manager` 进程接收 `--experimental-cluster-signing-duration` 参数，该参数控制证书签发的有效期限。

### 理解证书轮换配置

当 kubelet 启动时，如被配置为自举（使用`--bootstrap-kubeconfig` 参数），kubelet 会使用其初始证书连接到 Kubernetes API ，并发送证书签名的请求。 可以通过以下方式查看证书签名请求的状态：

```sh
kubectl get csr
```

最初，来自节点上 kubelet 的证书签名请求处于 `Pending` 状态。 如果证书签名请求满足特定条件， 控制器管理器会自动批准，此时请求会处于 `Approved` 状态。 接下来，控制器管理器会签署证书， 证书的有效期限由 `--experimental-cluster-signing-duration` 参数指定，签署的证书会被附加到证书签名请求中。

Kubelet 会从 Kubernetes API 取回签署的证书，并将其写入磁盘，存储位置通过 `--cert-dir`参数指定。 然后 kubelet 会使用新的证书连接到 Kubernetes API。

当签署的证书即将到期时，kubelet 会使用 Kubernetes API，发起新的证书签名请求。 同样地，控制器管理器会自动批准证书请求，并将签署的证书附加到证书签名请求中。 Kubelet 会从 Kubernetes API 取回签署的证书，并将其写入磁盘。 然后它会更新与 Kubernetes API 的连接，使用新的证书重新连接到 Kubernetes API。





## 在集群中管理TLS证书

每个 Kubernetes 集群都有一个集群根证书颁发机构（CA）。集群中的组件通常使用 CA 来验证 API server 的证书，由 API 服务器验证 kubelet 客户端证书等。为了支持这一点，CA 证书包被分发到集群中的每个节点，并作为一个 secret 附加分发到默认 service account 上。 或者，您的工作负载可以使用此 CA 建立信任。您的应用程序可以使用类似于 [ACME 草案](https://github.com/ietf-wg-acme/acme/)的协议，使用 `certificates.k8s.io` API 请求证书签名。

### 准备开始

一个集群。

### 集群中的 TLS 信任

让 Pod 中运行的应用程序信任集群根 CA 通常需要一些额外的应用程序配置。您将需要将 CA 证书包添加到 TLS 客户端或服务器信任的 CA 证书列表中。例如，您可以使用 golang TLS 配置通过解析证书链并将解析的证书添加到 [`tls.Config`](https://godoc.org/crypto/tls#Config) 结构中的 `RootCAs` 字段中。

CA 证书捆绑包将使用默认服务账户自动加载到 pod 中，路径为 `/var/run/secrets/kubernetes.io/serviceaccount/ca.crt`。如果您没有使用默认服务账户，请请求集群管理员构建包含您有权访问使用的证书包的 configmap。

### 请求认证

以下部分演示如何为通过 DNS 访问的 Kubernetes 服务创建 TLS 证书。

>   **Note:** 本教程使用 CFSSL：Cloudflare’s PKI 和 TLS 工具包[点击此处](https://blog.cloudflare.com/introducing-cfssl/)了解更多信息。

### 下载并安装 CFSSL

本例中使用的 cfssl 工具可以在 <https://pkg.cfssl.org/> 下载。

### 创建证书签名请求

通过运行以下命令生成私钥和证书签名请求（或 CSR）:

```shell
cat <<EOF | cfssl genkey - | cfssljson -bare server
{
  "hosts": [
    "my-svc.my-namespace.svc.cluster.local",
    "my-pod.my-namespace.pod.cluster.local",
    "172.168.0.24",
    "10.0.34.2"
  ],
  "CN": "my-pod.my-namespace.pod.cluster.local",
  "key": {
    "algo": "ecdsa",
    "size": 256
  }
}
EOF
```

其中 `172.168.0.24` 是服务的集群 IP，`my-svc.my-namespace.svc.cluster.local` 是服务的 DNS 名称，`10.0.34.2` 是 pod 的 IP 和 `my-pod.my-namespace.pod.cluster.local` 是 pod 的 DNS 名称。您能看到以下的输出：

```
2017/03/21 06:48:17 [INFO] generate received request
2017/03/21 06:48:17 [INFO] received CSR
2017/03/21 06:48:17 [INFO] generating key: ecdsa-256
2017/03/21 06:48:17 [INFO] encoded CSR
```

该命令生成两个文件；它生成包含 PEM 编码 [pkcs#10](https://tools.ietf.org/html/rfc2986) 认证请求的 `server.csr`，以及包含证书的 PEM 编码密钥的 `server-key.pem` 还有待生成。

### 创建证书签名请求对象发送到 Kubernetes API

使用以下命令创建 CSR yaml 文件，并发送到 API server：

```shell
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1beta1
kind: CertificateSigningRequest
metadata:
  name: my-svc.my-namespace
spec:
  groups:
  - system:authenticated
  request: $(cat server.csr | base64 | tr -d '\n')
  usages:
  - digital signature
  - key encipherment
  - server auth
EOF
```

请注意，在步骤1中创建的 `server.csr` 文件是 base64 编码并存储在 `.spec.request` 字段中的，我们还要求提供 “数字签名”，“密钥加密” 和 “服务器身份验证” 密钥用途的证书。我们[这里](https://godoc.org/k8s.io/api/certificates/v1beta1#KeyUsage)支持列出的所有关键用途和扩展的关键用途，以便您可以使用相同的 API 请求客户端证书和其他证书。

在 API server 中可以看到这些 CSR 处于 pending 状态。执行下面的命令您将可以看到：

```shell
kubectl describe csr my-svc.my-namespace
Name:                   my-svc.my-namespace
Labels:                 <none>
Annotations:            <none>
CreationTimestamp:      Tue, 21 Mar 2017 07:03:51 -0700
Requesting User:        yourname@example.com
Status:                 Pending
Subject:
        Common Name:    my-svc.my-namespace.svc.cluster.local
        Serial Number:
Subject Alternative Names:
        DNS Names:      my-svc.my-namespace.svc.cluster.local
        IP Addresses:   172.168.0.24
                        10.0.34.2
Events: <none>
```

### 获取批准的证书签名请求

批准证书签名请求是通过自动批准过程完成的，或由集群管理员一次性完成。有关这方面涉及的更多信息，请参见下文。

### 下载并使用证书

CSR 被签署并获得批准后，您应该看到以下内容：

```shell
kubectl get csr
NAME                  AGE       REQUESTOR               CONDITION
my-svc.my-namespace   10m       yourname@example.com    Approved,Issued
```

您可以通过运行以下命令下载颁发的证书并将其保存到 `server.crt` 文件中：

```shell
kubectl get csr my-svc.my-namespace -o jsonpath='{.status.certificate}' \
    | base64 --decode > server.crt
```

现在您可以将 `server.crt` 和 `server-key.pem` 作为键值对来启动 HTTPS 服务器。

### 批准证书签名请求

Kubernetes 管理员（具有适当权限）可以使用 `kubectl certificate approve` 和 `kubectl certificate deny` 命令手动批准（或拒绝）证书签名请求。但是，如果您打算大量使用此 API，则可以考虑编写自动化的证书控制器。

无论上述机器或人使用 kubectl，批准者的作用是验证 CSR 满足如下两个要求：

1.  CSR 的主体控制用于签署 CSR 的私钥。这解决了伪装成授权主体的第三方的威胁。在上述示例中，此步骤将验证该 pod 控制了用于生成 CSR 的私钥。
2.  CSR 的主体被授权在请求的上下文中执行。这解决了我们加入群集的我们不期望的主体的威胁。在上述示例中，此步骤将是验证该 pod 是否被允许加入到所请求的服务中。

当且仅当满足这两个要求时，审批者应该批准 CSR，否则拒绝 CSR。

### 关于批准许可的警告

批准 CSR 的能力决定谁信任群集中的谁。这包括 Kubernetes API 信任的人。批准 CSR 的能力不能过于广泛和轻率。在给予本许可之前，应充分了解上一节中提到的挑战和发布特定证书的后果。有关证书与认证交互的信息，请参阅[此处](https://v1-14.docs.kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certs)。

### 给集群管理员的一个建议

本教程假设将签名者设置为服务证书 API。Kubernetes controller manager 提供了一个签名者的默认实现。 要启用它，请将`--cluster-signing-cert-file` 和 `--cluster-signing-key-file` 参数传递给 controller manager，并配置具有证书颁发机构的密钥对的路径。