---
title: 生产级kubesphere部署
date: 2022-12-07 12:03:10
tags: kubesphere
categories:
- devops

---

[KubeSphere](https://kubesphere.io/) 是在 [Kubernetes](https://kubernetes.io/) 之上构建的面向云原生应用的**分布式操作系统**，完全开源，支持多云与多集群管理，提供全栈的 IT 自动化运维能力，简化企业的 DevOps 工作流。它的架构可以非常方便地使第三方应用与云原生生态组件进行即插即用 (plug-and-play) 的集成。

参考文档：https://kubesphere.com.cn/docs/v3.3/installing-on-kubernetes/on-prem-kubernetes/install-ks-on-linux-airgapped/

提示：本文是在原有kubernetes基础上搭建kubesphere，难度较大，我尝试装了好多次，虽然以前有成功的经验，但是这次还是失败了好多次，总结经验首先要有k8s的基础，在安装部署过程中会有多个pod失败，需要多次尝试修改才可以；第二要在一个完成干净的机器上安装好k8s；第三一定要保证机器的容量。如果还是一直失败，推荐先用其kk工具搭建一个完整的kubesphere，等待熟悉各个组件之后，再次尝试在原有k8s基础上安装。

## 环境准备

环境准备文档：https://kubesphere.io/zh/docs/v3.3/installing-on-kubernetes/introduction/prerequisites/

### 集群

本次部署在原有kubernetes集群上安装，首先要准备一个高可用集群。

集群版本如下：

| 软件       | 版本号                        | 说明     |
| :--------- | :---------------------------- | :------- |
| 内核       | 3.10.0-514.el7.x86_64         |          |
| centos     | CentOS Linux release 7.9.2009 |          |
| docker-ce  | 20.10.19                      | 社区版本 |
| kubernetes | 1.23.9                        | 稳定版本 |
| calico     | 2.23                          |          |

集群节点如下：

| 节点    | 配置   | ip      | 标签                              |
| ------- | ------ | ------- | --------------------------------- |
| master1 | 4c8g   | 116.81  |                                   |
| master2 | 4c8g   | 116.82  |                                   |
| master3 | 4c8g   | 116.83  | node-role.kubernetes.io/worker ci |
| work1   | 16c32g | 116.93  |                                   |
| work2   | 8c16g  | 116.121 |                                   |
| work3   | 8c16g  | 116.72  |                                   |

### 存储

kubesphere在存储中建议生产环境使用glusterfs和ceph，因此本次存储选用了glusterfs来进行。默认已经安装好了glusterfs和heketi。

安装文档：https://kubesphere.io/zh/docs/v3.3/reference/storage-system-installation/glusterfs-server/

创建glusterfs-sc.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: heketi-secret
  namespace: kube-system
type: kubernetes.io/glusterfs
data:
  key: "MTIzNDU2"    #请替换为您自己的密钥。Base64 编码。
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.beta.kubernetes.io/is-default-class: "true"
    storageclass.kubesphere.io/supported-access-modes: '["ReadWriteOnce","ReadOnlyMany","ReadWriteMany"]'
  name: glusterfs-sc-default
parameters:
  clusterid: "21240a91145aee4d801661689383dcd1"    #请替换为您自己的 GlusterFS 集群 ID。
  gidMax: "50000"
  gidMin: "40000"
  restauthenabled: "true"
  resturl: "http://192.168.0.2:8080"    #Gluster REST 服务/Heketi 服务 URL 可按需供应 gluster 存储卷。请替换为您自己的 URL。
  restuser: admin
  secretName: heketi-secret
  secretNamespace: kube-system
  volumetype: "replicate:3"    #请替换为您自己的存储卷类型。
provisioner: kubernetes.io/glusterfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
```

```bash
# 创建默认存储
$ kubectl apply -f glusterfs-sc.yaml

#验证
$ kubectl get sc
------------------------------------------
NAME                             PROVISIONER               RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
glusterfs-sc-default (default)   kubernetes.io/glusterfs   Delete          Immediate           true                   6s
```

### 私有harbor

#### 安装条件

官网给出了安装需要的最低硬件和软件的条件：https://goharbor.io/docs/2.0.0/install-config/installation-prereqs/。

#### 安装方式

可以从 [官方发布](https://github.com/goharbor/harbor/releases)页面下载 Harbor 安装程序。下载在线安装程序或离线安装程序。

- 在线安装程序：在线安装程序从 Docker 中心下载 Harbor 镜像。因此，安装程序的尺寸非常小。
- 离线安装程序：如果要部署 Harbor 的主机没有连接到 Internet，请使用离线安装程序。离线安装程序包含预先构建的映像，因此它比在线安装程序大在线和离线安装程序的安装过程几乎相同。这里我主要使用离线安装，因为在线安装因为墙、内部网络等原因，有的同学会下载很慢，而离线安装包都包含了预先构建的镜像，所以直接现在离线安装包最好！

#### 离线安装

到 [官方发布](https://github.com/goharbor/harbor/releases) 页面下载离线安装包，下载后将压缩包上传到服务器使用命令`tar -zxf harbor-offline-installer-v2.4.2.tgz`解压，解压后，打开`harbor.yml`，最主要是更改一下几点：

然后继续用命令执行安装

```bash
$ ./install.sh
```

然后根据在harbor.yml文件中配置的端口与IP地址(或域名)进行访问。

#### kubesphere离线镜像

```bash
#准备一个文件夹
$ mkdir /opt/harbor/kubesphere
$ cd /opt/harbor/kubesphere

# 下载需要的镜像列表
$ curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/images-list.txt

# 查看需要的镜像，不需要的可以直接删掉
```

## 部署

本文使用kubernetes上离线搭建kubusphere，下载安装yaml文件

```bash
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/cluster-configuration.yaml
curl -L -O https://github.com/kubesphere/ks-installer/releases/download/v3.3.1/kubesphere-installer.yaml
```

```bash
---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.3.1
spec:
  persistence:
    storageClass: ""        
  authentication:
    # adminPassword: "" 
    jwtSecret: ""          
  local_registry: "172.23.131.24:8222"        # Add your private registry address if it is needed.
  # dev_tag: ""             
  etcd:
    monitoring: false       
    endpointIps: localhost  
    port: 2379             
    tlsEnable: true
  common:
    core:
      console:
        enableMultiLogin: true  
        port: 30880
        type: NodePort
    # apiserver:         
    #  resources: {}
    # controllerManager:
    #  resources: {}
    redis:
      enabled: true
      enableHA: false
      volumeSize: 2Gi # Redis PVC size.
    openldap:
      enabled: false
      volumeSize: 2Gi   # openldap PVC size.
    minio:
      volumeSize: 20Gi # Minio PVC size.
    monitoring:
      # type: external   # Whether to specify the external prometheus stack, and need to modify the endpoint at the next line.
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090 # Prometheus endpoint to get metrics data.
      GPUMonitoring:    
        enabled: false
    gpu:            
      kinds:
      - resourceName: "nvidia.com/gpu"
        resourceType: "GPU"
        default: true
    es:   # Storage backend for logging, events and auditing.
      # master:
      #   volumeSize: 4Gi  # The volume size of Elasticsearch master nodes.
      #   replicas: 1      # The total number of master nodes. Even numbers are not allowed.
      #   resources: {}
      # data:
      #   volumeSize: 20Gi  # The volume size of Elasticsearch data nodes.
      #   replicas: 1       # The total number of data nodes.
      #   resources: {}
      logMaxAge: 7             # Log retention time in built-in Elasticsearch. It is 7 days by default.
      elkPrefix: logstash      # The string making up index names. The index name will be formatted as ks-<elk_prefix>-log.
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchHost: ""
      externalElasticsearchPort: ""
  alerting:               
    enabled: false      
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:                
    enabled: false         # Enable or disable the KubeSphere Auditing Log System.
    # operator:
    #   resources: {}
    # webhook:
    #   resources: {}
  devops:   
    enabled: false             # Enable or disable the KubeSphere DevOps System.
    # resources: {}
    jenkinsMemoryLim: 8Gi      # Jenkins memory limit.
    jenkinsMemoryReq: 4Gi   # Jenkins memory request.
    jenkinsVolumeSize: 8Gi     # Jenkins volume size.
  events:                 
    enabled: false       
    # operator:
    #   resources: {}
    # exporter:
    #   resources: {}
    # ruler:
    #   enabled: true
    #   replicas: 2
    #   resources: {}
  logging:               
    enabled: false         # Enable or disable the KubeSphere Logging System.
    logsidecar:
      enabled: true
      replicas: 2
      # resources: {}
  metrics_server:                   
    enabled: true             
  monitoring:
    storageClass: "" 
    node_exporter:
      port: 9100
      # resources: {}
    # kube_rbac_proxy:
    #   resources: {}
    # kube_state_metrics:
    #   resources: {}
    # prometheus:
    #   replicas: 1  # Prometheus replicas are responsible for monitoring different segments of data source and providing high availability.
    #   volumeSize: 20Gi  # Prometheus PVC size.
    #   resources: {}
    #   operator:
    #     resources: {}
    # alertmanager:
    #   replicas: 1          # AlertManager Replicas.
    #   resources: {}
    # notification_manager:
    #   resources: {}
    #   operator:
    #     resources: {}
    #   proxy:
    #     resources: {}
    gpu:                           # GPU monitoring-related plug-in installation.
      nvidia_dcgm_exporter:        # Ensure that gpu resources on your hosts can be used normally, otherwise this plug-in will not work properly.
        enabled: false             # Check whether the labels on the GPU hosts contain "nvidia.com/gpu.present=true" to ensure that the DCGM pod is scheduled to these nodes.
        # resources: {}
  multicluster:
    clusterRole: none  # host | member | none  # You can install a solo cluster, or specify it as the Host or Member Cluster.
  network:
    networkpolicy: # Network policies allow network isolation within the same cluster, which means firewalls can be set up between certain instances (Pods).
      # Make sure that the CNI network plugin used by the cluster supports NetworkPolicy. There are a number of CNI network plugins that support NetworkPolicy, including Calico, Cilium, Kube-router, Romana and Weave Net.
      enabled: false # Enable or disable network policies.
    ippool: # Use Pod IP Pools to manage the Pod network address space. Pods to be created can be assigned IP addresses from a Pod IP Pool.
      type: none # Specify "calico" for this field if Calico is used as your CNI plugin. "none" means that Pod IP Pools are disabled.
    topology: # Use Service Topology to view Service-to-Service communication based on Weave Scope.
      type: none # Specify "weave-scope" for this field to enable Service Topology. "none" means that Service Topology is disabled.
  openpitrix: # An App Store that is accessible to all platform tenants. You can use it to manage apps across their entire lifecycle.
    store:
      enabled: false # Enable or disable the KubeSphere App Store.
  servicemesh:         # (0.3 Core, 300 MiB) Provide fine-grained traffic management, observability and tracing, and visualized traffic topology.
    enabled: false     # Base component (pilot). Enable or disable KubeSphere Service Mesh (Istio-based).
    istio:  # Customizing the istio installation configuration, refer to https://istio.io/latest/docs/setup/additional-setup/customize-installation/
      components:
        ingressGateways:
        - name: istio-ingressgateway
          enabled: false
        cni:
          enabled: false
  edgeruntime:          # Add edge nodes to your cluster and deploy workloads on edge nodes.
    enabled: false
    kubeedge:        # kubeedge configurations
      enabled: false
      cloudCore:
        cloudHub:
          advertiseAddress: # At least a public IP address or an IP address which can be accessed by edge nodes must be provided.
            - ""            # Note that once KubeEdge is enabled, CloudCore will malfunction if the address is not provided.
        service:
          cloudhubNodePort: "30000"
          cloudhubQuicNodePort: "30001"
          cloudhubHttpsNodePort: "30002"
          cloudstreamNodePort: "30003"
          tunnelNodePort: "30004"
        # resources: {}
        # hostNetWork: false
      iptables-manager:
        enabled: true 
        mode: "external"
        # resources: {}
      # edgeService:
      #   resources: {}
  gatekeeper:        # Provide admission policy and rule management, A validating (mutating TBA) webhook that enforces CRD-based policies executed by Open Policy Agent.
    enabled: false   # Enable or disable Gatekeeper.
    # controller_manager:
    #   resources: {}
    # audit:
    #   resources: {}
  terminal:
    # image: 'alpine:3.15' # There must be an nsenter program in the image
    timeout: 600         # Container timeout, if set to 0, no timeout will be used. The unit is seconds
  telemetry_enabled: false # 请手动添加此行以禁用 Telemetry。
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

# 或者查看pod状态
$ kubectl get po -n kubesphere-system 
```

## 界面

日志显示的地址和用户名密码登录成功后，显示集群界面。

<img src="http://cdn.expiredunclecoder.tech/image-20221227131625658.png" alt="image-20221227131625658" style="zoom:67%;" />

##  存储

kubesphere在部署时会使用默认存储glusterfs来动态创建pv和pvc，通过页面我们可以知道相应的volume，这样在后期维护时我们可以找到存储数据。

<img src="http://cdn.expiredunclecoder.tech/image-20221227132709347.png" alt="image-20221227132709347" style="zoom:67%;" />

其中我们看到一共创建了两类，一类是redis的数据，一类是监控promethues数据。

我们知道redis的volume对应的是vol_bd609e3e550dbaae7908f1c5bd8afb08，进入到glusterfs服务器中

```bash
$ gluster volume info vol_bd609e3e550dbaae7908f1c5bd8afb08
--------------------------------
Volume Name: vol_bd609e3e550dbaae7908f1c5bd8afb08
Type: Replicate
Volume ID: 604ba5d4-e914-424f-a6d4-8826ee1804ca
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: xxx.xxx.xxx.xxx:/var/lib/heketi/mounts/vg_581e81855db6227ce13fd331a8264fea/brick_c7892bfb3b12aa300b99004eea3a68f3/brick
Brick2: xxx.xxx.xxx.xxx:/var/lib/heketi/mounts/vg_5213f198556ac476980d9696a65cdcbb/brick_dee86e8d77f7996e69e03cda2c6e9450/brick
Brick3: xxx.xxx.xxx.xxx:/var/lib/heketi/mounts/vg_e655ffb8f7e21679e4453413567324bb/brick_885d215d30d2b5ab3f45fc56ff9f03a1/brick
Options Reconfigured:
user.heketi.id: bd609e3e550dbaae7908f1c5bd8afb08
cluster.granular-entry-heal: on
storage.fips-mode-rchecksum: on
transport.address-family: inet
nfs.disable: on
performance.client-io-threads: off
```

## 组件

KubeSphere 解耦了一些核心功能组件。这些组件设计成了可插拔式，前面我们按照最小化来安装部署，下面尝试使用一些组件来完善功能。

文档：https://kubesphere.io/zh/docs/v3.3/pluggable-components/overview/

### 监控

kubesphere默认是使用了prometheus-operator，具体使用文档可以参考https://www.prometheus.wang/operator/what-is-prometheus-operator.html。其中kubesphere已经帮我们来完成了集群中容器监控和展示。如果自定义我们其他的监控，可以通过创建servicemonitor来完成。

具体kubesphere自定义监控文档可以参考：https://kubesphere.io/zh/docs/v3.3/project-user-guide/custom-application-monitoring/introduction/

<img src="http://cdn.expiredunclecoder.tech/image-20221227135048311.png" alt="image-20221227135048311" style="zoom: 50%;" />

#### mysql监控

使用mysql exporter创建servicemonitor来获取mysql的监控数据给到prometheus。具体安装如下：

1. 从dockerhub中找到mysql exporter镜像。https://hub.docker.com/r/prom/mysqld-exporter。

2. 通过kubesphere来创建deployment。

   <img src="http://cdn.expiredunclecoder.tech/image-20221227141247915.png" alt="image-20221227141247915" style="zoom:50%;" />

3. 创建成功后。

   <img src="http://cdn.expiredunclecoder.tech/image-20221227142041775.png" style="zoom: 50%;" />

4. 创建service来提供给servicemonitor

   <img src="http://cdn.expiredunclecoder.tech/image-20221227142600190.png" alt="image-20221227142600190" style="zoom:67%;" />

5. 创建相应servicemonitor,prom-mysql-exporter-sm.yaml

   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor 
   metadata:
     labels:
       app: prom-mysql-exporter-sm
     name: prom-mysql-exporter-sm
     namespace: kubesphere-monitoring-system
   spec:
     selector:
       matchLabels:
         app: prom-mysql-exporter-svc   # 该serviceMonitor 通过标签选择器 来自动发现exporter 的sevice
     namespaceSelector:
       matchNames:
       - kubesphere-monitoring-system
     endpoints:
     - port: 9104    
       interval: 30s 
       honorLabels: tru
   ```

6. 在自定义监控面板中创建mysql监控面板即可。

   <img src="http://cdn.expiredunclecoder.tech/image-20221227144340576.png" alt="image-20221227144340576" style="zoom:33%;" />

#### redis监控

同上。

## 问题

如果使用了不是干净的机器安装，或多或少会遇到网络问题，所以在安装kubesphere之前一定要重置机器。

如果在安装后发现pod一直CrashLoopBackOff，一定要通过命令log或者describe是看原因，我安装下来基本上是两种

1. 资源不够
2. 存储设置不对

在解决以上问题后，首先要delete掉问题pod，如果还是有问题，建议重启installer。

```bash
$ kubectl rollout restart deploy -n kubesphere-system ks-installer
```

### apiservice

安装installer时发现一直crash，看日志出现以下内容：

```bash
error getting GVR for kind 'ClusterConfiguration': unable to retrieve the complete list of server APIs: projectcalico.org/v3: the server is currently unable to handle the request

$ kubectl get apiservice
------------------------------------
v1beta2.flowcontrol.apiserver.k8s.io   Local                         True                           22h
v2.autoscaling                         Local                         True                           22h
v2beta1.autoscaling                    Local                         True                           22h
v2beta2.autoscaling                    Local                         True                           22h
v3.projectcalico.org                   calico-apiserver/calico-api   False (FailedDiscoveryCheck)   22h

#可以看到 False (FailedDiscoveryCheck) 的列表，删除该apiservice即可
$ kubectl delete apiservice v3.projectcalico.org
```

### default storageClass

安装过程中，如果出现以下内容，表示k8s里没有默认的动态存储，创建即可。

```bash
fatal: [localhost]: FAILED! => {
    "assertion": "\"(default)\" in default_storage_class_check.stdout",
    "changed": false,
    "evaluated_to": false,
    "msg": "Default StorageClass was not found !"
}
```

###  SelfLink

如果出现默认绑定pvc一直处于Pending，需要查看nfs pod的日志如果出现selfLink was empty, can't make reference的内容，可知因为selfLink在1.16版本以后已经弃用，在1.20版本停用。

```bash
#修改/etc/kubernetes/manifests/kube-apiserver.yaml文件
$ vim /etc/kubernetes/manifests/kube-apiserver.yaml
---------------------------------------
 - --feature-gates=RemoveSelfLink=false
 
 # 添加后需要删除apiserver的所有pod进行重启
```

## 卸载

文档：https://kubesphere.io/zh/docs/v3.3/installing-on-kubernetes/uninstall-kubesphere-from-k8s/

