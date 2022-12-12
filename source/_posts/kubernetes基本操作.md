---
title: kubernetes基本操作
date: 2022-12-12 09:24:51
tags:
categories: devops
---

## 集群相关



## pod相关

删除pod

每当删除namespace或pod 等一些Kubernetes资源时，有时资源状态会卡在terminating，很长时间无法删除，甚至有时增加--force flag(强制删除)之后还是无法正常删除。这时就需要edit该资源，将字段finalizers设置为null，之后Kubernetes资源就正常删除了。当删除pod时有时会卡住，pod状态变为terminating，无法删除pod

```bash
$ kubectl delete pod xxx -n xxx --force --grace-period=0
```



## 节点控制

### 删除节点

对节点执行维护操作之前（例如：内核升级，硬件维护等），您可以使用 `kubectl drain` 安全驱逐节点上面所有的 pod。安全驱逐的方式将会允许 pod 里面的容器遵循指定的 `PodDisruptionBudgets` 执行[优雅的中止](https://k8smeetup.github.io/docs/tasks/#lifecycle-hooks-and-termination-notice)。**注：** 默认情况下，`kubectl drain` 会忽略那些不能杀死的系统类型的 pod，如果您想了解更多详细的内容，请参考[kubectl drain](https://k8smeetup.github.io/docs/user-guide/kubectl/v1.9/#drain)。

```shell
#先将节点设置为维护模式(k8snode1是节点名称)
$ kubectl drain k8snode01 --delete-local-data --force --ignore-daemonsets node/k8snode01 

#删除节点
$ kubectl delete node k8snode01

#确认是否已经删除
$ kubectl get nodes
```



## 存储相关

### PV相关

#### 强制删除

集群内有一个已经不再使用的 **PV**，虽然已经删除了与其关联的 **Pod** 及 **PVC**，并对其执行了删除命令，但仍无法正常删除，一直处于 **Terminating** 状态：

```bash
$ kubectl patch pv k8s-pv-kafka02 -p '{"metadata":{"finalizers":null}}'
```

### PVC相关

#### 强制删除

```bash
$ kubectl patch pvc k8s-pvc-kafka02 -p '{"metadata":{"finalizers":null}}'
```



## 命名空间

### 删除相关

#### 强制删除

```bash
$ kubectl get ns kubesphere-system  -o json > temp.json

$ vim temp.json
"spec": {
"finalizers": [
	"kubernetes"
]
},

# 删除其他的内容如下,保存
"spec": {
},

# 然后新开一个窗口运行kubectl proxy跑一个API代理在本地的8081端口
$ kubectl proxy --port=8081

# 执行命令
$ curl -k -H "Content-Type: application/json" -X PUT --data-binary @temp.json http://127.0.0.1:8081/api/v1/namespaces/kubesphere-system/fina
```