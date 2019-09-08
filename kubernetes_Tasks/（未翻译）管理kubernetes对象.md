# 管理kubernetes对象

## Declarative Management of Kubernetes Objects Using Configuration Files

Kubernetes objects can be created, updated, and deleted by storing multiple object configuration files in a directory and using `kubectl apply` to recursively create and update those objects as needed. This method retains writes made to live objects without merging the changes back into the object configuration files. `kubectl diff` also gives you a preview of what changes `apply` will make.

### Before you begin

安装 [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

一个集群

原文：

Install [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://kubernetes.io/docs/setup/minikube), or you can use one of these Kubernetes playgrounds:

-   [Katacoda](https://www.katacoda.com/courses/kubernetes/playground)
-   [Play with Kubernetes](http://labs.play-with-k8s.com/)

To check the version, enter `kubectl version`.

### Trade-offs

The `kubectl` tool supports three kinds of object management:

-   Imperative commands
-   Imperative object configuration
-   Declarative object configuration

See [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/) for a discussion of the advantages and disadvantage of each kind of object management.

### Overview

Declarative object configuration requires a firm understanding of the Kubernetes object definitions and configuration. Read and complete the following documents if you have not already:

-   [Managing Kubernetes Objects Using Imperative Commands](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)
-   [Imperative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)

Following are definitions for terms used in this document:

-   *object configuration file / configuration file*: A file that defines the configuration for a Kubernetes object. This topic shows how to pass configuration files to `kubectl apply`. Configuration files are typically stored in source control, such as Git.
-   *live object configuration / live configuration*: The live configuration values of an object, as observed by the Kubernetes cluster. These are kept in the Kubernetes cluster storage, typically etcd.
-   *declarative configuration writer / declarative writer*: A person or software component that makes updates to a live object. The live writers referred to in this topic make changes to object configuration files and run `kubectl apply` to write the changes.

### How to create objects

Use `kubectl apply` to create all objects, except those that already exist, defined by configuration files in a specified directory:

```shell
kubectl apply -f <directory>/
```

This sets the `kubectl.kubernetes.io/last-applied-configuration: '{...}'`annotation on each object. The annotation contains the contents of the object configuration file that was used to create the object.

>   **Note:** Add the `-R` flag to recursively process directories.

Here’s an example of an object configuration file:



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Run `kubectl diff` to print the object that will be created:

```shell
kubectl diff -f https://k8s.io/examples/application/simple_deployment.yaml
```

>   **Note:** `diff` uses [server-side dry-run](https://kubernetes.io/docs/reference/using-api/api-concepts/#dry-run), which needs to be enabled on `kube-apiserver`.

Create the object using `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the `kubectl.kubernetes.io/last-applied-configuration`annotation was written to the live configuration, and it matches the configuration file:

```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # This is the json representation of simple_deployment.yaml
    # It was written by kubectl apply when the object was created
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

### How to update objects

You can also use `kubectl apply` to update all objects defined in a directory, even if those objects already exist. This approach accomplishes the following:

1.  Sets fields that appear in the configuration file in the live configuration.
2.  Clears fields removed from the configuration file in the live configuration.

```shell
kubectl diff -f <directory>/
kubectl apply -f <directory>/
```

>   **Note:** Add the `-R` flag to recursively process directories.

Here’s an example configuration file:



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Create the object using `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

>   **Note:** For purposes of illustration, the preceding command refers to a single configuration file instead of a directory.

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the `kubectl.kubernetes.io/last-applied-configuration`annotation was written to the live configuration, and it matches the configuration file:

```yaml
kind: Deployment
metadata:
  annotations:
    # ...
    # This is the json representation of simple_deployment.yaml
    # It was written by kubectl apply when the object was created
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

Directly update the `replicas` field in the live configuration by using `kubectl scale`. This does not use `kubectl apply`:

```shell
kubectl scale deployment/nginx-deployment --replicas=2
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the `replicas` field has been set to 2, and the `last-applied-configuration` annotation does not contain a `replicas` field:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # note that the annotation does not contain replicas
    # because it was not updated through apply
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  replicas: 2 # written by scale
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

Update the `simple_deployment.yaml` configuration file to change the image from `nginx:1.7.9` to `nginx:1.11.9`, and delete the `minReadySeconds` field:



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.11.9 # update the image
        ports:
        - containerPort: 80
```

Apply the changes made to the configuration file:

```shell
kubectl diff -f https://k8s.io/examples/application/update_deployment.yaml
kubectl apply -f https://k8s.io/examples/application/update_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows the following changes to the live configuration:

-   The `replicas` field retains the value of 2 set by `kubectl scale`. This is possible because it is omitted from the configuration file.
-   The `image` field has been updated to `nginx:1.11.9` from `nginx:1.7.9`.
-   The `last-applied-configuration` annotation has been updated with the new image.
-   The `minReadySeconds` field has been cleared.
-   The `last-applied-configuration` annotation no longer contains the `minReadySeconds` field.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # The annotation contains the updated image to nginx 1.11.9,
    # but does not contain the updated replicas to 2
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.11.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
    # ...
spec:
  replicas: 2 # Set by `kubectl scale`.  Ignored by `kubectl apply`.
  # minReadySeconds cleared by `kubectl apply`
  # ...
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.11.9 # Set by `kubectl apply`
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

>   **Warning:** Mixing `kubectl apply` with the imperative object configuration commands `create`and `replace` is not supported. This is because `create` and `replace` do not retain the `kubectl.kubernetes.io/last-applied-configuration` that `kubectl apply` uses to compute updates.

### How to delete objects

There are two approaches to delete objects managed by `kubectl apply`.

#### Recommended: `kubectl delete -f <filename>`

Manually deleting objects using the imperative command is the recommended approach, as it is more explicit about what is being deleted, and less likely to result in the user deleting something unintentionally:

```shell
kubectl delete -f <filename>
```

#### Alternative: `kubectl apply -f <directory/> --prune -l your=label`

Only use this if you know what you are doing.

>   **Warning:** `kubectl apply --prune` is in alpha, and backwards incompatible changes might be introduced in subsequent releases.

>   **Warning:** You must be careful when using this command, so that you do not delete objects unintentionally.

As an alternative to `kubectl delete`, you can use `kubectl apply` to identify objects to be deleted after their configuration files have been removed from the directory. Apply with `--prune` queries the API server for all objects matching a set of labels, and attempts to match the returned live object configurations against the object configuration files. If an object matches the query, and it does not have a configuration file in the directory, and it has a `last-applied-configuration` annotation, it is deleted.

```shell
kubectl apply -f <directory/> --prune -l <labels>
```

>   **Warning:** Apply with prune should only be run against the root directory containing the object configuration files. Running against sub-directories can cause objects to be unintentionally deleted if they are returned by the label selector query specified with `-l <labels>` and do not appear in the subdirectory.

### How to view an object

You can use `kubectl get` with `-o yaml` to view the configuration of a live object:

```shell
kubectl get -f <filename|url> -o yaml
```

### How apply calculates differences and merges changes

>   **Caution:** A *patch* is an update operation that is scoped to specific fields of an object instead of the entire object. This enables updating only a specific set of fields on an object without reading the object first.

When `kubectl apply` updates the live configuration for an object, it does so by sending a patch request to the API server. The patch defines updates scoped to specific fields of the live object configuration. The `kubectl apply` command calculates this patch request using the configuration file, the live configuration, and the `last-applied-configuration`annotation stored in the live configuration.

#### Merge patch calculation

The `kubectl apply` command writes the contents of the configuration file to the `kubectl.kubernetes.io/last-applied-configuration` annotation. This is used to identify fields that have been removed from the configuration file and need to be cleared from the live configuration. Here are the steps used to calculate which fields should be deleted or set:

1.  Calculate the fields to delete. These are the fields present in `last-applied-configuration` and missing from the configuration file.
2.  Calculate the fields to add or set. These are the fields present in the configuration file whose values don’t match the live configuration.

Here’s an example. Suppose this is the configuration file for a Deployment object:



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.11.9 # update the image
        ports:
        - containerPort: 80
```

Also, suppose this is the live configuration for the same Deployment object:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # note that the annotation does not contain replicas
    # because it was not updated through apply
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"minReadySeconds":5,"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.7.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
  # ...
spec:
  replicas: 2 # written by scale
  # ...
  minReadySeconds: 5
  selector:
    matchLabels:
      # ...
      app: nginx
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        # ...
        name: nginx
        ports:
        - containerPort: 80
      # ...
```

Here are the merge calculations that would be performed by `kubectl apply`:

1.  Calculate the fields to delete by reading values from `last-applied-configuration` and comparing them to values in the configuration file. Clear fields explicitly set to null in the local object configuration file regardless of whether they appear in the `last-applied-configuration`. In this example, `minReadySeconds` appears in the `last-applied-configuration` annotation, but does not appear in the configuration file.**Action:** Clear `minReadySeconds` from the live configuration.
2.  Calculate the fields to set by reading values from the configuration file and comparing them to values in the live configuration. In this example, the value of `image` in the configuration file does not match the value in the live configuration. **Action:** Set the value of `image` in the live configuration.
3.  Set the `last-applied-configuration` annotation to match the value of the configuration file.
4.  Merge the results from 1, 2, 3 into a single patch request to the API server.

Here is the live configuration that is the result of the merge:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    # ...
    # The annotation contains the updated image to nginx 1.11.9,
    # but does not contain the updated replicas to 2
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",
      "metadata":{"annotations":{},"name":"nginx-deployment","namespace":"default"},
      "spec":{"selector":{"matchLabels":{"app":nginx}},"template":{"metadata":{"labels":{"app":"nginx"}},
      "spec":{"containers":[{"image":"nginx:1.11.9","name":"nginx",
      "ports":[{"containerPort":80}]}]}}}}
    # ...
spec:
  selector:
    matchLabels:
      # ...
      app: nginx
  replicas: 2 # Set by `kubectl scale`.  Ignored by `kubectl apply`.
  # minReadySeconds cleared by `kubectl apply`
  # ...
  template:
    metadata:
      # ...
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.11.9 # Set by `kubectl apply`
        # ...
        name: nginx
        ports:
        - containerPort: 80
        # ...
      # ...
    # ...
  # ...
```

#### How different types of fields are merged

How a particular field in a configuration file is merged with the live configuration depends on the type of the field. There are several types of fields:

-   *primitive*: A field of type string, integer, or boolean. For example, `image` and `replicas`are primitive fields. **Action:** Replace.
-   *map*, also called *object*: A field of type map or a complex type that contains subfields. For example, `labels`, `annotations`,`spec` and `metadata` are all maps. **Action:** Merge elements or subfields.
-   *list*: A field containing a list of items that can be either primitive types or maps. For example, `containers`, `ports`, and `args` are lists. **Action:** Varies.

When `kubectl apply` updates a map or list field, it typically does not replace the entire field, but instead updates the individual subelements. For instance, when merging the `spec` on a Deployment, the entire `spec` is not replaced. Instead the subfields of `spec`, such as `replicas`, are compared and merged.

#### Merging changes to primitive fields

Primitive fields are replaced or cleared.

>   **Note:** `-` is used for “not applicable” because the value is not used.

| Field in object configuration file | Field in live object configuration | Field in last-applied-configuration | Action                                |
| :--------------------------------- | :--------------------------------- | :---------------------------------- | :------------------------------------ |
| Yes                                | Yes                                | -                                   | Set live to configuration file value. |
| Yes                                | No                                 | -                                   | Set live to local configuration.      |
| No                                 | -                                  | Yes                                 | Clear from live configuration.        |
| No                                 | -                                  | No                                  | Do nothing. Keep live value.          |

#### Merging changes to map fields

Fields that represent maps are merged by comparing each of the subfields or elements of the map:

>   **Note:** `-` is used for “not applicable” because the value is not used.

| Key in object configuration file | Key in live object configuration | Field in last-applied-configuration | Action                           |
| :------------------------------- | :------------------------------- | :---------------------------------- | :------------------------------- |
| Yes                              | Yes                              | -                                   | Compare sub fields values.       |
| Yes                              | No                               | -                                   | Set live to local configuration. |
| No                               | -                                | Yes                                 | Delete from live configuration.  |
| No                               | -                                | No                                  | Do nothing. Keep live value.     |

#### Merging changes for fields of type list

Merging changes to a list uses one of three strategies:

-   Replace the list if all its elements are primitives.
-   Merge individual elements in a list of complex elements.
-   Merge a list of primitive elements.

The choice of strategy is made on a per-field basis.

##### Replace the list if all its elements are primitives

Treat the list the same as a primitive field. Replace or delete the entire list. This preserves ordering.

**Example:** Use `kubectl apply` to update the `args` field of a Container in a Pod. This sets the value of `args` in the live configuration to the value in the configuration file. Any `args`elements that had previously been added to the live configuration are lost. The order of the `args` elements defined in the configuration file is retained in the live configuration.

```yaml
# last-applied-configuration value
    args: ["a", "b"]

# configuration file value
    args: ["a", "c"]

# live configuration
    args: ["a", "b", "d"]

# result after merge
    args: ["a", "c"]
```

**Explanation:** The merge used the configuration file value as the new list value.

##### Merge individual elements of a list of complex elements:

Treat the list as a map, and treat a specific field of each element as a key. Add, delete, or update individual elements. This does not preserve ordering.

This merge strategy uses a special tag on each field called a `patchMergeKey`. The `patchMergeKey` is defined for each field in the Kubernetes source code: [types.go](https://github.com/kubernetes/api/blob/d04500c8c3dda9c980b668c57abc2ca61efcf5c4/core/v1/types.go#L2747) When merging a list of maps, the field specified as the `patchMergeKey` for a given element is used like a map key for that element.

**Example:** Use `kubectl apply` to update the `containers` field of a PodSpec. This merges the list as though it was a map where each element is keyed by `name`.

```yaml
# last-applied-configuration value
    containers:
    - name: nginx
      image: nginx:1.10
    - name: nginx-helper-a # key: nginx-helper-a; will be deleted in result
      image: helper:1.3
    - name: nginx-helper-b # key: nginx-helper-b; will be retained
      image: helper:1.3

# configuration file value
    containers:
    - name: nginx
      image: nginx:1.10
    - name: nginx-helper-b
      image: helper:1.3
    - name: nginx-helper-c # key: nginx-helper-c; will be added in result
      image: helper:1.3

# live configuration
    containers:
    - name: nginx
      image: nginx:1.10
    - name: nginx-helper-a
      image: helper:1.3
    - name: nginx-helper-b
      image: helper:1.3
      args: ["run"] # Field will be retained
    - name: nginx-helper-d # key: nginx-helper-d; will be retained
      image: helper:1.3

# result after merge
    containers:
    - name: nginx
      image: nginx:1.10
      # Element nginx-helper-a was deleted
    - name: nginx-helper-b
      image: helper:1.3
      args: ["run"] # Field was retained
    - name: nginx-helper-c # Element was added
      image: helper:1.3
    - name: nginx-helper-d # Element was ignored
      image: helper:1.3
```

**Explanation:**

-   The container named “nginx-helper-a” was deleted because no container named “nginx-helper-a” appeared in the configuration file.
-   The container named “nginx-helper-b” retained the changes to `args` in the live configuration. `kubectl apply` was able to identify that “nginx-helper-b” in the live configuration was the same “nginx-helper-b” as in the configuration file, even though their fields had different values (no `args` in the configuration file). This is because the `patchMergeKey` field value (name) was identical in both.
-   The container named “nginx-helper-c” was added because no container with that name appeared in the live configuration, but one with that name appeared in the configuration file.
-   The container named “nginx-helper-d” was retained because no element with that name appeared in the last-applied-configuration.

##### Merge a list of primitive elements

As of Kubernetes 1.5, merging lists of primitive elements is not supported.

>   **Note:** Which of the above strategies is chosen for a given field is controlled by the `patchStrategy` tag in [types.go](https://github.com/kubernetes/api/blob/d04500c8c3dda9c980b668c57abc2ca61efcf5c4/core/v1/types.go#L2748) If no `patchStrategy` is specified for a field of type list, then the list is replaced.

### Default field values

The API server sets certain fields to default values in the live configuration if they are not specified when the object is created.

Here’s a configuration file for a Deployment. The file does not specify `strategy`:



```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

Create the object using `kubectl apply`:

```shell
kubectl apply -f https://k8s.io/examples/application/simple_deployment.yaml
```

Print the live configuration using `kubectl get`:

```shell
kubectl get -f https://k8s.io/examples/application/simple_deployment.yaml -o yaml
```

The output shows that the API server set several fields to default values in the live configuration. These fields were not specified in the configuration file.

```yaml
apiVersion: apps/v1
kind: Deployment
# ...
spec:
  selector:
    matchLabels:
      app: nginx
  minReadySeconds: 5
  replicas: 1 # defaulted by apiserver
  strategy:
    rollingUpdate: # defaulted by apiserver - derived from strategy.type
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate # defaulted apiserver
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx:1.7.9
        imagePullPolicy: IfNotPresent # defaulted by apiserver
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP # defaulted by apiserver
        resources: {} # defaulted by apiserver
        terminationMessagePath: /dev/termination-log # defaulted by apiserver
      dnsPolicy: ClusterFirst # defaulted by apiserver
      restartPolicy: Always # defaulted by apiserver
      securityContext: {} # defaulted by apiserver
      terminationGracePeriodSeconds: 30 # defaulted by apiserver
# ...
```

In a patch request, defaulted fields are not re-defaulted unless they are explicitly cleared as part of a patch request. This can cause unexpected behavior for fields that are defaulted based on the values of other fields. When the other fields are later changed, the values defaulted from them will not be updated unless they are explicitly cleared.

For this reason, it is recommended that certain fields defaulted by the server are explicitly defined in the configuration file, even if the desired values match the server defaults. This makes it easier to recognize conflicting values that will not be re-defaulted by the server.

**Example:**

```yaml
# last-applied-configuration
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

# configuration file
spec:
  strategy:
    type: Recreate # updated value
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

# live configuration
spec:
  strategy:
    type: RollingUpdate # defaulted value
    rollingUpdate: # defaulted value derived from type
      maxSurge : 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80

# result after merge - ERROR!
spec:
  strategy:
    type: Recreate # updated value: incompatible with rollingUpdate
    rollingUpdate: # defaulted value: incompatible with "type: Recreate"
      maxSurge : 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

**Explanation:**

1.  The user creates a Deployment without defining `strategy.type`.
2.  The server defaults `strategy.type` to `RollingUpdate` and defaults the `strategy.rollingUpdate` values.
3.  The user changes `strategy.type` to `Recreate`. The `strategy.rollingUpdate` values remain at their defaulted values, though the server expects them to be cleared. If the `strategy.rollingUpdate` values had been defined initially in the configuration file, it would have been more clear that they needed to be deleted.
4.  Apply fails because `strategy.rollingUpdate` is not cleared. The `strategy.rollingupdate` field cannot be defined with a `strategy.type` of `Recreate`.

Recommendation: These fields should be explicitly defined in the object configuration file:

-   Selectors and PodTemplate labels on workloads, such as Deployment, StatefulSet, Job, DaemonSet, ReplicaSet, and ReplicationController
-   Deployment rollout strategy

#### How to clear server-defaulted fields or fields set by other writers

Fields that do not appear in the configuration file can be cleared by setting their values to `null` and then applying the configuration file. For fields defaulted by the server, this triggers re-defaulting the values.

### How to change ownership of a field between the configuration file and direct imperative writers

These are the only methods you should use to change an individual object field:

-   Use `kubectl apply`.
-   Write directly to the live configuration without modifying the configuration file: for example, use `kubectl scale`.

#### Changing the owner from a direct imperative writer to a configuration file

Add the field to the configuration file. For the field, discontinue direct updates to the live configuration that do not go through `kubectl apply`.

#### Changing the owner from a configuration file to a direct imperative writer

As of Kubernetes 1.5, changing ownership of a field from a configuration file to an imperative writer requires manual steps:

-   Remove the field from the configuration file.
-   Remove the field from the `kubectl.kubernetes.io/last-applied-configuration`annotation on the live object.

### Changing management methods

Kubernetes objects should be managed using only one method at a time. Switching from one method to another is possible, but is a manual process.

>   **Note:** It is OK to use imperative deletion with declarative management.

#### Migrating from imperative command management to declarative object configuration

Migrating from imperative command management to declarative object configuration involves several manual steps:

1.  Export the live object to a local configuration file:

    ```shell
     kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
    ```

2.  Manually remove the `status` field from the configuration file.

    >   **Note:** This step is optional, as `kubectl apply` does not update the status field even if it is present in the configuration file.

3.  Set the `kubectl.kubernetes.io/last-applied-configuration` annotation on the object:

    ```shell
    kubectl replace --save-config -f <kind>_<name>.yaml
    ```

4.  Change processes to use `kubectl apply` for managing the object exclusively.

#### Migrating from imperative object configuration to declarative object configuration

1.  Set the `kubectl.kubernetes.io/last-applied-configuration` annotation on the object:

    ```shell
    kubectl replace --save-config -f <kind>_<name>.yaml
    ```

2.  Change processes to use `kubectl apply` for managing the object exclusively.

### Defining controller selectors and PodTemplate labels

>   **Warning:** Updating selectors on controllers is strongly discouraged.

The recommended approach is to define a single, immutable PodTemplate label used only by the controller selector with no other semantic meaning.

**Example:**

```yaml
selector:
  matchLabels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
```

### What's next

-   [Managing Kubernetes Objects Using Imperative Commands](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)
-   [Imperative Management of Kubernetes Objects Using Configuration Files](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)
-   [Kubectl Command Reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/)
-   [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/)





## Declarative Management of Kubernetes Objects Using Kustomize

[Kustomize](https://github.com/kubernetes-sigs/kustomize) is a standalone tool to customize Kubernetes objects through a [kustomization file](https://github.com/kubernetes-sigs/kustomize/blob/master/docs/glossary.md#kustomization).

Since 1.14, Kubectl also supports the management of Kubernetes objects using a kustomization file. To view Resources found in a directory containing a kustomization file, run the following command:

```shell
kubectl kustomize <kustomization_directory>
```

To apply those Resources, run `kubectl apply` with `--kustomize` or `-k` flag:

```shell
kubectl apply -k <kustomization_directory>
```

### Before you begin

安装 [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

一个集群

### Overview of Kustomize

Kustomize is a tool for customizing Kubernetes configurations. It has the following features to manage application configuration files:

-   generating resources from other sources
-   setting cross-cutting fields for resources
-   composing and customizing collections of resources

#### Generating Resources

ConfigMap and Secret hold config or sensitive data that are used by other Kubernetes objects, such as Pods. The source of truth of ConfigMap or Secret are usually from somewhere else, such as a `.properties` file or a ssh key file. Kustomize has `secretGenerator` and `configMapGenerator`, which generate Secret and ConfigMap from files or literals.

##### configMapGenerator

To generate a ConfigMap from a file, add an entry to `files` list in `configMapGenerator`. Here is an example of generating a ConfigMap with a data item from a file content.

```shell
# Create a application.properties file
cat <<EOF >application.properties
FOO=Bar
EOF

cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-1
  files:
  - application.properties
EOF
```

The generated ConfigMap can be checked by the following command:

```shell
kubectl kustomize ./
```

The generated ConfigMap is:

```yaml
apiVersion: v1
data:
  application.properties: |
    FOO=Bar
kind: ConfigMap
metadata:
  name: example-configmap-1-8mbdf7882g
```

ConfigMap can also be generated from literal key-value pairs. To generate a ConfigMap from a literal key-value pair, add an entry to `literals` list in configMapGenerator. Here is an example of generating a ConfigMap with a data item from a key-value pair.

```shell
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-2
  literals:
  - FOO=Bar
EOF
```

The generated ConfigMap can be checked by the following command:

```shell
kubectl kustomize ./
```

The generated ConfigMap is

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  name: example-configmap-2-g2hdhfc6tk
```

##### secretGenerator

You can generate Secrets from files or literal key-value pairs. To generate a Secret from a file, add an entry to `files` list in `secretGenerator`. Here is an example of generating a Secret with a data item from a file.

```shell
# Create a password.txt file
cat <<EOF >./password.txt
username=admin
password=secret
EOF

cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-1
  files:
  - password.txt
EOF
```

The generated Secret is as follows:

```yaml
apiVersion: v1
data:
  password.txt: dXNlcm5hbWU9YWRtaW4KcGFzc3dvcmQ9c2VjcmV0Cg==
kind: Secret
metadata:
  name: example-secret-1-t2kt65hgtb
type: Opaque
```

To generate a Secret from a literal key-value pair, add an entry to `literals` list in `secretGenerator`. Here is an example of generating a Secret with a data item from a key-value pair.

```shell
cat <<EOF >./kustomization.yaml
secretGenerator:
- name: example-secret-2
  literals:
  - username=admin
  - password=secret
EOF
```

The generated Secret is as follows:

```yaml
apiVersion: v1
data:
  password: c2VjcmV0
  username: YWRtaW4=
kind: Secret
metadata:
  name: example-secret-2-t52t6g96d8
type: Opaque
```

##### generatorOptions

The generated ConfigMaps and Secrets have a suffix appended by hashing the contents. This ensures that a new ConfigMap or Secret is generated when the content is changed. To disable the behavior of appending a suffix, one can use `generatorOptions`. Besides that, it is also possible to specify cross-cutting options for generated ConfigMaps and Secrets.

```shell
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: example-configmap-3
  literals:
  - FOO=Bar
generatorOptions:
  disableNameSuffixHash: true
  labels:
    type: generated
  annotations:
    note: generated
EOF
```

Run`kubectl kustomize ./` to view the generated ConfigMap:

```yaml
apiVersion: v1
data:
  FOO: Bar
kind: ConfigMap
metadata:
  annotations:
    note: generated
  labels:
    type: generated
  name: example-configmap-3
```

#### Setting cross-cutting fields

It is quite common to set cross-cutting fields for all Kubernetes resources in a project. Some use cases for setting cross-cutting fields:

-   setting the same namespace for all Resource
-   adding the same name prefix or suffix
-   adding the same set of labels
-   adding the same set of annotations

Here is an example:

```shell
# Create a deployment.yaml
cat <<EOF >./deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
EOF

cat <<EOF >./kustomization.yaml
namespace: my-namespace
namePrefix: dev-
nameSuffix: "-001"
commonLabels:
  app: bingo
commonAnnotations:
  oncallPager: 800-555-1212
resources:
- deployment.yaml
EOF
```

Run `kubectl kustomize ./` to view those fields are all set in the Deployment Resource:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    oncallPager: 800-555-1212
  labels:
    app: bingo
  name: dev-nginx-deployment-001
  namespace: my-namespace
spec:
  selector:
    matchLabels:
      app: bingo
  template:
    metadata:
      annotations:
        oncallPager: 800-555-1212
      labels:
        app: bingo
    spec:
      containers:
      - image: nginx
        name: nginx
```

#### Composing and Customizing Resources

It is common to compose a set of Resources in a project and manage them inside the same file or directory. Kustomize offers composing Resources from different files and applying patches or other customization to them.

##### Composing

Kustomize supports composition of different resources. The `resources` field, in the `kustomization.yaml` file, defines the list of resources to include in a configuration. Set the path to a resource’s configuration file in the `resources` list. Here is an example for an nginx application with a Deployment and a Service.

```shell
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a service.yaml file
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

# Create a kustomization.yaml composing them
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

The Resources from `kubectl kustomize ./` contains both the Deployment and the Service objects.

##### Customizing

On top of Resources, one can apply different customizations by applying patches. Kustomize supports different patching mechanisms through `patchesStrategicMerge` and `patchesJson6902`. `patchesStrategicMerge` is a list of file paths. Each file should be resolved to a [strategic merge patch](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/strategic-merge-patch.md). The names inside the patches must match Resource names that are already loaded. Small patches that do one thing are recommended. For example, create one patch for increasing the deployment replica number and another patch for setting the memory limit.

```shell
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a patch increase_replicas.yaml
cat <<EOF > increase_replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
EOF

# Create another patch set_memory.yaml
cat <<EOF > set_memory.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  template:
    spec:
      containers:
      - name: my-nginx
        resources:
        limits:
          memory: 512Mi
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
patchesStrategicMerge:
- increase_replicas.yaml
- set_memory.yaml
EOF
```

Run `kubectl kustomize ./` to view the Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        limits:
          memory: 512Mi
        name: my-nginx
        ports:
        - containerPort: 80
```

Not all Resources or fields support strategic merge patches. To support modifying arbitrary fields in arbitrary Resources, Kustomize offers applying [JSON patch](https://tools.ietf.org/html/rfc6902) through `patchesJson6902`. To find the correct Resource for a Json patch, the group, version, kind and name of that Resource need to be specified in `kustomization.yaml`. For example, increasing the replica number of a Deployment object can also be done through `patchesJson6902`.

```shell
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a json patch
cat <<EOF > patch.yaml
- op: replace
  path: /spec/replicas
  value: 3
EOF

# Create a kustomization.yaml
cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml

patchesJson6902:
- target:
    group: apps
    version: v1
    kind: Deployment
    name: my-nginx
  path: patch.yaml
EOF
```

Run `kubectl kustomize ./` to see the `replicas` field is updated:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: nginx
        name: my-nginx
        ports:
        - containerPort: 80
```

In addition to patches, Kustomize also offers customizing container images or injecting field values from other objects into containers without creating patches. For example, you can change the image used inside containers by specifying the new image in `images` field in `kustomization.yaml`.

```shell
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

cat <<EOF >./kustomization.yaml
resources:
- deployment.yaml
images:
- name: nginx
  newName: my.image.registry/nginx
  newTag: 1.4.0
EOF
```

Run `kubectl kustomize ./` to see that the image being used is updated:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - image: my.image.registry/nginx:1.4.0
        name: my-nginx
        ports:
        - containerPort: 80
```

Sometimes, the application running in a Pod may need to use configuration values from other objects. For example, a Pod from a Deployment object need to read the corresponding Service name from Env or as a command argument. Since the Service name may change as `namePrefix` or `nameSuffix` is added in the `kustomization.yaml` file. It is not recommended to hard code the Service name in the command argument. For this usage, Kustomize can inject the Service name into containers through `vars`.

```shell
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        command: ["start", "--host", "\$(MY_SERVICE_NAME)"]
EOF

# Create a service.yaml file
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF

cat <<EOF >./kustomization.yaml
namePrefix: dev-
nameSuffix: "-001"

resources:
- deployment.yaml
- service.yaml

vars:
- name: MY_SERVICE_NAME
  objref:
    kind: Service
    name: my-nginx
    apiVersion: v1
EOF
```

Run `kubectl kustomize ./` to see that the Service name injected into containers is `dev-my-nginx-001`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dev-my-nginx-001
spec:
  replicas: 2
  selector:
    matchLabels:
      run: my-nginx
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - command:
        - start
        - --host
        - dev-my-nginx-001
        image: nginx
        name: my-nginx
```

### Bases and Overlays

Kustomize has the concepts of **bases** and **overlays**. A **base** is a directory with a `kustomization.yaml`, which contains a set of resources and associated customization. A base could be either a local directory or a directory from a remote repo, as long as a `kustomization.yaml` is present inside. An **overlay** is a directory with a `kustomization.yaml` that refers to other kustomization directories as its `bases`. A **base**has no knowledge of an overlay and can be used in multiple overlays. An overlay may have multiple bases and it composes all resources from bases and may also have customization on top of them.

Here is an example of a base:

```shell
# Create a directory to hold the base
mkdir base
# Create a base/deployment.yaml
cat <<EOF > base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
EOF

# Create a base/service.yaml file
cat <<EOF > base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nginx
  labels:
    run: my-nginx
spec:
  ports:
  - port: 80
    protocol: TCP
  selector:
    run: my-nginx
EOF
# Create a base/kustomization.yaml
cat <<EOF > base/kustomization.yaml
resources:
- deployment.yaml
- service.yaml
EOF
```

This base can be used in multiple overlays. You can add different `namePrefix` or other cross-cutting fields in different overlays. Here are two overlays using the same base.

```shell
mkdir dev
cat <<EOF > dev/kustomization.yaml
bases:
- ../base
namePrefix: dev-
EOF

mkdir prod
cat <<EOF > prod/kustomization.yaml
bases:
- ../base
namePrefix: prod-
EOF
```

### How to apply/view/delete objects using Kustomize

Use `--kustomize` or `-k` in `kubectl` commands to recognize Resources managed by `kustomization.yaml`. Note that `-k` should point to a kustomization directory, such as

```shell
kubectl apply -k <kustomization directory>/
```

Given the following `kustomization.yaml`,

```shell
# Create a deployment.yaml file
cat <<EOF > deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      run: my-nginx
  replicas: 2
  template:
    metadata:
      labels:
        run: my-nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx
        ports:
        - containerPort: 80
EOF

# Create a kustomization.yaml
cat <<EOF >./kustomization.yaml
namePrefix: dev-
commonLabels:
  app: my-nginx
resources:
- deployment.yaml
EOF
```

Run the following command to apply the Deployment object `dev-my-nginx`:

```shell
> kubectl apply -k ./
deployment.apps/dev-my-nginx created
```

Run one of the following commands to view the Deployment object `dev-my-nginx`:

```shell
kubectl get -k ./
kubectl describe -k ./
```

Run the following command to delete the Deployment object `dev-my-nginx`:

```shell
> kubectl delete -k ./
deployment.apps "dev-my-nginx" deleted
```

### Kustomize Feature List

| Field                 | Type                                                         | Explanation                                                  |
| :-------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| namespace             | string                                                       | add namespace to all resources                               |
| namePrefix            | string                                                       | value of this field is prepended to the names of all resources |
| nameSuffix            | string                                                       | value of this field is appended to the names of all resources |
| commonLabels          | map[string]string                                            | labels to add to all resources and selectors                 |
| commonAnnotations     | map[string]string                                            | annotations to add to all resources                          |
| resources             | []string                                                     | each entry in this list must resolve to an existing resource configuration file |
| configmapGenerator    | [][ConfigMapArgs](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/types/kustomization.go#L195) | Each entry in this list generates a ConfigMap                |
| secretGenerator       | [][SecretArgs](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/types/kustomization.go#L201) | Each entry in this list generates a Secret                   |
| generatorOptions      | [GeneratorOptions](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/types/kustomization.go#L239) | Modify behaviors of all ConfigMap and Secret generator       |
| bases                 | []string                                                     | Each entry in this list should resolve to a directory containing a kustomization.yaml file |
| patchesStrategicMerge | []string                                                     | Each entry in this list should resolve a strategic merge patch of a Kubernetes object |
| patchesJson6902       | [][Json6902](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/patch/json6902.go#L23) | Each entry in this list should resolve to a Kubernetes object and a Json Patch |
| vars                  | [][Var](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/types/var.go#L31) | Each entry is to capture text from one resource’s field      |
| images                | [][Image](https://github.com/kubernetes-sigs/kustomize/blob/master/pkg/image/image.go#L23) | Each entry is to modify the name, tags and/or digest for one image without creating patches |
| configurations        | []string                                                     | Each entry in this list should resolve to a file containing [Kustomize transformer configurations](https://github.com/kubernetes-sigs/kustomize/tree/master/examples/transformerconfigs) |
| crds                  | []string                                                     | Each entry in this list should resolve to an OpenAPI definition file for Kubernetes types |

### What's next

-   [Kustomize](https://github.com/kubernetes-sigs/kustomize)
-   [Kubectl Book](https://kubectl.docs.kubernetes.io/)
-   [Kubectl Command Reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/)
-   [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/)





## Managing Kubernetes Objects Using Imperative Commands

Kubernetes objects can quickly be created, updated, and deleted directly using imperative commands built into the `kubectl` command-line tool. This document explains how those commands are organized and how to use them to manage live objects.

### Before you begin

安装 [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

一个集群

### Trade-offs

The `kubectl` tool supports three kinds of object management:

-   Imperative commands
-   Imperative object configuration
-   Declarative object configuration

See [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/object-management/) for a discussion of the advantages and disadvantage of each kind of object management.

### How to create objects

The `kubectl` tool supports verb-driven commands for creating some of the most common object types. The commands are named to be recognizable to users unfamiliar with the Kubernetes object types.

-   `run`: Create a new Deployment object to run Containers in one or more Pods.
-   `expose`: Create a new Service object to load balance traffic across Pods.
-   `autoscale`: Create a new Autoscaler object to automatically horizontally scale a controller, such as a Deployment.

The `kubectl` tool also supports creation commands driven by object type. These commands support more object types and are more explicit about their intent, but require users to know the type of objects they intend to create.

-   `create <objecttype> [<subtype>] <instancename>`

Some objects types have subtypes that you can specify in the `create` command. For example, the Service object has several subtypes including ClusterIP, LoadBalancer, and NodePort. Here’s an example that creates a Service with subtype NodePort:

```shell
kubectl create service nodeport <myservicename>
```

In the preceding example, the `create service nodeport` command is called a subcommand of the `create service` command.

You can use the `-h` flag to find the arguments and flags supported by a subcommand:

```shell
kubectl create service nodeport -h
```

### How to update objects

The `kubectl` command supports verb-driven commands for some common update operations. These commands are named to enable users unfamiliar with Kubernetes objects to perform updates without knowing the specific fields that must be set:

-   `scale`: Horizontally scale a controller to add or remove Pods by updating the replica count of the controller.
-   `annotate`: Add or remove an annotation from an object.
-   `label`: Add or remove a label from an object.

The `kubectl` command also supports update commands driven by an aspect of the object. Setting this aspect may set different fields for different object types:

-   `set` `<field>`: Set an aspect of an object.

>   **Note:** In Kubernetes version 1.5, not every verb-driven command has an associated aspect-driven command.

The `kubectl` tool supports these additional ways to update a live object directly, however they require a better understanding of the Kubernetes object schema.

-   `edit`: Directly edit the raw configuration of a live object by opening its configuration in an editor.
-   `patch`: Directly modify specific fields of a live object by using a patch string. For more details on patch strings, see the patch section in [API Conventions](https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#patch-operations).

### How to delete objects

You can use the `delete` command to delete an object from a cluster:

-   `delete <type>/<name>`

>   **Note:** You can use `kubectl delete` for both imperative commands and imperative object configuration. The difference is in the arguments passed to the command. To use `kubectl delete` as an imperative command, pass the object to be deleted as an argument. Here’s an example that passes a Deployment object named nginx:

```shell
kubectl delete deployment/nginx
```

### How to view an object

There are several commands for printing information about an object:

-   `get`: Prints basic information about matching objects. Use `get -h` to see a list of options.
-   `describe`: Prints aggregated detailed information about matching objects.
-   `logs`: Prints the stdout and stderr for a container running in a Pod.

### Using `set` commands to modify objects before creation

There are some object fields that don’t have a flag you can use in a `create` command. In some of those cases, you can use a combination of `set` and `create` to specify a value for the field before object creation. This is done by piping the output of the `create` command to the `set` command, and then back to the `create` command. Here’s an example:

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run | kubectl set selector --local -f - 'environment=qa' -o yaml | kubectl create -f -
```

1.  The `kubectl create service -o yaml --dry-run` command creates the configuration for the Service, but prints it to stdout as YAML instead of sending it to the Kubernetes API server.
2.  The `kubectl set selector --local -f - -o yaml` command reads the configuration from stdin, and writes the updated configuration to stdout as YAML.
3.  The `kubectl create -f -` command creates the object using the configuration provided via stdin.

### Using `--edit` to modify objects before creation

You can use `kubectl create --edit` to make arbitrary changes to an object before it is created. Here’s an example:

```sh
kubectl create service clusterip my-svc --clusterip="None" -o yaml --dry-run > /tmp/srv.yaml
kubectl create --edit -f /tmp/srv.yaml
```

1.  The `kubectl create service` command creates the configuration for the Service and saves it to `/tmp/srv.yaml`.
2.  The `kubectl create --edit` command opens the configuration file for editing before it creates the object.

### What's next

-   [Managing Kubernetes Objects Using Object Configuration (Imperative)](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-config/)
-   [Managing Kubernetes Objects Using Object Configuration (Declarative)](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
-   [Kubectl Command Reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/)
-   [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/)





## Imperative Management of Kubernetes Objects Using Configuration Files

Kubernetes objects can be created, updated, and deleted by using the `kubectl` command-line tool along with an object configuration file written in YAML or JSON. This document explains how to define and manage objects using configuration files.

### Before you begin

安装 [`kubectl`](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

一个集群

### Trade-offs

The `kubectl` tool supports three kinds of object management:

-   Imperative commands
-   Imperative object configuration
-   Declarative object configuration

See [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/object-management-kubectl/overview/) for a discussion of the advantages and disadvantage of each kind of object management.

### How to create objects

You can use `kubectl create -f` to create an object from a configuration file. Refer to the [kubernetes API reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/) for details.

-   `kubectl create -f <filename|url>`

### How to update objects

>   **Warning:** Updating objects with the `replace` command drops all parts of the spec not specified in the configuration file. This should not be used with objects whose specs are partially managed by the cluster, such as Services of type `LoadBalancer`, where the `externalIPs` field is managed independently from the configuration file. Independently managed fields must be copied to the configuration file to prevent `replace` from dropping them.

You can use `kubectl replace -f` to update a live object according to a configuration file.

-   `kubectl replace -f <filename|url>`

### How to delete objects

You can use `kubectl delete -f` to delete an object that is described in a configuration file.

-   `kubectl delete -f <filename|url>`

### How to view an object

You can use `kubectl get -f` to view information about an object that is described in a configuration file.

-   `kubectl get -f <filename|url> -o yaml`

The `-o yaml` flag specifies that the full object configuration is printed. Use `kubectl get -h` to see a list of options.

### Limitations

The `create`, `replace`, and `delete` commands work well when each object’s configuration is fully defined and recorded in its configuration file. However when a live object is updated, and the updates are not merged into its configuration file, the updates will be lost the next time a `replace` is executed. This can happen if a controller, such as a HorizontalPodAutoscaler, makes updates directly to a live object. Here’s an example:

1.  You create an object from a configuration file.
2.  Another source updates the object by changing some field.
3.  You replace the object from the configuration file. Changes made by the other source in step 2 are lost.

If you need to support multiple writers to the same object, you can use `kubectl apply` to manage the object.

### Creating and editing an object from a URL without saving the configuration

Suppose you have the URL of an object configuration file. You can use `kubectl create --edit` to make changes to the configuration before the object is created. This is particularly useful for tutorials and tasks that point to a configuration file that could be modified by the reader.

```shell
kubectl create -f <url> --edit
```

### Migrating from imperative commands to imperative object configuration

Migrating from imperative commands to imperative object configuration involves several manual steps.

1.  Export the live object to a local object configuration file:

```shell
kubectl get <kind>/<name> -o yaml > <kind>_<name>.yaml
```

1.  Manually remove the status field from the object configuration file.
2.  For subsequent object management, use `replace` exclusively.

```shell
kubectl replace -f <kind>_<name>.yaml
```

### Defining controller selectors and PodTemplate labels

>   **Warning:** Updating selectors on controllers is strongly discouraged.

The recommended approach is to define a single, immutable PodTemplate label used only by the controller selector with no other semantic meaning.

Example label:

```yaml
selector:
  matchLabels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
template:
  metadata:
    labels:
      controller-selector: "extensions/v1beta1/deployment/nginx"
```

### What's next

-   [Managing Kubernetes Objects Using Imperative Commands](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/imperative-command/)
-   [Managing Kubernetes Objects Using Object Configuration (Declarative)](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/declarative-config/)
-   [Kubectl Command Reference](https://kubernetes.io/docs/reference/generated/kubectl/kubectl/)
-   [Kubernetes API Reference](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.15/)