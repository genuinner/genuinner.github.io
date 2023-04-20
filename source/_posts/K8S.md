---
title: 容器化
ahthor: wood
---
## Linux容器
### 什么是LXC
容器相当于隔离用户空间的进程或进程组，拥有独立的用户空间，共享内核空间。

容器间的隔离通过命名空间和控制组两个内核功能实现的。
### 命名空间NameSpace
命名空间是Linux内核用来隔离内核资源的方式，为容器设置了边界，让容器拥有了自己的挂载点、用户、IP 地址、以及进程管理等。

Linux 中的重要的命名空间有：
+ pid namespace：负责隔离进程（PID：Process ID）。
+ net 命名空间：负责管理网络接口（NET：网络）。
+ ipc 命名空间：负责管理对 IPC 资源的访问（IPC：进程间通信）。
+ mnt 命名空间：负责管理文件系统挂载点（MNT：Mount）。
+ uts 命名空间：隔离内核和版本标识符。（UTS：Unix 分时系统）。
+ usr 命名空间：隔离用户 ID。简单来说，它隔离了主机和容器之间的用户 ID。
+ Cgroup 命名空间：将控制组信息与容器进程隔离。
### 控制组CGroups
Linux 控制组管理着容器使用的资源，我们可以限制容器的 CPU、内存、网络和 IO 资源。

## Docker
### 什么是docker
Docker是一个开源的容器引擎，它使用Linux内核功能在系统上创建容器。它最初是构建在LXC上的，后来用自己的容器运行时替换了LXC。
### 优点
+ 一致的运行环境，避免出现因环境配置问题不同导致的问题
+ 快速的启动时间
+ 隔离性
+ 弹性扩展
+ 方便迁移
+ 持续交付和部署
### 基础概念
+ 镜像：Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。
+ 仓库：集中存放镜像的地方，分为公开服务(如docker hub)和私有服务(本地搭建)。
+ 容器：容器是镜像运行时的实体，本质是隔离的进程。
### 架构

### 安装
#### 脚本
> 适合测试环境使用  
> curl -fsSL https://get.docker.com -o get-docker.sh  
> sudo sh ./get-docker.sh --mirror Aliyun
#### yum
> sudo yum install -y yum-utils  
> sudo yum-config-manager \  
> --add-repo \  
https://download.docker.com/linux/centos/docker-ce.repo  
> sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
#### 添加镜像
> sudo mkdir -p /etc/docker  
> sudo tee /etc/docker/daemon.json <<-'EOF'  
> {  
"registry-mirrors": ["https://1tov2mxg.mirror.aliyuncs.com"]  
> }  
> EOF  
> sudo systemctl daemon-reload  
> sudo systemctl restart docker  
### 命令
#### 镜像
+ docker images \<name>：查看本地镜像
+ docker pull \<name>:\<tag> ：拉取镜像
+ docker search \<name>:\<tag> ：查找镜像
+ docker image rm [id | \<name>:\<tag>] ：删除镜像，可以简化为docker rmi
+ docker save image -o bak.tar ：备份镜像
+ docker load -i bak.tar ：加载镜像
#### 容器
+ docker run [-d] [-v path:container_path] [-p 8080:8080] [--name tomcat1] 
+ docker ps [-q -a] ：查询运行的容器
+ docker start/restart/stop/kill/rm id ：操作容器
+ docker logs [-f] id ：查看日志
+ docker exec -it id bash ：进入容器
+ docker cp path id:path ：复制文件到容器内
+ docker inspect id ：查看容器状态
+ docker commit -m "description" -a "author" [id | name] image_name:tag ：容器打包成镜像
### 使用Idea开发
#### 1. 创建Dockerfile
```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:17-jdk-jammy

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:resolve

COPY src ./src

CMD ["./mvnw", "spring-boot:run"]
```
#### 2. 创建.dockerignore文件
```
target
```
#### 3. 构建镜像
```shell
docker build --tag java-docker
```
#### 4. 启动容器
```shell
docker run -d -p 8080:8080 java-docker
```
#### 5. 多容器通信
> docker network create mysqlnet  
> 指定容器网络  
> docker run --rm -d \  
> --name springboot-server \  
> --network mysqlnet \  
> -e MYSQL_URL=jdbc:mysql://mysqlserver/petclinic \  
> -p 8080:8080 java-docker
#### 6. 多阶段构建
通过--target指定最终构建的阶段，默认构建最后一个阶段的镜像。
```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:17-jdk-jammy as base
WORKDIR /app
COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:resolve
COPY src ./src

FROM base as development
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql", "-Dspring-boot.run.jvmArguments='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000'"]

FROM base as build
RUN ./mvnw package

FROM eclipse-temurin:17-jre-jammy as production
EXPOSE 8080
COPY --from=build /app/target/spring-petclinic-*.jar /spring-petclinic.jar
CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/spring-petclinic.jar"]
```
#### 7. docker compose
创建docker-compose.dev.yml文件

```yaml
version: '3.8'
services:
  petclinic:
    build:
      context: ../..
      target: development
    ports:
      - "8000:8000"
      - "8080:8080"
    environment:
      - SERVER_PORT=8080
      - MYSQL_URL=jdbc:mysql://mysqlserver/petclinic
    volumes:
      - ./:/app
    depends_on:
      - mysqlserver

  mysqlserver:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - mysql_data:/var/lib/mysql
      - mysql_config:/etc/mysql/conf.d
volumes:
  mysql_data:
  mysql_config:
```
通过docker-compose启动
>  docker-compose -f docker-compose.dev.yml up --build
#### 8. 远程调试
添加remote jvm debugger并启动连接到docker
![](docker-debugger.png)
#### 9. CI/CD

#### 10. Registry
通过docker启动一个注册服务
> docker run -d \  
> -p 5000:5000 \  
> --restart=always \  
> --name registry \  
> -v /mnt/registry:/var/lib/registry \  
> registry:2

从dockerhub拉取镜像放入本地注册服务
> docker pull ubuntu:16.04  
> docker tag ubuntu:16.04 localhost:5000/my-ubuntu  
> docker push localhost:5000/my-ubuntu  
> docker pull localhost:5000/my-ubuntu  
## K8S

### K8S架构
![](components-of-kubernetes.svg)
一个K8S集群由Control Panel(控制平面)和Node(节点)组成。  
**控制平面**：负责管理工作节点和维护集群状态。  
**工作节点**：负责执行控制平面分配的任务，运行实际的应用和工作负载。
#### 控制平面组件
+ kube-apiserver：控制平面的前端，用于处理内部和外部请求。
+ kube-scheduler：综合考虑资源需求和集群运行状况，为pod分配工作节点。
+ etcd：键值对数据库，用于存储配置数据和集群状态信息。
+ kube-controller-manager：负责实际运行集群，包括多个控制器
  + 节点控制器：负责在节点故障时进行通知和响应
  + 任务控制器：监测代表一次性任务的Job对象，创建Pods来运行这些任务
  + 端点控制器：填充端点对象（即Services和Pods之间的连接）
  + 服务账户和令牌控制器：为新的命名空间创建默认账户和API访问令牌
+ cloud-controller-manager：运行将集群连接到云提供商的API之上。
#### 节点组件
+ kubelet：集群在节点上的代理，保证容器都运行在Pod中。
+ kube-proxy：集群在节点上的网络代理，实现Services概念的一部分。维护节点网络规则和流量转发，实现从内部或外部与Pod进行通信。
+ 容器运行时：负责运行容器的软件。k8s支持containerd等任何实现CRI(Container Runtime Interface)的容器运行时。
节点上的组件包括kubelet、容器运行时和kube-proxy。
#### Pod
+ Pod是k8s中的最小调度单元，k8s直接管理pod而不是容器
+ 同一pod中的容器会被安排到同一节点
+ 每个pod有唯一的Ip地址，所有ip共享一个ip地址和端口空间，pod内容器可以使用localhost互相通信
#### deployment
deployment是对replicaset和pod更高级的抽象，它使pod拥有多副本、自愈、扩缩容、滚动升级等能力。

RepilcaSet是一个Pod的集合，确保任何时间有指定数量的Pod副本在运行，通常在Deployment中声明。
#### Services
Services为一组Pods提供相同的DNS名，并在他们之间进行负载均衡，k8s为pod提供的ip地址可能会发生变化，通过service名访问服务就不需要担心pod的ip发生变化。

ServiceType:
+ ClusterIp：服务公开在集群内部，集群内主机通过IP访问，集群内Pod可以通过service名访问
+ NodePort：通过每个节点的主机和端口暴露服务，集群外部通过任意节点主机+端口来访问服务
+ ExternalName：将集群外部网络引入集群内部
+ LoadBalancer：使用云服务商提供的负载均衡器暴露服务
#### NameSpace
通过命名空间将资源划分为相互隔离的组，例如可以设置开发、测试、生成等命名空间。

初始命名空间：
+ default：默认命名空间，未指定ns时默认分配到default
+ kube-system：k8s系统对象使用的命名空间
+ kube-public：公共命名空间，所有用户都可以读取它
+ kube-node-lease：租约对象使用的命名空间，每个节点关联一个lease对象，lease通过心跳检测节点是否发生故障。
### 使用kubeadm搭建集群
#### 安装要求
+ 2GB及以上内存
+ 2及以上CPU核心数
+ 节点网络互通
+ 主机名、MAC地址、product_uuid不可重复
    + cat /sys/class/dmi/id/product_uuid查看uuid
    + ip link查看MAC
+ 开启必需端口
+ 禁用交换分区
    + swapoff -a临时禁用
    + vi /etc/fstab永久禁用
#### 通用配置
```shell
# 禁用swap
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab

# 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 时间同步
yum install -y ntpdate
ntpdate ntp.aliyun.com

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 添加hosts
cat <<EOF >> /etc/hosts
192.168.108.100 master
192.168.108.102 slave1
192.168.108.104 slave2
EOF

# 转发桥接流量到iptables链上
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

# 安装docker并修改cgroup=systemd
curl -fsSL https://get.docker.com -o get-docker.sh  
sh ./get-docker.sh --mirror Aliyun
mkdir -p /etc/docker  
cat > /etc/docker/daemon.conf <<EOF
{
  "registry-mirrors": ["https://1tov2mxg.mirror.aliyuncs.com"] 
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  }
}
EOF

systemctl daemon-reload
systemctl restart docker
systemctl enable docker


# 配置k8s源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# 安装kubeadm、kubelet、kubectl
yum install -y kubelet kubeadm kubectl 
systemctl enable --now kubelet
systemctl start kubelet

# 配置containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml 
sed -i 's/sandbox_image = .*/sandbox_image = "registry.aliyuncs.com\/google_containers\/pause:3.6"/' /etc/containerd/config.toml
#[plugins."io.containerd.grpc.v1.cri".registry.mirrors]后添加镜像地址
#[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
#  endpoint = ["https://1tov2mxg.mirror.aliyuncs.com/"]
#[plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
#  endpoint = ["registry.aliyuncs.com/google_containers"]
# 配置本地私库可能需要额外配置hosts.toml
systemctl restart containerd


# 加载网络模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter


```
#### 初始化控制平面
```shell
# 初始化master
kubeadm init \
--apiserver-advertise-address=192.168.108.100 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version=v1.26.3 \
--control-plane-endpoint=master \
--pod-network-cidr=10.244.0.0/16 \
--service-cidr=10.96.0.0/12
# 根据提示执行命令
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
# 安装网络插件calico
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml

# 或者安装flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


```
#### 搭建cp高可用集群
1. 为apiserver创建高可用负载均衡器，keepalived+haproxy参考连接
https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md#options-for-software-load-balancing
2. 其他cp加入集群
#### 加入工作节点
```shell
kubeadm join master1:6443 --token 2h1s44.hgu640y31j544y4u \
	--discovery-token-ca-cert-hash sha256:59405e3c9d154d886f3f99a14ff99e8956ee50d91986beb73c2530cce4568653
```

### 二进制搭建k8s集群


### kubectl部署服务
#### Pod
```shell
# 创建pod
kubectl run mynginx --image=nginx:1.22 [ -it --rm ]#运行一次性pod
# 查看pod
kubectl get pod -owide
# 查看pod
kubectl describe pod mynginx
# 进入pod
kubectl exec -it mynginx -- /bin/bash
# 删除pod
kubectl delete pod mynginx
```
#### Deployment
```shell
# 创建deployment
kubectl create deployment nginx-deploy --image=nginx:1.22 --replicas=3
# 查看deploy
kubectl get deploy
# 查看replicaset
kubectl get replicaSet
# 扩缩容
kubectl scale deployment nginx-deploy --replicas=2
# 自动缩放(需要声明pod的资源限制，同时使用metrics server服务)
kubectl autoscale deployment/nginx-deploy --min=3 --max=10 --cpu-percent=75
# 查看自动缩放
kubectl get hpa
# 删除自动缩放
kubectl delete hpa nginx-deploy
# 查看deploy容器版本
kubectl get deploy -owide
# 滚动更新
kubectl set image deploy/nginx-deploy nginx=nginx:1.23
# 查看历史
kubectl rollout history deploy/nginx-deploy
# 查看指定版本
kubectl rollout history deploy/nginx-deploy --revision=1
# 回滚到指定版本
kubectl rollout undo deploy/nginx-deploy --to-revision=1
```

#### Service
```shell
# 创建服务
kubectl expose deploy/nginx-deploy --name=nginx-service --port=8080 --target-port=80
# 查看服务
kubectl get service
# 集群主机通过service的ip访问服务
curl 10.108.215.213:8080
# 在集群内部Pod通过service name访问服务
kubectl run test -it --image=nginx:1.22 --rm -- bash
curl nginx-service:8080
# 创建NodePort类型service
kubectl expose deploy/nginx-deploy --name=nginx-outside --type=NodePort --port=8081 --target-port=80
# 查看service
kubectl get service
#NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
#kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP          7d17h
#nginx-outside   NodePort    10.99.162.15     <none>        8081:31237/TCP   3s
#nginx-service   ClusterIP   10.108.215.213   <none>        8080/TCP         14m

# 外部主机通过节点主机+端口访问
curl 192.168.108.100:31237
```

#### NameSpace
```shell
# 查看ns
kubectl get ns
# 创建ns
kubectl create ns dev
# 指定pod命名空间
kubectl run nginx --image=nginx:1.22 -n=dev
# 设置默认命名空间
kubectl config set-context $(kubectl config current-context) --namespace=dev
# 删除命名空间
kubectl delete ns dev
```


### 声明式部署服务
通过yaml文件配置k8s对象，通常需要配置如下字段：
+ apiVersion：k8s API版本
+ kind：对象类别，如Pod、Deployment、Service、ReplicaSet等
+ metadata：对象的元数据，包括name、labels、UID和可选的namespace
+ spce：对象的配置
#### 标签与选择器
```yaml
# 基于等值的选择器
selector:
  matchLabels:
    compoent: redis
    version: 7.0
# 基于集合
#selector:
  #matchExpressions:
    #- {key: tier, operator: In, values: [cache, backend]}
    #- {key: env, operator: NotIn, values: [dev, prod]}
```

### crictl与ctr
#### crictl
k8s提供的用来检查和调试节点容器运行时和应用程序的运行时命令行接口。
```shell
# 查看容器
crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

```
#### ctr
containerd提供的命令行工具，可以用来导入导出镜像。
```shell
# 查看镜像，k8s使用命名空间为k8s.io
ctr -n k8s.io image ls
```

### 有状态应用
#### 创建MySQL实例
配置mysql的yaml文件，kubectl apply -f mysql-demo.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-demo
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "123456"
    volumeMounts:
      - mountPath: /var/lib/mysql
        name: data-volume
  volumes:
    - name: data-volume
      hostPath:
        path: /home/mysql/data
        type: DirectoryOrCreate
```
#### 通过ConfigMap指定配置文件
mysql有时需要指定配置文件，在k8s中创建的容器可能被调度到任何节点，不能保证所有节点都有配置文件，需要通过ConfigMap的方式指定配置文件。

ConfigMap保存在etcd中，可以用作环境变量、命令行参数或存储卷，用于将配置信息与容器镜像解耦，保存的数据不能超过1MB。
```shell
# 编辑configmap
kubectl edit cm mysql-config
```
将ConfigMap挂载为配置文件
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "123456"
    volumeMounts:
      - mountPath: /var/lib/mysql
        name: data-volume
      - mountPath: /etc/mysql/conf.d
        name: conf-volume
        readOnly: true
  volumes:
    - name: data-volume
      hostPath:
        path: /home/mysql/data
        type: DirectoryOrCreate
    - name: conf-volume
      configMap:
        name: mysql-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-config
data:
  mysql.cnf: |
    [mysqld]
    character-set-server=utf8mb4
    collation-server=utf8mb4_general_ci
    init-connect='SET NAMES utf8mb4'
    
    [client]
    default-character-set=utf8mb4
    
    [mysql]
    default-character-set=utf8mb4

```
#### Secret管理加密信息
创建secret并用作环境变量mysql的密码，不同于挂载卷，环境变量不会自动更新，需要重启pod
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-password
type: Opaque
data:
  PASSWORD: MTIzNDU2Cg==
---
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
  labels:
    app: mysql
spec:
  containers:
  - name: mysql
    image: mysql:5.7
    env:
    - name: MYSQL_ROOT_PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysql-password
          key: PASSWORD
          optional: false
```

#### 临时卷
临时卷随Pod一起创建和删除，位于节点的/var/lib/kubelet/pods/路径下

主要包括三种类型：
+ emptyDir：初始内容为空的本地临时目录，可以通过设置emptyDir.medium为Memory使用内存空间，通常用作日志或缓存
+ configMap：通过configMap注入的配置文件
+ secret：通过secret注入的加密文件

#### 持久卷
+ PV：集群的一块储存，可以由管理员事先制备，或使用存储类动态创建，是集群资源。
+ PVC：用户对存储资源的申领，用户不直接操作PV而是通过PVC来申请资源。

静态持久卷
```shell
# 创建存储类、local持久卷、pvc
kubectl apply -f kube-pv.yaml
# 去worker手动创建挂载目录
mkdir /mnt/disks/ssd1
# 修改pod使用pvc
kubectl apply -f mysql-pod.yaml
```
```yaml
# kube-pv.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/ssd1
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - slave1
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: local-storage

```
```yaml
  volumes:
    - name: data-volume
      persistentVolumeClaim:
        claimName: myclaim
```

#### 动态创建持久卷
动态创建需要使用存储类，用户在PVC中指定StorageClass来自动创建卷。
local卷目前不支持动态制备，需要创建外部provisioner，这里使用rancher.io/local-path
```shell
# 创建 external provisioner
kubectl apply -f local-path-sc.yaml
# 修改pvc的storageClassName
kubectl apply -f local-pvc.yaml
# 启动pod
kubectl apply -f mysql-pod.yaml
```



#### StatefulSet
Deployment用于部署无状态应用，StatefulSet用来部署有状态应用。
通过无头服务避免service的负载均衡，直接通过指定容器的DNS来访问
```shell
# 部署sts
kubectl apply -f statefulset.yaml

#nslookup mysql-svc
#Server:    10.96.0.10
#Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

#Name:      mysql-svc
#Address 1: 10.244.2.12 mysql-sts-0.mysql-svc.default.svc.cluster.local
#Address 2: 10.244.2.14 mysql-sts-2.mysql-svc.default.svc.cluster.local
#Address 3: 10.244.1.15 mysql-sts-1.mysql-svc.default.svc.cluster.local

```


#### mysql主从复制
https://kubernetes.io/zh-cn/docs/tasks/run-application/run-replicated-stateful-application/
```shell
kubectl apply -f mysql-replica.yaml
```

#### 端口转发
支持临时通过访问主机端口来访问内部容器
```shell
kubectl port-forward pods/mysql-sts-0 --address=192.168.108.100 33060:3306
# 通过192.168.108.100:33060访问数据库
```

### Helm
Heml是k8s应用的包管理工具，类似于CentOS中的yum，使用chart来封装k8s的yaml文件，可以实现快速的部署应用。

通过https://artifacthub.io/查找服务
```shell
# 创建集群
helm install cluster -f values.yaml bitnami/mysql
# 删除集群
helm delete cluster
```

### Ingress
ingress对公开到外部的服务进行统一的反向代理，是外部访问集群内服务的统一入口

提供裸机环境Ingress-Nginx的部署方式
https://kubernetes.github.io/ingress-nginx/deploy/baremetal/
#### MetalLB+LoadBalancer

```shell
# registry.k8s.io无法访问，从docker.io下载镜像
helm upgrade --install ingress-nginx ingress-nginx \
 --repo https://kubernetes.github.io/ingress-nginx \
 --namespace ingress-nginx --create-namespace \
 --set controller.admissionWebhooks.patch.image.registry=docker.io \
 --set controller.admissionWebhooks.patch.image.image=dyrnq/kube-webhook-certgen \
 --set controller.image.image=dyrnq/ingress-nginx-controller \
 --set controller.image.registry=docker.io
# GitHub无法访问可以指定本地chart
helm upgrade ingress-nginx ./ingress-nginx-4.6.0.tgz --namespace ingress-nginx --install --create-namespace

# 或者提前加载到containerd
ctr -n=k8s.io image pull docker.io/dyrnq/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794
# k8s可能使用image@digest的方式而不是image:tag，使用docker打包会丢失digest信息
ctr -n=k8s.io image tag docker.io/dyrnq/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794 registry.k8s.io/ingress-nginx/kube-webhook-certgen@sha256:01d181618f270f2a96c04006f33b2699ad3ccb02da48d0f89b22abce084b292f
# ctr -n=k8s.io image export --platform linux/amd64 webhook.tar docker.io/dyrnq/kube-webhook-certgen:v20230312-helm-chart-4.5.2-28-g66a760794
# ctr -n=k8s.io image import webhook.tar

# 非云提供商环境需要配置MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
# 分配ip池
kubectl apply -f MetalLB-ippool.yaml

# 发现LoadBalancer分配了EXTERNAL-IP
kubectl -n ingress-nginx get svc
#NAME                                 TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
#ingress-nginx-controller             LoadBalancer   10.98.68.187    192.168.108.200   80:31876/TCP,443:31180/TCP   4h19m
#ingress-nginx-controller-admission   ClusterIP      10.109.178.71   <none>         443/TCP                      4h19m

# 测试Ingress
kubectl create deployment demo --image=httpd --port=80
kubectl expose deployment demo
kubectl create ingress demo-localhost --class=nginx \
  --rule="demo.localdev.me/*=demo:80"
# 访问
curl -D- http://192.168.108.200 -H 'Host: demo.localdev.me'

```
#### DeamonSet+HostNetwork
 将pod直接映射到host端口，无需公开服务

#### NodePort
 暴露为NodePort服务，通过任意节点NAT转发到pod
