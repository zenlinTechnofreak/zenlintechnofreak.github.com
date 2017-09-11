---
layout: post
title:  "Kubernetes 网络实战"
date:   2017-09-10 21:08:01
categories: kubernetes
tags: kubernetes network flannel
---

* content
{:toc}
## 前言
本文主要以实例讲解kube-proxy和flannel，以此理解K8S网络。

## K8S的三个网段   

- ## node网段

主要负责K8S集群中的各个node之间通信的网络；

```shell
root@zenlin:~# kubectl get nodes
NAME          STATUS    AGE       VERSION
zenlin        Ready     19d       v1.7.3
zenlinnode1   Ready     19d       v1.7.3
zenlinnode2   Ready     19d       v1.7.3
#以下为各node的物理IP
root@zenlin:~# kubectl get nodes zenlin -o 'jsonpath={.status.addresses[0].address}' 
10.229.43.65
root@zenlin:~# kubectl get nodes zenlinnode1 -o 'jsonpath={.status.addresses[0].address}'
10.229.53.146
root@zenlin:~# kubectl get nodes zenlinnode2 -o 'jsonpath={.status.addresses[0].address
10.229.45.161
```



```shell
root@zenlin:~# kubectl get nodes zenlin -o 'jsonpath={.spec.podCIDR}'
10.244.0.0/24
root@zenlin:~# kubectl get nodes zenlinnode1 -o 'jsonpath={.spec.podCIDR}'
10.244.1.0/24
root@zenlin:~# kubectl get nodes zenlinnode2 -o 'jsonpath={.spec.podCIDR}'
10.244.2.0/24
```

这是K8S的每个node上的podCIDR，用于给每个pod设置ip连接到node的网桥上，这将会在后面的章节中用到。

- ## service网段

  每个新创建的service都会分配到一个service的cluster IP，在笔者的集群中为10.0.0.0这个网段内分配。

```shell
root@zenlin:~# kubectl get svc
NAME            CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
details         10.111.84.67     <none>        9080/TCP                     3d
grafana         10.110.131.254   <none>        3000/TCP                     7d
istio-egress    10.99.236.93     <none>        80/TCP                       7d
kubernetes      10.96.0.1        <none>        443/TCP                      19d
nginx-service   10.105.211.16    <none>        8000/TCP                     9d
php-apache      10.106.156.64    <none>        80/TCP                       13d
productpage     10.104.17.86     <none>        9080/TCP                     3d
prometheus      10.109.48.180    <none>        9090/TCP                     7d
ratings         10.100.168.14    <none>        9080/TCP                     3d
reviews         10.110.130.11    <none>        9080/TCP                     3d
servicegraph    10.104.205.192   <none>        8088/TCP                     7d
zipkin          10.105.183.37    <nodes>       9411:30305/TCP               3d
```

- ## pod网段

  在笔者环境中，安装了fannal隧道网络，则pod网络其实就是fannel 网络，以帮助不同node之间的pod之间进行通信，fannel使整个K8S集群网络扁平化，不管是在node内还是node之间，pod之间的通信都可以通过pod IP进行。

  可以看到，在笔者的环境中，pod的网络都位于10.244.0.0/16网段内，所有的pod将会被分配在这个网段。

```shell
root@zenlin:~# kubectl get po -owide
NAME                                        READY     STATUS             RESTARTS   AGE       IP             NODE
details-v1-3006205406-dnm69                 2/2       Running            0          3d        10.244.1.137   zenlinnode1
grafana-1011650190-1f9h4                    1/1       Running            0          7d        10.244.1.114   zenlinnode1
nginx-deployment-1885164871-dzls7           1/1       Running            0          9d        10.244.1.53    zenlinnode1
nginx-deployment-1885164871-q37zh           1/1       Running            0          9d        10.244.2.199   zenlinnode2
productpage-v1-4256385220-nvq0d             2/2       Running            0          3d        10.244.2.240   zenlinnode2
prometheus-4245872192-xz47s                 1/1       Running            0          7d       
```

##  

## 认识kubeube-proxy和flannel

- ## **kube-proxy**

kube-proxy就是一个简单的网络代理和负载均衡器，主要实现了内部**从pod到service和外部从nodePort到service的访问。**

kube-proxy主要有userspace和iptables两种模式，默认使用的是iptables这种模式，笔者的环境也是，故，我们只介绍iptables模式。

![kubeproxy-iptables](/assets/images/kubeproxy-iptables.jpg){: .align-center}

在这种模式下，kube-proxy监听kubernetes master添加和删除services和端点对象。对每个服务，kube-proxy安装iptalbes规则，用来捕获service的cluster IP和port的流量，并将重定向到serivce的后端集合之一（如pod），对于每个端点对象，它选择一个后端pod来安装iptables 规则。

默认情况下，后端的选择是随机的。可以通过将service.spec.sessionAffinity设置为“ClientIP”（默认为“无”）来选择基于客户端IP的会话关联。

由于iptables实际上是走的kernel态的Netflix，这会比userspace这种模式更快，更可靠。如果最初选择的pod不响应，则iptables代理不能自动重试。

具体关于kube-proxy的更多信息可参考官网：[services-networking/service](https://kubernetes.io/docs/concepts/services-networking/service/)

- ## **kube-proxy 实例探测**

通过K8S 创建nginx service，部署两个po，如下：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 8000
    targetPort: 80
    protocol: TCP
  selector:
    app: nginx
```

通过K8S可以看到，创建了两个po和一个svc，两个pod分别位于两个node，cluster-ip分别为10.244.1.144 和  10.244.2.245，svc的cluster-ip为10.105.211.16

```shell
root@zenlin:~# kubectl get svc |grep nginx
nginx-service   10.105.211.16    <none>        8000/TCP                                                 14d
```

```shell
root@zenlin:~# kubectl get po -owide -n zenlin 
NAME                                    READY     STATUS             RESTARTS   AGE       IP             NODE
ubuntu-k8s-deployment-623915768-ctbd2   0/1       Running			 0          7d        10.244.1.144   zenlinnode1
ubuntu-k8s-deployment-623915768-hsdrk   1/1       Running            0          7d        10.244.2.245   zenlinnode2
```

分别在三个节点上，查看kube-proxy下发的iptables规则：

```shell
root@zenlin:~# iptables-save |grep nginx
-A KUBE-SEP-RGXD32JDVTBAD2NU -s 10.244.1.53/32 -m comment --comment "default/nginx-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-RGXD32JDVTBAD2NU -p tcp -m comment --comment "default/nginx-service:" -m tcp -j DNAT --to-destination 10.244.1.53:80
# 先做mark，再做dnat，将serviceip（10.244.1.53/32）改为pod ip（10.244.1.53:80），送给pod。
-A KUBE-SEP-YEFU7DZX4NWUTRQS -s 10.244.2.199/32 -m comment --comment "default/nginx-service:" -j KUBE-MARK-MASQ
-A KUBE-SEP-YEFU7DZX4NWUTRQS -p tcp -m comment --comment "default/nginx-service:" -m tcp -j DNAT --to-destination 10.244.2.199:80
# 先做mark，再做dnat，将serviceip（10.244.1.53/32）改为pod ip（10.244.2.199:80），送给pod。
-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.105.211.16/32 -p tcp -m comment --comment "default/nginx-service: cluster IP" -m tcp --dport 8000 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.105.211.16/32 -p tcp -m comment --comment "default/nginx-service: cluster IP" -m tcp --dport 8000 -j KUBE-SVC-GKN7Y2BSGW4NJTYL
-A KUBE-SVC-GKN7Y2BSGW4NJTYL -m comment --comment "default/nginx-service:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-RGXD32JDVTBAD2NU
-A KUBE-SVC-GKN7Y2BSGW4NJTYL -m comment --comment "default/nginx-service:" -j KUBE-SEP-YEFU7DZX4NWUTRQS
```



-A KUBE-SERVICES ! -s 10.244.0.0/16 -d 10.105.211.16/32 -p tcp -m comment --comment "default/nginx-service: cluster IP" -m tcp --dport 8000 -j KUBE-MARK-MASQ
-A KUBE-SERVICES -d 10.105.211.16/32 -p tcp -m comment --comment "default/nginx-service: cluster IP" -m tcp --dport 8000 -j **KUBE-SVC-GKN7Y2BSGW4NJTYL**

iptables 在看到目的地为 **10.105.211.16（nginx-service的cluster ip）**的目的地端口80端口的tcp报文时，走**KUBE-SVC-GKN7Y2BSGW4NJTYL** 规则；

-A **KUBE-SVC-GKN7Y2BSGW4NJTYL** -m comment --comment "default/nginx-service:" -m statistic --mode random --probability 0.50000000000 **-j KUBE-SEP-RGXD32JDVTBAD2NU**
-A **KUBE-SVC-GKN7Y2BSGW4NJTYL -**m comment --comment "default/nginx-service:" **-j KUBE-SEP-YEFU7DZX4NWUTRQS**

以上两条，实现**按照50%的统计概率随机匹配**到KUBE-SEP-RGXD32JDVTBAD2NU和KUBE-SEP-YEFU7DZX4NWUTRQS这两条规则。

-A **KUBE-SEP-RGXD32JDVTBAD2NU** -s 10.244.1.53/32 -m comment --comment "default/nginx-service:" -j KUBE-MARK-MASQ
-A **KUBE-SEP-RGXD32JDVTBAD2NU** -p tcp -m comment --comment "default/nginx-service:" -m tcp -j DNAT --to-destination 10.244.1.53:80

先做mark，再做dnat，将serviceip（10.244.1.53/32）**改为pod ip（10.244.1.53:80），送给pod。**

-A **KUBE-SEP-YEFU7DZX4NWUTRQS** -s 10.244.2.199/32 -m comment --comment "default/nginx-service:" -j KUBE-MARK-MASQ
-A **KUBE-SEP-YEFU7DZX4NWUTRQS** -p tcp -m comment --comment "default/nginx-service:" -m tcp -j DNAT --to-destination 10.244.2.199:80

先做mark，再做dnat，将serviceip（10.244.1.53/32）**改为pod ip（10.244.2.199:80），送给pod。**

- ## **flannel**

默认的K8S网络中，集群中不同的node的docker container会被node上的docker daemon分配IP，这将直接导致不同node上的docker container IP存在重叠，但是并不是全集群唯一的IP，随着node数量的增加，一旦出现网络问题之后跟踪定位会显得复杂度大大增大。

Flannel属于隧道网络（Overlay Networking）的一种实现，可以让集群中的不同node创建的docker container 都被分配全集群唯一的Cluster IP，使能“同属同一个内网” 且 “IP地址不重复”，并且不同节点上的容器能够直接通过内网IP通信。

Flannel目前已经支持udp、vxlan、host-gw、aws-vpc、gce和alloc路由等数据转发方式，笔者的环境上使用的是默认的UDP，采用以下的通信方式进行通信：

![flannel-udp](/assets/images/flannel-udp.jpg){: .align-center}

源主机的flanneld服务将数据内容进行UDP封装后投递给目的节点的flanneld服务处理。

1. Flannel通过etcd给每个节点分配可用的IP地址段，并让docker daemon感知，这样docker daemon只在该网段内为所在node节点上的contaienr分配IP。flannel通过保存在etcd中的记录确保分配给不同node的docker daemon的网段不会重复。
2. 从源node到目的node的路由是如何处理，我们在后面的章节中，用实例来讲解。


- ## **flannel实例探测**

我们通过K8S创建一个ubuntu镜像的deployment，两个pod：

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: ubuntu-deployment
  namespace: zenlin
spec:
  replicas: 2
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: ubuntu
    spec:
      containers:
      - image: ubuntu:zenlin
        imagePullPolicy: IfNotPresent
        name: ubuntu
        command: ["/bin/sh"]
        args: ["-c", "while true; do echo hello; sleep 10;done"]
        ports:
        - containerPort: 80
          protocol: TCP
        resources: {}
        securityContext:
          privileged: true
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      terminationGracePeriodSeconds: 30                                      
```

```bash
root@zenlin:~#  kubectl get po -owide -n zenlin |grep ubuntu
ubuntu-k8s-deployment-623915768-fnssv   1/1       Running   0          8h        10.244.1.145   zenlinnode1
ubuntu-k8s-deployment-623915768-nwzk9   1/1       Running   0          8h        10.244.2.246   zenlinnode2
```

可观测到，两个pod分为位于两个node上，并且cluster ip分别为10.244.1.145 和  10.244.2.246

我们进入node1上的pod ubuntu-k8s-deployment-623915768-nwzk9：

```bash
root@zenlin:~# kubectl exec -it ubuntu-k8s-deployment-623915768-fnssv /bin/bash -n zenlin
root@ubuntu-k8s-deployment-623915768-fnssv:/# ifconfig
eth0      Link encap:Ethernet  HWaddr 9a:59:82:99:75:d6  
          inet addr:10.244.1.145  Bcast:0.0.0.0  Mask:255.255.255.255
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:12 errors:0 dropped:0 overruns:0 frame:0
          TX packets:4 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0 
          RX bytes:928 (928.0 B)  TX bytes:280 (280.0 B)

lo        Link encap:Local Loopback  
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:1 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1 
          RX bytes:28 (28.0 B)  TX bytes:28 (28.0 B)

root@ubuntu-k8s-deployment-623915768-fnssv:/#
```

再查看node1和node2节点上的路由分别为：

```bash
root@zenlinnode1:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.229.52.1     0.0.0.0         UG    0      0        0 eth0
10.229.52.0     0.0.0.0         255.255.254.0   U     0      0        0 eth0
10.244.0.0      0.0.0.0         255.255.0.0     U     0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
root@zenlinnode1:~# 
```

```bash
root@zenlinnode2:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.229.44.1     0.0.0.0         UG    0      0        0 eth0
10.229.44.0     0.0.0.0         255.255.254.0   U     0      0        0 eth0
10.244.0.0      0.0.0.0         255.255.0.0     U     0      0        0 flannel.1
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
```

当flannel被安装后，会在K8S的所有节点上安装上flanneld服务，各flanneld之间通过etcd查询各自分配的网段，并且彼此将进行通信。

以上，当一个数据包从ubuntu-k8s-deployment-623915768-fnssv 这个pod发往ubuntu-k8s-deployment-623915768-nwzk9 这个pod时，流程如下：

1. 源数据包从容器内出发到达本机docker0网桥（本机docker通信请读者自行脑补哈），这个时候，根据发送节点的路由表，发现ip 10.244.1.145 只与 10.244.0.0（node1上的falnnel1.1）匹配，数据从docker0出来后被投递到flannel0；
2. node1节点上的flanneld会对数据包进行UDP封装形成 `playlod | Inner IP | UDP | Outer IP ` 的数据包；
3. node1的flanneld会根据Outer IP投递给node2的flanneld服务；
4. node2上的flannld拿到数据后，进行解包，数据包进入node2上的flannel.1虚拟网卡；
5. 数据包被转送给docker0虚拟网卡，剩下的部分就像本节点内的容器通信一样，达到目标容器。
