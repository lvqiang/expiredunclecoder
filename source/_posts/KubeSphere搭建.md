[KubeSphere](https://kubesphere.io/) 是在 [Kubernetes](https://kubernetes.io/) 之上构建的面向云原生应用的**分布式操作系统**，完全开源，支持多云与多集群管理，提供全栈的 IT 自动化运维能力，简化企业的 DevOps 工作流。它的架构可以非常方便地使第三方应用与云原生生态组件进行即插即用 (plug-and-play) 的集成。

参考文档：https://kubesphere.com.cn/docs/v3.3/installing-on-kubernetes/on-prem-kubernetes/install-ks-on-linux-airgapped/

## 私有harbor

### 安装条件

官网给出了安装需要的最低硬件和软件的条件：https://goharbor.io/docs/2.0.0/install-config/installation-prereqs/。

### 安装方式

可以从 [官方发布](https://github.com/goharbor/harbor/releases)页面下载 Harbor 安装程序。下载在线安装程序或离线安装程序。

- 在线安装程序：在线安装程序从 Docker 中心下载 Harbor 镜像。因此，安装程序的尺寸非常小。
- 离线安装程序：如果要部署 Harbor 的主机没有连接到 Internet，请使用离线安装程序。离线安装程序包含预先构建的映像，因此它比在线安装程序大在线和离线安装程序的安装过程几乎相同。这里我主要使用离线安装，因为在线安装因为墙、内部网络等原因，有的同学会下载很慢，而离线安装包都包含了预先构建的镜像，所以直接现在离线安装包最好！

### 离线安装

到 [官方发布](https://github.com/goharbor/harbor/releases) 页面下载离线安装包，下载后将压缩包上传到服务器使用命令`tar -zxf harbor-offline-installer-v2.4.2.tgz`解压，解压后，打开`harbor.yml`，最主要是更改一下几点：

然后继续用命令执行安装

```bash
$ ./install.sh
```

然后根据在harbor.yml文件中配置的端口与IP地址(或域名)进行访问。

### kubesphere离线镜像

```bash
#准备一个文件夹
$ mkdir /opt/harbor/kubesphere
$ cd /opt/harbor/kubesphere

# 下载需要的镜像列表
$ curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/images-list.txt

# 查看需要的镜像，不需要的可以直接删掉
```





## NFS搭建

NFS即网络文件系统Network File System，它是一种[分布式文件系统](https://cloud.tencent.com/product/chdfs?from=10680)协议，最初是由Sun MicroSystems公司开发的类Unix操作系统之上的一款经典网络存储方案，其功能是在允许客户端主机可以像访问本地存储一样通过网络访问服务端文件。

Kubernetes的NFS存储用于将某事先存在的NFS服务器导出export的存储空间挂载到Pod中来供Pod[容器](https://cloud.tencent.com/product/tke?from=10680)使用。与emptyDir不同的是，NFS存储在Pod对象终止后仅是被卸载而非删除。另外，NFS是文件系统及共享服务，它支持同时存在多路挂载请求。定义NFS存储时，常用到以下字段。

- server：NFS服务器的IP地址或者主机名，必选字段。
- path：NFS服务器导出(共享)的文件系统路径，必选字段。
- readOnly：是否以只读挂载，默认为false。

### NFS安装

每台机器都需要安装nfs的客户端和服务器端。

```bash
$ yum -y install nfs-utils

#开机启动
$ systemctl enable nfs

#启动
$ systemctl start nfs

#验证
$ systemctl status nfs
```

### NFS配置

```bash
#创建要共享的目录
$ mkdir /opt/data/es -p

#设置文件夹权限否则可能导致PVC创建失败
$ chmod -R 777 /opt/data/es

#编辑NFS配置并加入以下内容
$ vim /etc/exports
/opt/data/es 192.168.31.0/24(rw,sync,no_root_squash,no_subtree_check)

#载入配置
$ exportfs -rv

#重启启动
$ systemctl restart nfs
```

配置具体解释：

- /opt/data/es：NFS服务要共享的目录
- 192.168.31.0/24：允许访问NFS服务器的网段，也可以写 * ，表示所有地址都可以访问NFS服务
- rw：访问到此目录的服务器都具备读写权限
- sync：数据同步写入内存和硬盘
- no_root_squash：所有用户对根目录具备完全管理访问权限，如果安装es会要求此配置
- no_subtree_check：不检查父目录的权限

### NFS验证

```bash
#查看NFS配置是否生效
$ cat /var/lib/nfs/etab
/opt/data/es    192.168.31.0/24(rw,sync,wdelay,hide,nocrossmnt,secure,root_squash,no_all_squash,no_subtree_check,secure_locks,acl,no_pnfs,anonuid=65534,anongid=65534,sec=sys,rw,secure,root_squash,no_all_squash)

#通过showmount命令查看NFS共享情况
$ showmount -e 192.168.31.241
Export list for 192.168.31.241:
opt/data/es 192.168.31.0/24
```

### kubernetes配置默认存储

```bash
# 创建一个新的namespace
$ kubectl create ns kubesphere-system
```



```bash
#创建以下yaml文件
$ vim default-sc.yaml

#并且执行
$ kubectl apply -f default-sc.yaml

#验证
$ kubectl get sc
------
NAME                            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-default-storage (default)   nfs/kubesphere   Retain          Immediate           true                   3m16s
```

```yaml
## 创建了一个存储类
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-default-storage
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/support-snapshot: "false"
provisioner: nfs/kubesphere
parameters:
  archiveOnDelete: "true"  ## 删除pv的时候，pv的内容是否要备份
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kubesphere-system
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: 172.23.131.24:8222/kubesphere/nfs-subdir-external-provisioner:v4.0.2
          # resources:
          #    limits:
          #      cpu: 10m
          #    requests:
          #      cpu: 10m
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: nfs/kubesphere
            - name: NFS_SERVER
              value: 172.23.131.24 ## 指定自己nfs服务器地址
            - name: NFS_PATH  
              value: /opt/data/kubesphere/system  ## nfs服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.23.131.24
            path: /opt/data/kubesphere/system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kubesphere-system
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: kubesphere-system
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kubesphere-system
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  # replace with namespace where provisioner is deployed
  namespace: kubesphere-system
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: kubesphere-system
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
```



## 部署

本文使用kubernetes上离线搭建kubusphere，下载安装yaml文件

```bash
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/cluster-configuration.yaml
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/kubesphere-installer.yaml
```

### 配置

```bash
#修改cluster-configuration.yaml文件
$ vim cluster-configuration.yaml
-----------------
spec:
  persistence:
    storageClass: ""
  authentication:
    jwtSecret: ""
  local_registry: 172.23.131.24:8222 # Add this line manually; make sure you use your own registry address.
  
#修改kubesphere-installer.yaml文件
$ sed -i "s#^\s*image: kubesphere.*/ks-installer:.*#        image: 172.23.131.24:8222/kubesphere/ks-installer:v3.0.0#" kubesphere-installer.yaml
```

### 安装

```bash
$ kubectl apply -f kubesphere-installer.yaml
$ kubectl apply -f cluster-configuration.yaml
```

### 验证

```bash
$ kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f

```



## 卸载

文档：https://kubesphere.io/zh/docs/v3.3/installing-on-kubernetes/uninstall-kubesphere-from-k8s/

```bash
#!/usr/bin/env bash

function delete_sure(){
  cat << eof
$(echo -e "\033[1;36mNote:\033[0m")
Delete the KubeSphere cluster, including the module kubesphere-system kubesphere-devops-system kubesphere-monitoring-system kubesphere-logging-system openpitrix-system.
eof

read -p "Please reconfirm that you want to delete the KubeSphere cluster.  (yes/no) " ans
while [[ "x"$ans != "xyes" && "x"$ans != "xno" ]]; do
    read -p "Please reconfirm that you want to delete the KubeSphere cluster.  (yes/no) " ans
done

if [[ "x"$ans == "xno" ]]; then
    exit
fi
}


delete_sure

# delete ks-install
kubectl delete deploy ks-installer -n kubesphere-system 2>/dev/null

# delete helm
for namespaces in kubesphere-system kubesphere-devops-system kubesphere-monitoring-system kubesphere-logging-system openpitrix-system kubesphere-monitoring-federated
do
  helm list -n $namespaces | grep -v NAME | awk '{print $1}' | sort -u | xargs -r -L1 helm uninstall -n $namespaces 2>/dev/null
done

# delete kubefed
kubectl get cc -n kubesphere-system ks-installer -o jsonpath="{.status.multicluster}" | grep enable
if [[ $? -eq 0 ]]; then
  helm uninstall -n kube-federation-system kubefed 2>/dev/null
  #kubectl delete ns kube-federation-system 2>/dev/null
fi


helm uninstall -n kube-system snapshot-controller 2>/dev/null

# delete kubesphere deployment
kubectl delete deployment -n kubesphere-system `kubectl get deployment -n kubesphere-system -o jsonpath="{.items[*].metadata.name}"` 2>/dev/null

# delete monitor statefulset
kubectl delete prometheus -n kubesphere-monitoring-system k8s 2>/dev/null
kubectl delete statefulset -n kubesphere-monitoring-system `kubectl get statefulset -n kubesphere-monitoring-system -o jsonpath="{.items[*].metadata.name}"` 2>/dev/null
# delete grafana
kubectl delete deployment -n kubesphere-monitoring-system grafana 2>/dev/null
kubectl --no-headers=true get pvc -n kubesphere-monitoring-system -o custom-columns=:metadata.namespace,:metadata.name | grep -E kubesphere-monitoring-system | xargs -n2 kubectl delete pvc -n 2>/dev/null

# delete pvc
pvcs="kubesphere-system|openpitrix-system|kubesphere-devops-system|kubesphere-logging-system"
kubectl --no-headers=true get pvc --all-namespaces -o custom-columns=:metadata.namespace,:metadata.name | grep -E $pvcs | xargs -n2 kubectl delete pvc -n 2>/dev/null


# delete rolebindings
delete_role_bindings() {
  for rolebinding in `kubectl -n $1 get rolebindings -l iam.kubesphere.io/user-ref -o jsonpath="{.items[*].metadata.name}"`
  do
    kubectl -n $1 delete rolebinding $rolebinding 2>/dev/null
  done
}

# delete roles
delete_roles() {
  kubectl -n $1 delete role admin 2>/dev/null
  kubectl -n $1 delete role operator 2>/dev/null
  kubectl -n $1 delete role viewer 2>/dev/null
  for role in `kubectl -n $1 get roles -l iam.kubesphere.io/role-template -o jsonpath="{.items[*].metadata.name}"`
  do
    kubectl -n $1 delete role $role 2>/dev/null
  done
}

# remove useless labels and finalizers
for ns in `kubectl get ns -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl label ns $ns kubesphere.io/workspace-
  kubectl label ns $ns kubesphere.io/namespace-
  kubectl patch ns $ns -p '{"metadata":{"finalizers":null,"ownerReferences":null}}'
  delete_role_bindings $ns
  delete_roles $ns
done

# delete clusters
for cluster in `kubectl get clusters -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch cluster $cluster -p '{"metadata":{"finalizers":null}}' --type=merge
done
kubectl delete clusters --all 2>/dev/null

# delete workspaces
for ws in `kubectl get workspaces -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch workspace $ws -p '{"metadata":{"finalizers":null}}' --type=merge
done
kubectl delete workspaces --all 2>/dev/null

# delete devopsprojects
for devopsproject in `kubectl get devopsprojects -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch devopsprojects $devopsproject -p '{"metadata":{"finalizers":null}}' --type=merge
done

for pip in `kubectl get pipeline -A -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch pipeline $pip -n `kubectl get pipeline -A | grep $pip | awk '{print $1}'` -p '{"metadata":{"finalizers":null}}' --type=merge
done

for s2ibinaries in `kubectl get s2ibinaries -A -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch s2ibinaries $s2ibinaries -n `kubectl get s2ibinaries -A | grep $s2ibinaries | awk '{print $1}'` -p '{"metadata":{"finalizers":null}}' --type=merge
done

for s2ibuilders in `kubectl get s2ibuilders -A -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch s2ibuilders $s2ibuilders -n `kubectl get s2ibuilders -A | grep $s2ibuilders | awk '{print $1}'` -p '{"metadata":{"finalizers":null}}' --type=merge
done

for s2ibuildertemplates in `kubectl get s2ibuildertemplates -A -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch s2ibuildertemplates $s2ibuildertemplates -n `kubectl get s2ibuildertemplates -A | grep $s2ibuildertemplates | awk '{print $1}'` -p '{"metadata":{"finalizers":null}}' --type=merge
done

for s2iruns in `kubectl get s2iruns -A -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch s2iruns $s2iruns -n `kubectl get s2iruns -A | grep $s2iruns | awk '{print $1}'` -p '{"metadata":{"finalizers":null}}' --type=merge
done

kubectl delete devopsprojects --all 2>/dev/null


# delete validatingwebhookconfigurations
for webhook in ks-events-admission-validate users.iam.kubesphere.io network.kubesphere.io validating-webhook-configuration
do
  kubectl delete validatingwebhookconfigurations.admissionregistration.k8s.io $webhook 2>/dev/null
done

# delete mutatingwebhookconfigurations
for webhook in ks-events-admission-mutate logsidecar-injector-admission-mutate mutating-webhook-configuration
do
  kubectl delete mutatingwebhookconfigurations.admissionregistration.k8s.io $webhook 2>/dev/null
done

# delete users
for user in `kubectl get users -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch user $user -p '{"metadata":{"finalizers":null}}' --type=merge
done
kubectl delete users --all 2>/dev/null


# delete helm resources
for resource_type in `echo helmcategories helmapplications helmapplicationversions helmrepos helmreleases`; do
  for resource_name in `kubectl get ${resource_type}.application.kubesphere.io -o jsonpath="{.items[*].metadata.name}"`; do
    kubectl patch ${resource_type}.application.kubesphere.io ${resource_name} -p '{"metadata":{"finalizers":null}}' --type=merge
  done
  kubectl delete ${resource_type}.application.kubesphere.io --all 2>/dev/null
done

# delete workspacetemplates
for workspacetemplate in `kubectl get workspacetemplates.tenant.kubesphere.io -o jsonpath="{.items[*].metadata.name}"`
do
  kubectl patch workspacetemplates.tenant.kubesphere.io $workspacetemplate -p '{"metadata":{"finalizers":null}}' --type=merge
done
kubectl delete workspacetemplates.tenant.kubesphere.io --all 2>/dev/null

# delete federatednamespaces in namespace kubesphere-monitoring-federated
for resource in $(kubectl get federatednamespaces.types.kubefed.io -n kubesphere-monitoring-federated -oname); do
  kubectl patch "${resource}" -p '{"metadata":{"finalizers":null}}' --type=merge -n kubesphere-monitoring-federated
done

# delete crds
for crd in `kubectl get crds -o jsonpath="{.items[*].metadata.name}"`
do
  if [[ $crd == *kubesphere.io ]]; then kubectl delete crd $crd 2>/dev/null; fi
done

# delete relevance ns
for ns in kubesphere-alerting-system kubesphere-controls-system kubesphere-devops-system kubesphere-logging-system kubesphere-monitoring-system kubesphere-monitoring-federated openpitrix-system kubesphere-system
do
  kubectl delete ns $ns 2>/dev/null
done
```

