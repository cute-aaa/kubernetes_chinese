# 扩展kubernetes

## 使用自定义资源

### 使用自定义资源定义(Custom Resource Definition)来扩展Kubernetes API

这个页面展示了如何通过创建[CustomResourceDefinition](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#customresourcedefinition-v1beta1-apiextensions)将 [自定义资源](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/))安装到Kubernetes API中。

#### 准备工作

-   一个集群。

-   确保Kubernetes集群拥有1.7.0或更高版本的主版本。
-   阅读了 [自定义资源](https://kubernetes.io/docs/concepts/api-extension/custom-resources/).

#### 创建CustomResourceDefinition

当你创建一个新的 自定义资源定义 (CustomResourceDefinition, CRD) 时，kubernetes API 服务器会为你指定的每一个版本创建一个 RESTful 资源路径。CRD可以是命名空间的，也可以是集群范围的，可以在CRD的`scope`字段中指定。与现有内置对象一样，删除命名空间将删除该命名空间中的所有自定义对象。CRD本身不是命名空间范围的，而是对所有命名空间都可用。

比如，将下面的CRD保存到 `resourcedefinition.yaml`：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # list of versions supported by this CustomResourceDefinition
  versions:
    - name: v1
      # Each version can be enabled/disabled by Served flag.
      served: true
      # One and only one version must be marked as the storage version.
      storage: true
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
  preserveUnknownFields: false
  validation:
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            cronSpec:
              type: string
            image:
              type: string
            replicas:
              type: integer
```

然后创建它：

```shell
kubectl apply -f resourcedefinition.yaml
```

然后一个新的命名空间的 RESTful API 端点(endpoint)就会创建在：

```
/apis/stable.example.com/v1/namespaces/*/crontabs/...
```

然后这个这个端点URL就可以用来创建和管理自定义对象。这些对象的 `类别(kind)` 就是上面自定义资源对象中指定的  `CronTab` 。

创建端点需要花费数秒时间，你可以通过查看  `Established` 的状态是否变为true，或者查看资源的API server的发现信息是否显示出来。

#### 创建自定义对象

当自定义资源定义对象创建好之后，你就可以创建自定义对象(custom objects)了，自定义对象可以包含自定义字段。这些字段可以包含任意的JSON，下面的示例中，自定义的 `cronSpec` 和 `image` 字段在 `CronTab` 类别中被设置，`CronTab` 类别是上面自定义资源定义中指定的。

将下面的yaml保存为 `my-crontab.yaml`：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

然后创建：

```shell
kubectl apply -f my-crontab.yaml
```

可以使用kubectl来管理CronTab对象，像这样：

```shell
kubectl get crontab
```

应该会输出一个像这样的列表：

```console
NAME                 AGE
my-new-cron-object   6s
```

使用kubectl时资源名不区分大小写，并且使用CRD中定义的单数或复数字段都可以，也可以使用任何简写。

也可以查看原始YAML数据：

```shell
kubectl get ct -o yaml
```

你应该能看到在你用来创建的yaml中包含自定义的 `cronSpec` 和 `image` 字段：

```yaml
apiVersion: v1
kind: List
items:
- apiVersion: stable.example.com/v1
  kind: CronTab
  metadata:
    creationTimestamp: 2017-05-31T12:56:35Z
    generation: 1
    name: my-new-cron-object
    namespace: default
    resourceVersion: "285"
    selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
    uid: 9423255b-4600-11e7-af6a-28d2447dc82b
  spec:
    cronSpec: '* * * * */5'
    image: my-awesome-cron-image
metadata:
  resourceVersion: ""
  selfLink: ""
```

#### 删除CRD

当你删除一个CRD时，服务会卸载 RESTful API端点并且 **删除所有存储在其中的自定义对象**。

```shell
kubectl delete -f resourcedefinition.yaml
kubectl get crontabs
Error from server (NotFound): Unable to list {"stable.example.com" "v1" "crontabs"}: the server could not find the requested resource (get crontabs.stable.example.com)
```

即使你再创建一个同样的CRD，启动时里面也是空的。

#### 指定结构模式

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

传统的自定义资源，存储任意JSON(在`apiVersion`、`kind` 和 `metadata` 旁边，由API服务器隐式地验证)。通过 [OpenAPI v3.0 validation](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation)，可以指定模式，并在创建和更新过程中对其进行验证，比较一下下面的，注意每个模式的细节和限制。

在 `apiextensions.k8s.io/v1` 中，结构模式的定义是强制性的，而在 `v1beta1` 中依然是可选的。

结构模式是[OpenAPI v3.0验证模式(validation schema)](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation) ，它遵循以下规则：

1.  为root，和对于对象节点中每个使用 `properties` 或 `additionalProperties` 指明的对象，和数组节点中的每个使用 `items` 指明的项(item)，指明一个非空类型(通过 OpenAPI中的 `type` )，除非：
    -   节点设置了 `x-kubernetes-int-or-string: true` 
    -   节点设置了 `x-kubernetes-preserve-unknown-fields: true` 

2.  对象中的每个字段和数组中的每一项(item)，只要指定了 `allOf`, `anyOf`, `oneOf` 或 `not` 中的任意一个，模式同时指定那些逻辑连接符之外的字段/项(比较例1和例2)（意思就是要使用这几个连接符的话那么这几个逻辑连接符里指定的字段，在逻辑连接符之外也必须要有同名字段）。

3.  不要把 `description`, `type`, `default`, `additionProperties`, `nullable` 跟 `allOf`, `anyOf`, `oneOf` , `not` 设置在一起，除非使用 `x-kubernetes-int-or-string: true` 的两种特殊模式（见下文）。

4.  如果指明了 `metadata` ，那么只会允许限制 `metadata.name` 和 `metadata.generateName` 。

非结构化的例1：

```yaml
allOf:
- properties:
    foo:
      ...
```

与第二条规则冲突。设置成下面这样或许会正确：

```yaml
properties:
  foo:
    ...
allOf:
- properties:
    foo:
      ...
```

非结构化的例2：

```yaml
allOf:
- items:
    properties:
      foo:
        ...
```

与第二条规则冲突，设置成下面这样或许会正确：

```yaml
items:
  properties:
    foo:
      ...
allOf:
- items:
    properties:
      foo:
        ...
```

非结构化的例3：

```yaml
properties:
  foo:
    pattern: "abc"
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
      finalizers:
        type: array
        items:
          type: string
          pattern: "my-finalizer"
anyOf:
- properties:
    bar:
      type: integer
      minimum: 42
  required: ["bar"]
  description: "foo bar object"
```

不是结构模式，因为违反了下面这些：

-   没有设置root上的类型（规则1）
-   没有设置 `foo` 的类型（规则1）
-    `anyOf` 中的 `bar` 没有在外面声明（规则2）
-   `bar` 的 `type` 声明在了 `anyOf` 中（规则3）
-    `description` 声明在了 `anyOf` 中（规则3）
-   `metadata.finalizer` 可能不会被限制（规则4）

相比之下，下面的对应模式是结构化的：

```yaml
type: object
description: "foo bar object"
properties:
  foo:
    type: string
    pattern: "abc"
  bar:
    type: integer
  metadata:
    type: object
    properties:
      name:
        type: string
        pattern: "^a"
anyOf:
- properties:
    bar:
      minimum: 42
  required: ["bar"]
```

违反结构模式的规则会被记录在CRD的 `NonStructural` 中。

非结构模式会禁止使用以下功能：

-   [发布模式验证(Validation Schema Publishing)](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#publish-validation-schema-in-openapi-v2)
-   [网络钩子转化(Webhook Conversion)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/#webhook-conversion)
-   [默认模式验证(Validation Schema Defaulting)](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#defaulting)
-   [裁剪(Pruning)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#preserving-unknown-fields)

未来或许会有更多功能。

#### 裁剪与保存未知字段

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

像在etcd中一样，传统的CRD保存任何（或者说通过验证的）JSON。这意味着将持久化未指定的字段(如果存在OpenAPI v3.0模式验证)。这与本地kubernetes资源形成了对比，像示例中那样（e.g.）。在持久化到etcd之前，未知的字段被删除。我称这个为未知字段的“裁剪”。

在CRD中，如果在全局 `spec.validation.openAPIV3Schema` 或为每个版本定义了一个[结构化的 OpenAPI v3 验证模式](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#specifying-a-structural-schema) ，就可以通过设置 `spec.preserveUnknownFields` 为 `false` 来启用裁剪功能。这样在创建和更新时未指定的字段就会被删除。

与上面启用了裁剪的CRD `crontabs.stable.example.com` 进行比较。把下面的yaml保存为 `my-crontab.yaml`:

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  someRandomField: 42
```

创建：

```shell
kubectl create --validate=false -f my-crontab.yaml -o yaml
```

你会得到：

```yaml
apiVersion: stable.example.com/v1
kind: CronTab
metadata:
  creationTimestamp: 2017-05-31T12:56:35Z
  generation: 1
  name: my-new-cron-object
  namespace: default
  resourceVersion: "285"
  selfLink: /apis/stable.example.com/v1/namespaces/default/crontabs/my-new-cron-object
  uid: 9423255b-4600-11e7-af6a-28d2447dc82b
spec:
  cronSpec: '* * * * */5'
  image: my-awesome-cron-image
```

创建时使用的yaml中的 `someRandomField` 字段已经被裁剪了。

注意， `kubectl create` 调用使用 `--validate=false` 来跳过客户端验证。因为 [OpenAPI 验证模式也被发布(OpenAPI validation schemas are also published)](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#publish-validation-schema-in-openapi-v2) 到 kubectl，这也会验证未知字段并在这些对象发送到API服务器之前拒绝它们。

在 `apiextensions.k8s.io/v1beta1` 中，裁剪默认是关闭状态，也就是说（i.e.） `spec.preserveUnknownFields` 默认是 `true` 。在 `apiextensions.k8s.io/v1` 中，不允许创建带有 `spec.preserveUnknownFields: true` 的CRD。

#### 控制裁剪

在CRD中指定 `spec.preserveUnknownField: false` ，裁剪会对所有版本中的类型中的所有自定义资源开启，不过在[结构化 OpenAPI v3 验证模式(structural OpenAPI v3 validation schema)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#specifying-a-structural-schema)中通过设置 `x-kubernetes-preserve-unknown-fields: true` 可以选择让JSON子树退出规则：

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
```

 `json` 字段可以存储任何JSON值，且不会被裁剪。

可以指定一部分允许的JSON，比如：

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    description: this is arbitrary JSON
```

这样只允许对象类型的值。

对每个指定的属性(property)（或 `additionalProperties`）再次启用裁剪：

```yaml
type: object
properties:
  json:
    x-kubernetes-preserve-unknown-fields: true
    type: object
    properties:
      spec:
        type: object
        properties:
          foo:
            type: string
          bar:
            type: string
```

这样，值：

```yaml
json:
  spec:
    foo: abc
    bar: def
    something: x
  status:
    something: x
```

被裁剪为：

```yaml
json:
  spec:
    foo: abc
    bar: def
  status:
    something: x
```

这意味着指定的 `spec` 对象中的 `something` 字段被裁剪了，但所有外面的都没有被裁剪（spec里的something字段没有在上面指明所以被删除了）。

#### 整型还是字符串

使用了 `x-kubernetes-int-or-string: true` 不受规则1的限制，比如下面也是结构化的：

```yaml
type: object
properties:
  foo:
    x-kubernetes-int-or-string: true
```

此外，这些节点被部分排除在规则3之外（一部分不受规则3限制），因为规则3允许以下两种模式(正是这些模式，没有因为附加字段而产生变化)（规则3限制anyOf里不能有type属性）：

```yaml
x-kubernetes-int-or-string: true
anyOf:
- type: integer
- type: string
...
```

和

```yaml
x-kubernetes-int-or-string: true
allOf:
- anyOf:
  - type: integer
  - type: string
- ... # zero or more
...
```

使用其中一种规范，integer和string就都有效了。

在[发布验证模式](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#publish-validation-schema-in-openapi-v2)中，`x-kubernetes-int-or-string: true` 会被展开为上面的两种模式之一。

#### 原生扩展

原生扩展(在 [k8s.io/apimachinery](https://github.com/kubernetes/apimachinery/blob/03ac7a9ade429d715a1a46ceaa3724c18ebae54f/pkg/runtime/types.go#L94) 的 `runtime.RawExtension` 中定义) 拥有完整的kubernetes对象，也就是说，包含了 `apiVersion` 和 `kind` 字段。

通过设置 `x-kubernetes-embedded-resource: true` ，可以指定嵌入式对象（完全没有约束或者部分指定）。例如：

```yaml
type: object
properties:
  foo:
    x-kubernetes-embedded-resource: true
    x-kubernetes-preserve-unknown-fields: true
```

这里，`foo` 字段拥有完整的对象，比如：

```yaml
foo:
  apiVersion: v1
  kind: Pod
  spec:
    ...
```

因为在旁边指定了 `x-kubernetes-preserve-unknown-fields: true` ，所以不会进行裁剪。 `x-kubernetes-preserve-unknown-fields: true` 的使用是可选的。

使用 `x-kubernetes-embedded-resource: true`, 则 `apiVersion`, `kind` 和 `metadata` 会隐式地指定和验证。

#### 提供多个版本的CRD

详见 [自定义资源定义版本控制(Custom resource definition versioning)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/) 查看关于提供多个CRD版本和将对象从这个版本迁移到另一个版本的信息。

#### 高级主题

##### 终结器(Finalizers)

*终结器* 允许控制器实现异步的预删除资源钩子(asynchronous pre-delete hooks)。自定义对象也像内置对象一样支持终结器。

可以像这样为自定义对象添加一个终结器：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  finalizers:
  - finalizer.stable.example.com
```

终结器可以是任意的string值，他们存在时，确保资源不会被硬删除(hard delete)。

终结器的对象上的第一个删除请求会设置 `metadata.deletionTimestamp` 字段而不会删除它，当这个值被设置之后，终结器列表中的条目只能被删除。

当 `metadata.deletionTimestamp` 被设置之后，监视对象的控制器通过轮询该对象的更新请求来执行它们所处理的所有终结器（或者应翻译为 控制器通过轮询该对象的更新请求来监测对象执行了哪些请求？原文：controllers watching the object execute any finalizers they handle, by polling update requests for that object.）。当所有的终结器都被执行完毕，那么资源就会被删除。

 `metadata.deletionGracePeriodSeconds` 的值控制轮询更新的间隔（多久轮询一次）。

从列表中删除其终结器是每个控制器的任务。

kubernetes只有在终结器列表为空时才最终删除对象，这意味着所有终结器都已执行。

##### 验证

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

自定义对象的验证可以通过 [OpenAPI v3 模式(schema)](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#schemaObject) 或 [validatingadmissionwebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#validatingadmissionwebhook) 。此外，模式(schema)还被应用了以下限制：

-   不能设置这些字段：
    -   `definitions`,
    -   `dependencies`,
    -   `deprecated`,
    -   `discriminator`,
    -   `id`,
    -   `patternProperties`,
    -   `readOnly`,
    -   `writeOnly`,
    -   `xml`,
    -   `$ref`.
-    `uniqueItems` 字段不能设为true
-    `additionalProperties` 字段不能设为false
-    `additionalProperties` 字段与 `properties` 相互独立。

这些字段只能通过启用特定特征门来设置：

-   `default`: the `CustomResourceDefaulting` 特征门(feature gate)必须启用，对比[默认验证模式](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#defaulting).

注意：对于某些CRD特性所需的进一步限制，请与[结构化模式](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#specifying-a-structural-schema) 对比。

>   **注意：** OpenAPI v3 验证在beta版可用。需要开启 `CustomResourceValidation` 特征门，这在许多beta版集群来中自动启用的。详见 [特征门(feature gate)](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) 文档。

模式在CRD中定义。下面的例子中，CRD在自定义对象上启用了下面的验证：

-   `spec.cronSpec` 必须是string，而且必须是正则表达式形式。
-   `spec.replicas` 必须是integer，最小为1，最大为10。

将CRD保存为 `resourcedefinition.yaml`:

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
```

创建：

```shell
kubectl apply -f resourcedefinition.yaml
```

如果字段中有非法值，那么类别为 `CronTab` 的自定义对象的创建请求会被拒绝。下面的示例中，自定义对象的字段包含了非法值：

-   `spec.cronSpec` 不匹配正则表达式
-   `spec.replicas` 比10大

将下面的yaml保存为 `my-crontab.yaml` ：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * *"
  image: my-awesome-cron-image
  replicas: 15
```

创建：

```shell
kubectl apply -f my-crontab.yaml
```

会得到一个错误：

```console
The CronTab "my-new-cron-object" is invalid: []: Invalid value: map[string]interface {}{"apiVersion":"stable.example.com/v1", "kind":"CronTab", "metadata":map[string]interface {}{"name":"my-new-cron-object", "namespace":"default", "deletionTimestamp":interface {}(nil), "deletionGracePeriodSeconds":(*int64)(nil), "creationTimestamp":"2017-09-05T05:20:07Z", "uid":"e14d79e7-91f9-11e7-a598-f0761cb232d1", "selfLink":"", "clusterName":""}, "spec":map[string]interface {}{"cronSpec":"* * * *", "image":"my-awesome-cron-image", "replicas":15}}:
validation failure list:
spec.cronSpec in body should match '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
spec.replicas in body should be less than or equal to 10
```

如果字段包含合法值，对象的创建请求会被同意。

将下面的yaml保存为 `my-crontab.yaml`：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 5
```

创建：

```shell
kubectl apply -f my-crontab.yaml
crontab "my-new-cron-object" created
```

##### 默认值（Defaulting）

**FEATURE STATE:** `Kubernetes v1.15` [alpha](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

>   **注意：**
>
>   默认值在1.15之后的alpha版本中可用。默认关闭，可以通过 `CustomResourceDefaulting` 特征门(feature gate)来启用。详见 [特征门(feature gate)](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) 文档。
>
>   默认值也需要结构化模式和裁剪。

默认值允许在 [OpenAPI v3 验证模式(validation schema)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#validation) 中指定一个默认值：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  version: v1
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  preserveUnknownFields: false
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      type: object
      properties:
        spec:
          type: object
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
              default: "5 0 * * *"
            image:
              type: string
            replicas:
              type: integer
              minimum: 1
              maximum: 10
              default: 1
```

这时 `cronSpec` 和 `replicas` 都是默认的：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  image: my-awesome-cron-image
```

引出：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "5 0 * * *"
  image: my-awesome-cron-image
  replaces: 1
```

注意，默认值针对对象

-   使用默认的请求版本向API server发送请求
-   使用默认的存储版本从etcd读取
-   使用默认的admission webhook对象版本修改带有非空补丁的admission插件之后

注意，从etcd读取数据时应用的默认值不会自动写回etcd。需要通过API发出更新请求才能将这些默认值持久化到etcd中。

##### 在 OpenAPI v2 中发布验证模式

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

>   **Note:** 发布OpenAPI v2 在1.15的beta版本或1.14的alpha版本之后可用。 `CustomResourcePublishOpenAPI` 特性必须开启，在许多beta集群中默认开启。详见[特征门文档](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/)。

启用OpenAPI v2 的发布特性开启之后，CRD OpenAPI v3验证模式将作为OpenAPI v2规范的一部分从Kubernetes API服务器发布。

[kubectl](https://kubernetes.io/docs/reference/kubectl/overview) 使用发布的模式对定制资源执行客户端验证(`kubectl create` 和 `kubectl apply`  )、模式解释(kubectl explain)。发布的模式还可以用于其他目的，比如客户端生成或文档。

OpenAPI v3验证模式被转换为OpenAPI v2模式，并出现在 [OpenAPI v2 规范(spec)](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#openapi-and-swagger-definitions) 的 `definitions` 和 `paths` 字段中。转换时会应用以下修改来保证向下兼容1.13之前的版本。这些修改可以防止kubectl过于严格导致拒绝它不理解却合法的OpenAPI模式。转换不会修改CRD中定义的验证模式，因此不会影响API server中的[验证 。

1.  由于OpenAPI v2不支持以下字段，因此将删除它们(在未来版本中，OpenAPI v3将在没有这些限制的情况下使用)
    -   移除了 `allOf`, `anyOf`, `oneOf` , `not` 字段
2.  如果设置了 `nullable: true` ，我们将删除 `type`, `nullable`, `items` , `properties` ，因为OpenAPI v2不能表示nullable。为了避免kubectl拒绝一个好的对象，这是必要的。

##### 额外的打印列

从kubernetes 1.11开始，kubectl使用服务端输出，由服务决定执行 `kubectl get` 命令时哪些列会被打印。你可以使用CRD来自定义这些列。下面的示例中添加了 `Spec`, `Replicas`,  `Age` 列。

1.  将CRD保存为 `resourcedefinition.yaml` 。

    ```yaml
      apiVersion: apiextensions.k8s.io/v1beta1
      kind: CustomResourceDefinition
      metadata:
        name: crontabs.stable.example.com
      spec:
        group: stable.example.com
        version: v1
        scope: Namespaced
        names:
          plural: crontabs
          singular: crontab
          kind: CronTab
          shortNames:
          - ct
        additionalPrinterColumns:
        - name: Spec
          type: string
          description: The cron spec defining the interval a CronJob is run
          JSONPath: .spec.cronSpec
        - name: Replicas
          type: integer
          description: The number of jobs launched by the CronJob
          JSONPath: .spec.replicas
        - name: Age
          type: date
          JSONPath: .metadata.creationTimestamp
    ```

2.  创建CRD：

    ```shell
      kubectl apply -f resourcedefinition.yaml
    ```

3.  使用前一节中的 `my-crontab.yaml` 创建一个实例。

4.  调用服务端输出：

    ```shell
      kubectl get crontab my-new-cron-object
    ```

    注意输出中的 `NAME`, `SPEC`, `REPLICAS`,  `AGE` 列：

    ```
      NAME                 SPEC        REPLICAS   AGE
      my-new-cron-object   * * * * *   1          7s
    ```

 `NAME` 列是隐式的，不需要在CRD中定义。

###### 优先级（Priority）

每个列都包含了一个 `priority` 字段，优先级在标准视图或宽视图中显示的列之间有所区别(使用 `-o wide` 指定宽视图)。

-   标准视图会显示优先级为 `0` 的列
-   优先级大于0的列只在宽视图中显示

###### 类型（Type）

列的 `type` 字段可以是下面的任意一个：(对比 [OpenAPI v3 数据类型(data types)](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/3.0.0.md#dataTypes))

-   `integer` – 非浮点数
-   `number` – 浮点数
-   `string` – 字符串
-   `boolean` – true 或 false
-   `date` – 从时间戳以来以不同时间呈现（就是距时间戳的时间）

如果自定义资源(CustomResource)中的值不符合列中指定的类型，这个值会被忽略。使用自定义资源验证(CustomResource validation) 来确保值的类型是正确的。

###### 格式（format）

列的 `format` 字段可以是下面任意一个：

-   `int32`
-   `int64`
-   `float`
-   `double`
-   `byte`
-   `date`
-   `date-time`
-   `password`

 `format` 列控制 `kubectl` 输出值时使用的样式。

##### 子资源（Subresources）

**FEATURE STATE:** `Kubernetes v1.15` [beta](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#)

自定义资源支持 `/status` 和 `/scale` 子资源。

可以使用 [kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver) 上的 `CustomResourceSubresources` 特征门禁用此特性：

```sh
--feature-gates=CustomResourceSubresources=false
```

status子资源和scale子资源（状态子资源和缩放子资源）可以通过在CRD中定义来选择性启用。

###### 状态子资源（Status subresource）

启用状态子资源之后，将暴露自定义资源的 `/status` 子资源。

-   status和spec节(spec stanzas)分别由自定义资源中的 `.status` 和 `.spec` JSONPaths 表示。
-   对 `/status` 子资源的 `PUT` 请求只接受自定义资源对象，忽略除status节(status stanza)之外的任何更改。 
-   对 `/status` 子资源的 `PUT` 请求只验证自定义资源的status节(status stanza)。
-   对 `/status` 子资源的 `PUT` / `POST` / `PATCH` 请求忽略对状态节(status stanza)的更改。
-   除 `.metadata` 或 `.status` 的更改之外的所有更改都会使 `.metadata.generation` 的值自增。
-   在CRD OpenAPI验证模式的root下，只允许以下构造：
    -   Description
    -   Example
    -   ExclusiveMaximum
    -   ExclusiveMinimum
    -   ExternalDocs
    -   Format
    -   Items
    -   Maximum
    -   MaxItems
    -   MaxLength
    -   Minimum
    -   MinItems
    -   MinLength
    -   MultipleOf
    -   Pattern
    -   Properties
    -   Required
    -   Title
    -   Type
    -   UniqueItems

##### 缩放子资源（Scale subresource）

启用缩放子资源之后，会暴露自定义资源中的 `/scale` 子资源。 `autoscaling/v1.Scale` 对象作为 `/scale` 的有效负载发送。

要启用scale子资源，在CRD中定义以下值：

-   `SpecReplicasPath` 定义了自定义资源中对应于 `Scale.Spec.Replicas` 的JSONPath。
    -   必填值
    -   只允许 `.spec` 下的JSONPaths和带点符号的JSONPath
    -   如果自定义资源中的 `SpecReplicasPath` 下没有值， `/scale` 子资源将在GET上返回一个错误
-   `StatusReplicasPath` 定义了自定义资源中对应于 `Scale.Status.Replicas` 的JSONPath。
    -   必填值
    -   只允许 `.status` 下的JSONPaths和带点符号的JSONPath
    -   如果自定义资源中的 `StatusReplicasPath` 下没有值， `/scale` 子资源中的状态副本值(status replica value)默认为0
-   `LabelSelectorPath` 定义了自定义资源中对应于 `Scale.Status.Selector` 的JSONPath。
    -   可选值
    -   必须设置为与HPA一起工作
    -   只允许 `.status` 或 `.spec` 下的JSONPaths和带点符号的JSONPath
    -   如果自定义资源中的 `LabelSelectorPath` 下没有值，`/scale` 子资源中的状态选择器值将默认为空字符串。
    -   这个JSON路径指向的字段必须是一个包含一个字符串形式的序列化标签选择器的字符串字段(而不是一个复杂的选择器结构)。

在下面的示例中，同时启用了status子资源和scale子资源。

将下面的CRD保存为 `resourcedefinition.yaml`：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
  # subresources describes the subresources for custom resources.
  subresources:
    # status enables the status subresource.
    status: {}
    # scale enables the scale subresource.
    scale:
      # specReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Spec.Replicas.
      specReplicasPath: .spec.replicas
      # statusReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Replicas.
      statusReplicasPath: .status.replicas
      # labelSelectorPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Selector.
      labelSelectorPath: .status.labelSelector
```

创建：

```shell
kubectl apply -f resourcedefinition.yaml
```

当CRD对象创建完成之后，就可以创建自定义对象了。

将下面的yaml保存为 `my-crontab.yaml` ：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
  replicas: 3
```

创建：

```shell
kubectl apply -f my-crontab.yaml
```

新的命名空间级 RESTful API 端点(endpoint)创建在：

```
/apis/stable.example.com/v1/namespaces/*/crontabs/status
```

和

```
/apis/stable.example.com/v1/namespaces/*/crontabs/scale
```

自定义资源可以进行使用 `kubectl scale` 命令进行缩放(scale)。例如，下面的命令设置创建5个自定义资源的 `.spec.replicas` ：

```shell
kubectl scale --replicas=5 crontabs/my-new-cron-object
crontabs "my-new-cron-object" scaled

kubectl get crontabs my-new-cron-object -o jsonpath='{.spec.replicas}'
5
```

可以使用 [Pod中断处理(PodDisruptionBudget)](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/docs/tasks/run-application/configure-pdb/) 来保护启用了缩放子资源的自定义资源。

##### 分类（Categories）

分类是自定义资源所属的资源组的列表（比如 `all` ）。可以使用 `kubectl get <category-name>` 来列出属于该分类的资源。这个特性是 **beta**版，在 v1.10之后的自定义资源中可用。

下面的示例中在CRD中添加了分类列表中的 `all` ，并演示了如何使用 `kubectl get all` 输出自定义资源。

将下面的CRD保存为 `resourcedefinition.yaml`：

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: crontabs.stable.example.com
spec:
  group: stable.example.com
  versions:
    - name: v1
      served: true
      storage: true
  scope: Namespaced
  names:
    plural: crontabs
    singular: crontab
    kind: CronTab
    shortNames:
    - ct
    # categories is a list of grouped resources the custom resource belongs to.
    categories:
    - all
```

创建：

```shell
kubectl apply -f resourcedefinition.yaml
```

CRD对象创建之后，就可以创建自定义对象了。

将下面的yaml保存为 `my-crontab.yaml` ：

```yaml
apiVersion: "stable.example.com/v1"
kind: CronTab
metadata:
  name: my-new-cron-object
spec:
  cronSpec: "* * * * */5"
  image: my-awesome-cron-image
```

创建：

```shell
kubectl apply -f my-crontab.yaml
```

使用 `kubectl get` 来指定分类：

```
kubectl get all
```

然后会在类别 `CronTab` 中包含自定义资源：

```console
NAME                          AGE
crontabs/my-new-cron-object   3s
```

#### 下一步

-   查看 [CRD(CustomResourceDefinition)](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/#customresourcedefinition-v1beta1-apiextensions-k8s-io).
-   提供 [多个版本](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definition-versioning/) 的CRD







### 自定义资源定义(Custom ResourceDe finitions)的版本

#### 准备工作





## 配置聚合层

配置[聚合层](https://v1-14.docs.kubernetes.io/docs/concepts/api-extension/apiserver-aggregation/)允许 Kubernetes apiserver 使用其它 API 进行扩展，这些 API 不是核心 Kubernetes API 的一部分。

### 准备工作

一个集群。

>   **注意:** 在您的环境中启用聚合层时，如需要支持代理和扩展 apiserver 之间的双向 TLS 身份验证，需要满足一些设置要求。 Kubernetes 和 kube-apiserver 都有多个 CA，因此要确保代理的证书由聚合层 CA 签署，而不是由其它 CA （如主 CA）来签署。
>
>   >   **小心:** 为不同的客户机类型重用相同的CA会对集群的功能产生负面影响。详见[CA重用和冲突](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/#ca-reusage-and-conflicts).

### 身份验证流

不像自定义资源定义那样，聚合API（Aggregation API）除了标准Kubernetes apiserver之外，还包含了另一个服务器——您的扩展apiserver。Kubernetes apiserver您的扩展apiserver需要相互通信。为了保证这种通信，Kubernetes apiserver使用x509证书对扩展apiserver进行身份验证。

这一部分描述了身份验证和身份验证流是怎样工作的，要怎样配置他们。

高级别的流流程大致是这样：

1.  Kubernetes apiserver：对请求用户进行身份验证，并授权他们对请求的API路径的权限。
2.  Kubernetes apiserver:将请求代理到扩展apiserver。
3.  扩展apiserver:验证来自Kubernetes apiserver的请求
4.  扩展apiserver:授权来自原始用户的请求
5.  扩展apiserver:执行

本节的剩余部分将详细描述这些步骤。

可以在下图中看到该流程：

![aggregation auth flows](https://d33wubrfki0l68.cloudfront.net/3c5428678a95c3715894011d8dd4812d2cf229b9/e745c/images/docs/aggregation-api-auth-flow.png).

上面的swinlanes的源代码可以在本文档的源代码中找到。

#### Kubernetes Apiserver的身份验证和授权

对扩展apiserver提供服务的API路径的请求与所有API请求以相同的方式开始：（就是说由扩展apiserver提供服务的API路径以与k8s通信开始，其他API请求也以与k8s通信开始，两者的开头是相同的）：与Kubernetes apiserver通信。该路径已经由扩展apiserver在Kubernetes apiserver中注册。

用户与kubernetes apiserver通信，请求访问某个路径，kubernetes apiserver使用配置的标准的身份验证和授权来对用户进行身份验证并授权访问指定路径。

有关对Kubernetes集群进行身份验证的概述，见[“对集群进行身份验证”](https://kubernetes.io/docs/reference/access-authn-authz/authentication/). 有关对kubernetes集群资源进行授权的概述，见[“授权概述”](https://kubernetes.io/docs/reference/access-authn-authz/authorization/).

现在，一切都是标准的kubernetes API 请求，身份验证，授权。

kubernetes apiserver已经准备好给扩展apiserver发送请求了。

#### Kubernetes Apiserver 代理请求

现在kubernetes apiserver将会发送或代理请求到已经注册处理请求的扩展apiserver。为了做到这一点，你需要知道以下几点：

1.  Kubernetes apiserver应该如何对扩展apiserver进行身份验证，告诉扩展apiserver来自网络的请求来自一个有效的Kubernetes apiserver?
2.  Kubernetes apiserver应该如何告诉扩展apiserver请求的组和用户已经通过验证了呢？

为了保证上面两点，你必须为kubernetes apiserver配置一些标记。

##### Kubernetes Apiserver 客户端身份验证

kubernetes apiserver使用TLS来连接到扩展apiserver，并使用客户端证书对自己进行身份验证。您必须在启动时为Kubernetes apiserver提供以下标志：

-   私钥文件： `--proxy-client-key-file`
-   已签名的客户端证书 `--proxy-client-cert-file`
-   用来对客户端证书签名的CA证书 `--requestheader-client-ca-file`
-   已签名证书中有效的公用名称（CN） `--requestheader-allowed-names`

Kubernetes apiserver 会使用所有由 `--proxy-client-*-file` 指定的证书对扩展apiserver进行验证。为了让符合标准的扩展apiserver认为请求是有效的，必须满足以下条件：

1.  连接必须使用由 `--requestheader-client-ca-file` 中指定的CA进行签名的客户端证书。
2.  连接中客户端证书的CN必须是 `--requestheader-allowed-names` 中列出的。**Note:** 可以将此选项设为空： `--requestheader-allowed-names=""` 。这将向扩展apiserver表明任何CN都是可接受的。

使用这些选项启动之后，kubernetes apiserver就会：

1.  用他们对扩展apiserver进行身份验证。
2.  在 `kube-system` 命名空间中创建一个名为 `extension-apiserver-authentication` 的configmap，里面会被用来放CA证书和允许的CN，扩展apiservers可以检索他们来验证请求。

注意，Kubernetes apiserver使用同一个的客户机证书对所有扩展apiserver进行身份验证。它不会为每个扩展apiserver都创建一个客户机证书，而是只创建一个用于Kubernetes apiserver身份验证的证书。然后所有扩展apiserver请求都使用这个证书。

##### Original Request Username and Group

kubernetes apiserver将请求代理到扩展apiserver时，会把原始请求中通过身份验证的用户名和组告知给扩展apiserver。这些信息包含在代理请求的http头中。你必须跟kubernetes apiserver说一下你要用到的名称。

-   保存用户名的头：`--requestheader-username-headers` 
-   保存组的头： `--requestheader-group-headers`
-   把前缀附加到所有额外头：`--requestheader-extra-headers-prefix`

这些头的名称也会放在 `extension-apiserver-authentication`configmap,中，以便扩展apiserver检索和使用他们。

#### 扩展apiserver 对请求进行身份验证

扩展apiserver在收到来自Kubernetes apiserver的代理请求后，必须验证请求是不是真的来自一个有效的身份验证代理，也就是Kubernetes apiserver。扩展apiserver使用以下方式进行验证。

1.  从 `kube-system` 的configmap中检索以下内容：

    -   客户端CA证书
    -   允许的CN列表
    -   用户名、组和额外信息他们仨的头名

2.  检查TLS连接是否使用客户端证书进行了身份验证，包括：

    -   已被CA签名，且CA与检索到的CA相匹配。
    -   有一个在允许的CN列表中的CN，除非列表为空，允许所有CN。
    -   在合适的头中提取用户名和组

如果上面都通过了，那么这个请求就是合法的身份验证代理(这里是Kubernetes apiserver)发出的有效代理请求。

注意，提供上述功能的责任由扩展apiserver实现。很多人都是使用默认的 `k8s.io/apiserver/`  包。有些可能提供使用命令行覆盖它的选项。

为了拥有检索configmap的权限，扩展apiserver需要适当的角色，在 `kube-system` 中有一个叫作 `extension-apiserver-authentication-reader` 的默认角色，可以分配这个角色。

#### 扩展apiserver授权给请求

扩展apiserver现在可以验证从头文件检索到的用户/组是否被授权执行给定的请求。通过给kubernetes apiserver发送一个标准的 [SubjectAccessReview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/) 请求来实现。

扩展apiserver想要提交 `SubjectAccessReview` 请求给kubernetes apiserver，需要正确的权限。kubernetes包含了一个拥有适当权限的名为 `system:auth-delegator` 的默认 `集群角色(ClusterRole)` ，可以授予到扩展apiserver的服务账户(service account)。

#### 扩展apiserver执行请求

如果 `SubjectAccessReview` 通过了，那么扩展apiserver就会执行这个请求。

### 启用kubernetes apiserver标志

使用 `kube-apiserver` 标志来启用聚合层。你的提供商可能已经关心了这一点。(原话：They may have already been taken care of by your provider.)

```sh
--requestheader-client-ca-file=<path to aggregator CA cert>
--requestheader-allowed-names=front-proxy-client
--requestheader-extra-headers-prefix=X-Remote-Extra-
--requestheader-group-headers=X-Remote-Group
--requestheader-username-headers=X-Remote-User
--proxy-client-cert-file=<path to aggregator proxy cert>
--proxy-client-key-file=<path to aggregator proxy key>
```

#### CA重用和冲突

Kubernetes apiserver 有两个客户端CA选项：

-   `--client-ca-file`
-   `--requestheader-client-ca-file`

他们是相互独立的，但如果使用不当可能会彼此冲突。

-   `--client-ca-file` ：当请求到达kubernetes apiserver，如果这个选项是启用状态，那么kubernetes apiserver会检查请求里的证书，如果是被 `--client-ca-file` 中的某个CA证书签名的，那么就会被当作合法请求。用户使用公共名称 `CN=` 的值，组则是 `O=` 的值。详见[关于TLS身份验证的文档 ](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#x509-client-certs) 。
-   `--requestheader-client-ca-file`: 当前请求到达kubernetes apiserver，如果这个选项是启用状态，那么kubernetes apiserver会检查请求里的证书，如果是被 `--requestheader-client-ca-file` 中的某个CA证书签名的，那么这个请求会被当做一个可能的合法请求。然后Kubernetes apiserver再检查公共名称 `CN=` 是否是 `--requestheader-allowed-names` 列表中的名称之一。如果是，请求被批准;如果不是，则请求被拒绝。

如果 *同时*  指定 `--client-ca-file` 和 `--requestheader-client-ca-file` ，那么请求会先检查 `--requestheader-client-ca-file` 的CA，然后才是 `--client-ca-file` 。一般情况下，不同的CA，不管是根CA(root CAs)还是中间CA(intermediate CAs)，都会用于每一个选项(are used for each of these options;)；常规客户端请求与 `--client-ca-file` 进行匹配，聚合请求与 `--requestheader-client-ca-file` 进行匹配。但如果都使用 *同样的*  CA，那么平常能匹配通过的客户端请求将会匹配失败，因为CA会去匹配 `--requestheader-client-ca-file` 中的CA，而公共名称  `CN=` **匹配不了**  `--requestheader-allowed-names` 中的任何一个CN，这将导致你的库班elets 和其他的控制组件以及最终用户(end-users)无法对Kubernetes apiserver进行身份验证。

因此，可以使用不同的CA证书，在 `--client-ca-file` 中对控制面板组件和最终用户进行授权，在 `--requestheader-client-ca-file` 中对聚合apiserver请求进行授权。

>   **警告：** **不要**在不同的上下文中重用CA，除非你知道这样做的风险或者知道保护CA的方法。

如果kube-proxy没有运行在运行API server的主机上（就是在主机上的API server没有运行的情况下运行了kube-proxy），那么你必须保证开启了系统开启了下面的 `kube-apiserver`  标志：

```sh
--enable-aggregator-routing=true
```

#### 注册API服务对象(APIService Object)

你可以动态配置将哪些客户端请求代理到扩展apiserver。下面是一个注册示例：

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: <name of the registration object>
spec:
  group: <API group name this extension apiserver hosts>
  version: <API version this extension apiserver hosts>
  groupPriorityMinimum: <priority this APIService for this group, see API documentation>
  versionPriority: <prioritizes ordering of this version within a group, see API documentation>
  service:
    namespace: <namespace of the extension apiserver service>
    name: <name of the extension apiserver service>
  caBundle: <pem encoded ca cert that signs the server cert used by the webhook>
```

##### 与扩展apiserver联系

当kubernetes apiserver决定一个请求需要被发送到扩展apiserver时，它需要知道怎样跟他联系。

`服务`节(stanza)是对扩展apiserver服务的引用。服务的命名空间和名称是必填的。端口是可选的，默认443。路径也是可选的，默认“/”。

下面是一个扩展apiserver的示例，它被配置为在子路径“/my-path”上的“1234”端口进行调用，并使用自定义CA包根据ServerName `my-service-name.my-service-namespace.svc` 来验证TLS连接。

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
...
spec:
  ...
  service:
    namespace: my-service-namespace
    name: my-service-name
    port: 1234
  caBundle: "Ci0tLS0tQk...<base64-encoded PEM bundle>...tLS0K"
...
```

### 下一步

-   [设置一个扩展apiserver](https://kubernetes.io/docs/tasks/access-kubernetes-api/setup-extension-api-server/) 来处理聚合层
-   更高级的概述，看 [使用聚合层来扩展kubernetes api](https://kubernetes.io/docs/concepts/api-extension/apiserver-aggregation/).
-   学习怎样 [使用自定义资源定义(Custom Resource Definitions)来扩展kubernetes api](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/) 。







## 设置一个扩展API服务器

设置一个扩展的 API server 来使用聚合层以让 Kubernetes apiserver 使用其它 API 进行扩展，这些 API 不是核心 Kubernetes API 的一部分。

### 准备工作

一个集群。

### 设置一个扩展的 api-server 来使用聚合层

以下步骤描述如何 *在一个高层次* 设置一个扩展的 apiserver。无论您使用的是 YAML 配置还是使用 API，这些步骤都适用。目前我们正在尝试区分出两者的区别。有关使用 YAML 配置的具体示例，您可以在 Kubernetes 库中查看 [sample-apiserver](https://github.com/kubernetes/sample-apiserver/blob/master/README.md)。

或者，您可以使用现有的第三方解决方案，例如 [apiserver-builder](https://github.com/Kubernetes-incubator/apiserver-builder/blob/master/README.md)，它将生成框架并自动执行以下所有步骤。

1.  确保启用了 APIService API（检查 `--runtime-config`）。默认应该是启用的，除非被特意关闭了。
2.  您可能需要制定一个 RBAC 规则，以允许您添加 APIService 对象，或让您的集群管理员创建一个。（由于 API 扩展会影响整个集群，因此不建议在实时集群中对 API 扩展进行测试/开发/调试）
3.  创建 Kubernetes 命名空间，扩展的 api-service 将运行在该命名空间中。
4.  创建（或获取）用来签署服务器证书的 CA 证书，扩展 api-server 中将使用该证书做 HTTPS 连接。
5.  为 api-server 创建一个服务端的证书（或秘钥）以使用 HTTPS。这个证书应该由上述的 CA 签署。同时应该还要有一个 Kube DNS 名称的 CN，这是从 Kubernetes 服务派生而来的，格式为 `<service name>.<service name namespace>.svc`。
6.  使用命名空间中的证书（或秘钥）创建一个 Kubernetes secret。
7.  为扩展 api-server 创建一个 Kubernetes deployment，并确保以卷的方式挂载了 secret。它应该包含对扩展 api-server 镜像的引用。Deployment 也应该在同一个命名空间中。
8.  确保您的扩展 apiserver 从该卷中加载了那些证书，并在 HTTPS 握手过程中使用它们。
9.  在您的命令空间中创建一个 Kubernetes service account。
10.  为资源允许的操作创建 Kubernetes 集群角色。
11.  以您命令空间中的 service account 创建一个 Kubernetes 集群角色绑定，绑定到您刚创建的角色上。
12.  以您命令空间中的 service account 创建一个 Kubernetes 集群角色绑定，绑定到 `system:auth-delegator` 集群角色，以将 auth 决策委派给 Kubernetes 核心 API 服务器。
13.  以您命令空间中的 service account 创建一个 Kubernetes 集群角色绑定，绑定到 `extension-apiserver-authentication-reader` 角色。这将让您的扩展 api-server 能够访问 `extension-apiserver-authentication` configmap。
14.  创建一个 Kubernetes apiservice。上述的 CA 证书应该使用 base64 编码，剥离新行并用作 apiservice 中的 spec.caBundle。这不应该是命名空间化的。如果使用了 [kube-aggregator API](https://github.com/kubernetes/kube-aggregator/)，那么只需要传入 PEM 编码的 CA 绑定，因为 base 64 编码已经完成了。
15.  使用 kubectl 来获得您的资源。它应该返回 “找不到资源”。这意味着一切正常，但您目前还没有创建该资源类型的对象。

### 接下来

-   如果你还未配置，请 [配置聚合层](https://v1-14.docs.kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/) 并启用 apiserver 的相关参数。
-   高级概述，请参阅 [使用聚合层扩展 Kubernetes API](https://v1-14.docs.kubernetes.io/docs/concepts/api-extension/apiserver-aggregation)。
-   了解如何 [使用 Custom Resource Definition 扩展 Kubernetes API](https://v1-14.docs.kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/)。





## 使用HTTP代理访问Kubernetes API

本文说明如何使用 HTTP 代理访问 Kubernetes API。

### 准备工作

-   一个集群。

-   如果您的集群中还没有任何应用，使用如下命令启动一个 Hello World 应用：

    ```shell
    kubectl run node-hello –image=gcr.io/google-samples/node-hello:1.0 –port=8080
    ```

### 使用 kubectl 启动代理服务器

使用如下命令启动 Kubernetes API 服务器的代理：

```
kubectl proxy --port=8080
```

### 探究 Kubernetes API

当代理服务器在运行时，你可以通过 `curl`、`wget` 或者浏览器访问 API。

获取 API 版本：

```
curl http://localhost:8080/api/

{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.2.15:8443"
    }
  ]
}
```

获取 Pod 列表：

```
curl http://localhost:8080/api/v1/namespaces/default/pods

{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/default/pods",
    "resourceVersion": "33074"
  },
  "items": [
    {
      "metadata": {
        "name": "kubernetes-bootcamp-2321272333-ix8pt",
        "generateName": "kubernetes-bootcamp-2321272333-",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/pods/kubernetes-bootcamp-2321272333-ix8pt",
        "uid": "ba21457c-6b1d-11e6-85f7-1ef9f1dab92b",
        "resourceVersion": "33003",
        "creationTimestamp": "2016-08-25T23:43:30Z",
        "labels": {
          "pod-template-hash": "2321272333",
          "run": "kubernetes-bootcamp"
        },
        ...
}
```

### 接下来

想了解更多信息，请参阅 [kubectl 代理](https://v1-14.docs.kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#proxy)。