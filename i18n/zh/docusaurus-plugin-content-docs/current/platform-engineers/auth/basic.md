---
title: Kubernetes RBAC
---

KubeVela 1.4 开始，我们加入了认证和授权的功能，这使得控制器可以严格根据使用者的权限去做应用的交付和管理，隔离不同的租户，让应用交付变得更安全。

## 安装

为了不影响之前的用户体验，在 1.4 版本中我们为多集群权限做了功能开关，默认不开启，你可以在安装或升级时指定开启，方法如下。

1. 删除原先的集群绑定权限 `vela-core:manager-rolebinding`，避免 KubeVela 控制器使用已有的高权限。
  ```
  kubectl delete ClusterRoleBinding kubevela-vela-core:manager-rolebinding
  ```

2. 升级控制器，开启用户权限认证功能：
  ```
  helm upgrade --install kubevela kubevela/vela-core --create-namespace -n vela-system --set authentication.enabled=true --set authentication.withUser=true --wait
  ```

3. 确保命令行工具 Vela CLI 版本为 v1.4.1+，参考[安装文档](../../install#2-install-kubevela-cli).

4. (可选) 安装 [vela-prism](https://github.com/kubevela/prism) 组件，开启高级的权限管理能力。
    ```bash
    helm repo add prism https://charts.kubevela.net/prism
    helm repo update
    helm install vela-prism prism/vela-prism -n vela-system
    ```

## 使用

### 准备好集群

在我们开始之前，请确保你已经有 2 个集群添加到管控中，假设命名为 `c2` 和 `c3`。你可以参考[多集群管理文档](../system-operation/managing-clusters#manage-the-cluster-via-cli) 查看如何添加集群。

```bash
$ vela cluster list
NAME    ALIAS   CREDENTIAL_TYPE   ENDPOINT                      ACCEPTED   LABELS
local           Internal          -                             true       
c3              X509Certificate   <c3 apiserver url>            true       
c2              X509Certificate   <c2 apiserver url>            true       
```

### 创建用户组（用户）

```
$ vela auth gen-kubeconfig --user alice > alice.kubeconfig
Private key generated.
Certificate request generated.
Certificate signing request alice generated.
Certificate signing request alice approved.
Signed certificate retrieved.
```

这里的 alice (--user指定) 既可以表示某个用户组，也可以代表某个用户。通常我们建议用来表示用户组，以便降低整体权限控制的复杂度。在 VelaUX 中则与“项目”这个概念对应，每一个项目创建一个独立的用户组权限，而 VelaUX 中支持使用 LDAP 对接公司的账号体系，可以对具体的某个用户账号授权，加入到某个项目中（即加入用户组），以此完成 VelaUX 终端用户和底层资源权限的打通。


### 对用户组授权

```
$ vela auth grant-privileges --user alice --for-namespace dev --for-cluster=c2 --create-namespace
ClusterRole kubevela:writer created in c2.
RoleBinding dev/kubevela:writer:binding created in c2.
Privileges granted.
```

这里采用了 KubeVela 简化的权限能力，对 c2 集群授权了 `dev` 命名空间的“读/写”权限，同时还可以方便的创建命名空间。授权命令可以多次执行，用于增加权限。


### 查看用户组权限

```
$ vela auth list-privileges --user alice --cluster local,c2
User=alice
├── [Cluster]  local
│   └── no privilege found
└── [Cluster]  c2
    └── [ClusterRole]  kubevela:writer
        ├── [Scope]  
        │   └── [Namespaced]  dev (RoleBinding kubevela:writer:binding)
        └── [PolicyRules]  
            ├── APIGroups:       *
            │   Resources:       *
            │   Verb:            get, list, watch, create, update, patch, delete
            └── NonResourceURLs: *
                Verb:            get, list, watch, create, update, patch, delete
```

你可以一目了然的看到这个用户组在不同集群中的权限，在 local 集群中无权限。


### 使用权限

创建用户组之后（`vela auth gen-kubeconfig`），会生成一个 KubeConfig 对应该权限，最终用户可以拿着这个 KubeConfig 使用 vela 命令行工具进行操作，生态的插件功能也可以通过这个 KubeConfig 跟 KubeVela 的 API 对接。 使用的方式与 KubeConfig 原先的用法完全一致，你可以将 KubeConfig 放到 `~/.kube/config` 中，也可以通过环境变量使用。

```bash
cat <<EOF | KUBECONFIG=alice.kubeconfig vela up -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: podinfo
  namespace: dev
spec:
  components:
  - name: podinfo
    type: webservice
    properties:
      image: stefanprodan/podinfo:6.0.1
  policies:
  - type: topology
    name: topology
    properties:
      clusters: ["c2"]
EOF
```

可以通过如下命令查看多集群下资源的部署状态：

```bash
$ KUBECONFIG=alice.kubeconfig vela status podinfo -n dev
About:

  Name:         podinfo                      
  Namespace:    dev                          
  Created at:   2022-05-31 17:06:14 +0800 CST
  Status:       running                      

Workflow:

  mode: DAG
  finished: true
  Suspend: false
  Terminated: false
  Steps
  - id:rk3npcpycl
    name:deploy-topology
    type:deploy
    phase:succeeded 
    message:

Services:

  - Name: podinfo  
    Cluster: c2  Namespace: dev
    Type: webservice
    Healthy Ready:1/1
    No trait applied
```

### 未授权的操作会被禁止

对于创建出的 KubeConfig，如果 Alice 访问 dev 这个命名空间以外的资源，服务端会禁止这个操作。

```bash
KUBECONFIG=alice.kubeconfig kubectl get pod -n vela-system
Error from server (Forbidden): pods is forbidden: User "alice" cannot list resource "pods" in API group "" in the namespace "vela-system"
```

Alice 使用 Application 创建涉及其他集群的资源也会被禁止，比如创建如下的应用  `podinfo-bad`：

```bash
$ cat <<EOF | KUBECONFIG=alice.kubeconfig vela up -f -
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: podinfo-bad
  namespace: dev
spec:
  components:
  - name: podinfo-bad
    type: webservice
    properties:
      image: stefanprodan/podinfo:6.0.1
  policies:
  - type: topology
    name: topology
    properties:
      clusters: ["c3"]
EOF
```

Alice在查看应用状态时会了解到错误情况：

```bash
$ KUBECONFIG=alice.kubeconfig vela status podinfo-bad -n dev
About:

  Name:         podinfo-bad                  
  Namespace:    dev                          
  Created at:   2022-05-31 17:09:16 +0800 CST
  Status:       runningWorkflow              

Workflow:

  mode: DAG
  finished: false
  Suspend: false
  Terminated: false
  Steps
  - id:tw539smx7m
    name:deploy-topology
    type:deploy
    phase:failed 
    message:step deploy: run step(provider=multicluster,do=deploy): Found 1 errors. [(error encountered in cluster c3: HandleComponentsRevision: controllerrevisions.apps is forbidden: User "alice" cannot list resource "controllerrevisions" in API group "apps" in the namespace "dev")]

```

### 只读权限

我们也可以给用户创建一个只读权限，这也是 KubeVela 封装好的预置权限，比如给用户 Bob 提供只读的 KubeConfig：

```bash
$ vela auth gen-kubeconfig --user bob > bob.kubeconfig
Private key generated.
Certificate request generated.
Certificate signing request bob generated.
Certificate signing request bob approved.
Signed certificate retrieved.

$ vela auth grant-privileges --user bob --for-namespace dev --for-cluster=local,c2 --readonly
ClusterRole kubevela:reader created in local.
RoleBinding dev/kubevela:reader:binding created in local.
ClusterRole kubevela:reader created in c2.
RoleBinding dev/kubevela:reader:binding created in c2.
Privileges granted.
```

用户 Bob 就可以看到 `dev` 这个命名空间的应用状态了。

```bash
$ KUBECONFIG=bob.kubeconfig vela ls -n dev
APP             COMPONENT       TYPE            TRAITS  PHASE                   HEALTHY STATUS          CREATED-TIME                 
podinfo         podinfo         webservice              running                 healthy Ready:1/1       2022-05-31 17:06:14 +0800 CST
podinfo-bad     podinfo-bad     webservice              workflowTerminated                              2022-05-31 17:09:16 +0800 CST

$ KUBECONFIG=bob.kubeconfig vela status podinfo -n dev
About:

  Name:         podinfo                      
  Namespace:    dev                          
  Created at:   2022-05-31 17:06:14 +0800 CST
  Status:       running                      

Workflow:

  mode: DAG
  finished: true
  Suspend: false
  Terminated: false
  Steps
  - id:rk3npcpycl
    name:deploy-topology
    type:deploy
    phase:succeeded 
    message:

Services:

  - Name: podinfo  
    Cluster: local  Namespace: dev
    Type: webservice
    Healthy Ready:1/1
    No trait applied

  - Name: podinfo  
    Cluster: c2  Namespace: dev
    Type: webservice
    Healthy Ready:1/1
    No trait applied
```

但是如果他想做其他操作，比如删除一个应用，就会被禁止：

```bash
$ KUBECONFIG=bob.kubeconfig vela delete podinfo-bad -n dev
Deleting Application "podinfo-bad"
Error: delete application err: applications.core.oam.dev "podinfo-bad" is forbidden: User "bob" cannot delete resource "applications" in API group "core.oam.dev" in the namespace "dev"
2022/05/31 17:17:51 delete application err: applications.core.oam.dev "podinfo-bad" is forbidden: User "bob" cannot delete resource "applications" in API group "core.oam.dev" in the namespace "dev"
```

而对于有权限的 Alice 来说，她可以删除应用：

```bash
$ KUBECONFIG=alice.kubeconfig vela delete podinfo-bad -n dev
application.core.oam.dev "podinfo-bad" deleted
```

### 查看应用的资源

如果用户想要细粒度的资源查看能力，就要安装之前提到的  `vela-prism` 组件了。安装完成后就可以使用如下命令查看资源关联关系和状态：

```bash
$ KUBECONFIG=bob.kubeconfig vela status podinfo -n dev --tree --detail
CLUSTER       NAMESPACE     RESOURCE           STATUS    APPLY_TIME          DETAIL
c2        ─── dev       ─── Deployment/podinfo updated   2022-05-31 17:06:14 Ready: 1/1  Up-to-date: 1  Available: 1  Age: 13m
local     ─── dev       ─── Deployment/podinfo updated   2022-05-31 17:06:14 Ready: 1/1  Up-to-date: 1  Available: 1  Age: 13m
```

> 注意，如果没有安装 `vela-prism` 组件，非管理员用户都无法查看资源状态。

## 扩展阅读

这个文档提供了系统授权的基本操作说明，事实上 KubeVela 支持更细粒度的权限管理，并与 Kubernetes RBAC 权限一致，你可以阅读[底层实现原理](./advance)文档了解更多详情。

> 如果你是平台管理员，你可以阅读[系统集成](./integration)文档了解更多与 Kubernetes RBAC 集成的细节。
