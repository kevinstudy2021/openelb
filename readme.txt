
# 注意如果不能获取k8s.gcr.io镜像，需要替换其中的镜像
kubectl apply -f https://raw.githubusercontent.com/openelb/openelb/master/deploy/openelb.yaml

# 查看
☸ ➜ kubectl get pods -n openelb-system              
NAME                                READY   STATUS      RESTARTS      AGE
openelb-admission-create--1-cf857   0/1     Completed   0             58m
openelb-admission-patch--1-dhgrq    0/1     Completed   2             58m
openelb-manager-848495684-nppkr     1/1     Running     1 (35m ago)   48m
openelb-manager-848495684-svn7z     1/1     Running     1 (35m ago)   48m
☸ ➜ kubectl get validatingwebhookconfiguration       
NAME                                      WEBHOOKS   AGE
openelb-admission                         1          62m
☸ ➜ kubectl get mutatingwebhookconfigurations        
NAME                                    WEBHOOKS   AGE
openelb-admission                       1          62m

☸ ➜ kubectl get crd |grep kubesphere
bgpconfs.network.kubesphere.io           2022-04-10T08:01:18Z
bgppeers.network.kubesphere.io           2022-04-10T08:01:18Z
eips.network.kubesphere.io               2022-04-10T08:01:18Z

# 接下来我们来演示下如何使用 layer2 模式的 OpenELB，首先需要保证所有 Kubernetes 集群节点必须在同一个二层网络（在同一个路由器下）
# LoadBalance IP 要与node节点的IP在同一网段，但不能重叠


kubectl edit configmap kube-proxy -n kube-system
......
ipvs:
  strictARP: true
......

然后执行下面的命令重启 kube-proxy 组件即可：

☸ ➜ kubectl rollout restart daemonset kube-proxy -n kube-system

# 如果安装 OpenELB 的节点有多个网卡，则需要指定 OpenELB 在二层模式下使用的网卡，如果节点只有一个网卡，则可以跳过此步骤，假设安装了 OpenELB 的 master1 节点有两个网卡（eth0 192.168.0.2 和 ens33 192.168.0.111），并且 eth0 192.168.0.2 将用于 OpenELB，那么需要为 master1 节点添加一个 annotation 来指定网卡：

☸ ➜ kubectl annotate nodes master1 layer2.openelb.kubesphere.io/v1alpha1="192.168.0.2"

# 接下来就可以创建一个 Eip 对象来充当 OpenELB 的 IP 地址池了，创建一个如下所示的资源对象：

apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
  name: eip-pool
spec:
  address: 192.168.0.100-192.168.0.108
  protocol: layer2
  disable: false
  interface: ens33

# 格式
IP地址，例如 192.168.0.100
IP地址/子网掩码，例如 192.168.0.0/24
IP地址1-IP地址2，例如192.168.0.91-192.168.0.100

创建完成 Eip 对象后可以通过 Status 来查看该 IP 池的具体状态：

☸ ➜ kubectl get eip          
NAME       CIDR                          USAGE   TOTAL
eip-pool   192.168.0.100-192.168.0.108   0       9
☸ ➜ kubectl get eip eip-pool -oyaml
apiVersion: network.kubesphere.io/v1alpha2
kind: Eip
metadata:
  finalizers:
  - finalizer.ipam.kubesphere.io/v1alpha1
  name: eip-pool
spec:
  address: 192.168.0.100-192.168.0.108
  interface: ens33
  protocol: layer2
status:
  firstIP: 192.168.0.100
  lastIP: 192.168.0.108
  poolSize: 9
  ready: true
  v4: true



到这里 LB 的地址池就准备好了，接下来我们创建一个简单的服务，通过 LB 来进行暴露，如下所示：

# openelb-nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
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
        ports:
        - containerPort: 80
这里部署一个简单的 nginx 服务：

☸ ➜ kubectl apply -f openelb-nginx.yaml 
☸ ➜ kubectl get pods                  
NAME                     READY   STATUS    RESTARTS      AGE
nginx-7848d4b86f-zmm8l   1/1     Running   0             42s


# 然后创建一个 LoadBalancer 类型的 Service 来暴露我们的 nginx 服务，如下所示：

# openelb-nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  annotations:
    lb.kubesphere.io/v1alpha1: openelb
    protocol.openelb.kubesphere.io/v1alpha1: layer2
    eip.openelb.kubesphere.io/v1alpha2: eip-pool
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
    - name: http
      port: 80
      targetPort: 80

# 注意这里我们为 Service 添加了几个 annotations 注解：

lb.kubesphere.io/v1alpha1: openelb 用来指定该 Service 使用 OpenELB
protocol.openelb.kubesphere.io/v1alpha1: layer2 表示指定 OpenELB 用于 Layer2 模式
eip.openelb.kubesphere.io/v1alpha2: eip-pool 用来指定了 OpenELB 使用的 Eip 对象，如果未配置此注解，OpenELB 会自动使用与协议匹配的第一个可用 Eip 对象，此外也可以删除此注解并添加 spec:loadBalancerIP 字段（例如 spec:loadBalancerIP: 192.168.0.108）以将特定 IP 地址分配给 Service。

# 同样直接创建上面的 Service：
☸ ➜ kubectl apply -f openelb-nginx-svc.yaml                
service/nginx created
☸ ➜ kubectl get svc nginx                   
NAME    TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
nginx   LoadBalancer   10.100.126.91   192.168.0.101   80:31555/TCP   4s













