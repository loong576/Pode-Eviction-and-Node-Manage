**环境说明：**

| 主机名 |  操作系统版本   |      ip      | docker version | kubelet version | 配置 |    备注    |
| :----: | :-------------: | :----------: | :------------: | :-------------: | :--: | :--------: |
| master | Centos 7.6.1810 | 172.27.9.131 | Docker 18.09.6 |     V1.14.2     | 2C2G | master主机 |
| node01 | Centos 7.6.1810 | 172.27.9.135 | Docker 18.09.6 |     V1.14.2     | 2C2G |  node节点  |
| node02 | Centos 7.6.1810 | 172.27.9.136 | Docker 18.09.6 |     V1.14.2     | 2C2G |  node节点  |


&nbsp;

**k8s集群部署详见：**[Centos7.6部署k8s(v1.14.2)集群 ](https://blog.51cto.com/3241766/2405624)

**k8s学习资料详见：**[基本概念、kubectl命令和资料分享 ](https://blog.51cto.com/3241766/2413457)

**emptyDir详见：**[存储卷和数据持久化(Volumes and Persistent Storage) ](https://blog.51cto.com/3241766/2435182)

## **一、背景**

> 当node节点进行如打补丁、操作系统升级等操作时，需停机维护，这就涉及pod驱逐迁移，本文将详细介绍node节点维护的整个过程。

## **二、pdb简介**

> - pdb为poddisruptionbudgets缩写，意为主动驱逐保护；
> - 没有pdb。当进行节点维护时，如果某个服务的多个pod在该节点上，则节点的停机可能会造成服务中断或者服务降级。举个例子，某服务有5个pod，最低3个pod能保证服务质量，否则会造成响应慢等影响，此时该服务的4个pod在node01上，如果对node01进行停机维护，此时只有1个pod能正常对外服务，在node01的4个pod迁移过程中，就会影响该服务正常响应；
> - pdb能保证应用在节点维护时不低于一定数量的pod运行，从而保持服务质量；

## **三、准备工作**

### **1.新建pod**

```bash
[root@master ~]# more nginx-master.yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-master
spec:
  replicas: 10 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      restartPolicy: Always
      containers:
      - name: nginx
        image: nginx:latest
[root@master ~]# kubectl apply -f nginx-master.yml 
deployment.extensions/nginx-master created
[root@master ~]# kubectl get po -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nginx-master-9d4cf4f77-47vfj   1/1     Running   0          28s   10.244.0.129   master   <none>           <none>
nginx-master-9d4cf4f77-69jn6   1/1     Running   0          28s   10.244.2.206   node02   <none>           <none>
nginx-master-9d4cf4f77-6drhg   1/1     Running   0          28s   10.244.1.218   node01   <none>           <none>
nginx-master-9d4cf4f77-b7zfd   1/1     Running   0          28s   10.244.1.219   node01   <none>           <none>
nginx-master-9d4cf4f77-fxsjd   1/1     Running   0          28s   10.244.2.204   node02   <none>           <none>
nginx-master-9d4cf4f77-ktnvk   1/1     Running   0          28s   10.244.0.128   master   <none>           <none>
nginx-master-9d4cf4f77-mzrx7   1/1     Running   0          28s   10.244.1.217   node01   <none>           <none>
nginx-master-9d4cf4f77-pcznk   1/1     Running   0          28s   10.244.2.203   node02   <none>           <none>
nginx-master-9d4cf4f77-px98b   1/1     Running   0          28s   10.244.2.205   node02   <none>           <none>
nginx-master-9d4cf4f77-wtcwt   1/1     Running   0          28s   10.244.1.220   node01   <none>           <none>
```

新建pod，镜像为最新版的nginx，deployment为nginx-master，数量为10。可以看到10个pod分布在node01、node02和master 3台不同主机上。



### **2.新建pdb**	

```bash
[root@master ~]# more pdb-nginx.yaml 
apiVersion: policy/v1beta1
kind: PodDisruptionBudget
metadata:
  name: pdb-nginx
spec:
  minAvailable: 9
  selector:
    matchLabels:
      app: nginx
[root@master ~]# kubectl apply -f pdb-nginx.yaml 
poddisruptionbudget.policy/pdb-nginx created
[root@master ~]# kubectl get pdb
NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
pdb-nginx   9               N/A               1                     8s
```

新建pdb pdb-nginx，Label Selector和deployment一样都为app: nginx，minAvailable: 9意为存活的nginx pod至少为9个。

## **四、节点维护**

本文以节点node02维护为例介绍。

### **1.设置节点不可调度**

```bash
[root@master ~]# kubectl cordon node02
node/node02 cordoned
[root@master ~]# kubectl get node
NAME     STATUS                     ROLES    AGE    VERSION
master   Ready                      master   184d   v1.14.2
node01   Ready                      <none>   183d   v1.14.2
node02   Ready,SchedulingDisabled   <none>   182d   v1.14.2
[root@master ~]# kubectl get po -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nginx-master-9d4cf4f77-47vfj   1/1     Running   0          30m   10.244.0.129   master   <none>           <none>
nginx-master-9d4cf4f77-69jn6   1/1     Running   0          30m   10.244.2.206   node02   <none>           <none>
nginx-master-9d4cf4f77-6drhg   1/1     Running   0          30m   10.244.1.218   node01   <none>           <none>
nginx-master-9d4cf4f77-b7zfd   1/1     Running   0          30m   10.244.1.219   node01   <none>           <none>
nginx-master-9d4cf4f77-fxsjd   1/1     Running   0          30m   10.244.2.204   node02   <none>           <none>
nginx-master-9d4cf4f77-ktnvk   1/1     Running   0          30m   10.244.0.128   master   <none>           <none>
nginx-master-9d4cf4f77-mzrx7   1/1     Running   0          30m   10.244.1.217   node01   <none>           <none>
nginx-master-9d4cf4f77-pcznk   1/1     Running   0          30m   10.244.2.203   node02   <none>           <none>
nginx-master-9d4cf4f77-px98b   1/1     Running   0          30m   10.244.2.205   node02   <none>           <none>
nginx-master-9d4cf4f77-wtcwt   1/1     Running   0          30m   10.244.1.220   node01   <none>           <none>
```

设置node02不可调度，查看各节点状态，发现node02为SchedulingDisabled，此时master不会将新的pod调度到该节点上，但是node02上pod还是正常运行。

### **2.驱逐节点上的pod**

```bash
[root@master ~]# kubectl drain node02 --delete-local-data --ignore-daemonsets --force 
node/node02 already cordoned
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/c7d5n528mr.png)

**参数说明：**

> - **--delete-local-data**  即使pod使用了emptyDir也删除
> - **--ignore-daemonsets**  忽略deamonset控制器的pod，如果不忽略，deamonset控制器控制的pod被删除后可能马上又在此节点上启动起来,会成为死循环；
> - -**-force**  不加force参数只会删除该NODE上由ReplicationController, ReplicaSet, DaemonSet,StatefulSet or Job创建的Pod，加了后还会删除'裸奔的pod'(没有绑定到任何replication controller)

可以看到同一时刻只有一个pod进行迁移，对外提供服务的pod始终有9个。

![图片.png](https://ask.qcloudimg.com/draft/6211241/25lu688tni.png)

迁移pod nginx-master-9d4cf4f77-pcznk到node01

![图片.png](https://ask.qcloudimg.com/draft/6211241/zg5gus56lp.png)

迁移pod nginx-master-9d4cf4f77-px98b到master，此时前一个pod nginx-master-9d4cf4f77-pcznk已经迁移完成。

![图片.png](https://ask.qcloudimg.com/draft/6211241/uluw4lrdga.png)

迁移pod nginx-master-9d4cf4f77-69jn6到master

![图片.png](https://ask.qcloudimg.com/draft/6211241/w0anikjs0g.png)

迁移pod nginx-master-9d4cf4f77-fxsjd到master



这个也再次验证了同一时刻只有一个pod迁移，nginx服务始终有9个pod对外提供服务。

### **3.维护结束**

```bash
[root@master ~]# kubectl uncordon node02
node/node02 uncordoned
[root@master ~]# kubectl get nodes      
NAME     STATUS   ROLES    AGE    VERSION
master   Ready    master   184d   v1.14.2
node01   Ready    <none>   183d   v1.14.2
node02   Ready    <none>   183d   v1.14.2
```

维护结束，重新将node02节点置为可调度状态。

## **五、pod回迁**

pod回迁貌似还没什么好的办法，这里采用delete然后重建的方式回迁。

```bash
[root@master ~]# kubectl get po -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nginx-master-9d4cf4f77-2vnvk   1/1     Running   0          33m   10.244.1.222   node01   <none>           <none>
nginx-master-9d4cf4f77-47vfj   1/1     Running   0          73m   10.244.0.129   master   <none>           <none>
nginx-master-9d4cf4f77-6drhg   1/1     Running   0          73m   10.244.1.218   node01   <none>           <none>
nginx-master-9d4cf4f77-7n7pt   1/1     Running   0          32m   10.244.0.131   master   <none>           <none>
nginx-master-9d4cf4f77-b7zfd   1/1     Running   0          73m   10.244.1.219   node01   <none>           <none>
nginx-master-9d4cf4f77-ktnvk   1/1     Running   0          73m   10.244.0.128   master   <none>           <none>
nginx-master-9d4cf4f77-mzrx7   1/1     Running   0          73m   10.244.1.217   node01   <none>           <none>
nginx-master-9d4cf4f77-pdkst   1/1     Running   0          32m   10.244.0.130   master   <none>           <none>
nginx-master-9d4cf4f77-pskmp   1/1     Running   0          32m   10.244.0.132   master   <none>           <none>
nginx-master-9d4cf4f77-wtcwt   1/1     Running   0          73m   10.244.1.220   node01   <none>           <none>
[root@master ~]# kubectl delete po nginx-master-9d4cf4f77-47vfj
pod "nginx-master-9d4cf4f77-47vfj" deleted
[root@master ~]# kubectl delete po nginx-master-9d4cf4f77-2vnvk
pod "nginx-master-9d4cf4f77-2vnvk" deleted
[root@master ~]# kubectl get po -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP             NODE     NOMINATED NODE   READINESS GATES
nginx-master-9d4cf4f77-6drhg   1/1     Running   0          76m   10.244.1.218   node01   <none>           <none>
nginx-master-9d4cf4f77-7n7pt   1/1     Running   0          35m   10.244.0.131   master   <none>           <none>
nginx-master-9d4cf4f77-b7zfd   1/1     Running   0          76m   10.244.1.219   node01   <none>           <none>
nginx-master-9d4cf4f77-f92hp   1/1     Running   0          44s   10.244.2.207   node02   <none>           <none>
nginx-master-9d4cf4f77-ktnvk   1/1     Running   0          76m   10.244.0.128   master   <none>           <none>
nginx-master-9d4cf4f77-mzrx7   1/1     Running   0          76m   10.244.1.217   node01   <none>           <none>
nginx-master-9d4cf4f77-pdkst   1/1     Running   0          35m   10.244.0.130   master   <none>           <none>
nginx-master-9d4cf4f77-pskmp   1/1     Running   0          35m   10.244.0.132   master   <none>           <none>
nginx-master-9d4cf4f77-tdghn   1/1     Running   0          15s   10.244.2.208   node02   <none>           <none>
nginx-master-9d4cf4f77-wtcwt   1/1     Running   0          76m   10.244.1.220   node01   <none>           <none>
```

在业务低峰delete pod nginx-master-9d4cf4f77-47vfj和nginx-master-9d4cf4f77-2vnvk，由于node02上的pod之前都被驱逐，此时资源使用率最低，所以pod重建时会调度值该节点，完成pod回迁。



## **六、节点删除**

### **1.删除节点**

实际运维过程中可能会删除某个node节点，本文还是以node02为例，介绍如果删除节点。

```bash
[root@master ~]# kubectl cordon node02
[root@master ~]# kubectl drain node02 --delete-local-data --ignore-daemonsets --force 
[root@master ~]# kubectl delete node node02
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/1hd4hjw9ts.png)

```bash
[root@node02 ~]# kubeadm reset
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/f0vprteysh.png)



### **2.节点重新加入**

master节点上运行

```bash
[root@master ~]# kubeadm token create --print-join-command
kubeadm join 172.27.9.131:6443 --token kpz40z.tuxb4t4m1q37vwl1     --discovery-token-ca-cert-hash sha256:5f656ae26b5e7d4641a979cbfdffeb7845cc5962bbfcd1d5435f00a25c02ea50 
```

node02重新加入集群

```bash
[root@node02 ~]# kubeadm join 172.27.9.131:6443 --token svrip0.lajrfl4jgal0ul6i     --discovery-token-ca-cert-hash sha256:5f656ae26b5e7d4641a979cbfdffeb7845cc5962bbfcd1d5435f00a25c02ea50 
```

![图片.png](https://ask.qcloudimg.com/draft/6211241/nuwzxfo8yv.png)

查看node

![图片.png](https://ask.qcloudimg.com/draft/6211241/mvp7diny6p.png)



**详细搭建过程：**[k8s实践(十四)：Pod驱逐迁移和Node节点维护](https://github.com/loong576/Pode-Eviction-and-Node-Manage.git)
