---
layout:     post
title:      利用 kube-vip 部署一个高可用的 Kubernetes 集群
date:       2022-01-08
author:     Charlie Lin
catalog:    true
tags:
    - Kubernetes
    - kube-vip
    - 高可用
---

# Kubernetes 高可用部署（kube-vip）

# 节点规划

由于使用内部 etcd 集群（etcd 部署在 master 节点上），故需要奇数台 master 机器

| role | hostname | ip |
| --- | --- | --- |
| vip |  | 192.168.181.235 |
| master1 | longzhou-vt-ctl3.lz.dscc.99.com | 192.168.181.85 |
| master2 | longzhou-vt-ctl4.lz.dscc.99.com | 192.168.181.84 |
| master3 | longzhou-vt-insight1.lz.dscc.99.com | 192.168.181.66 |
| node1 | longzhou-lkv1.lz.dscc.99.com | 172.24.129.23 |
| node2 | longzhou-lkv2.lz.dscc.99.com | 172.24.129.24 |
| node3 | longzhou-lkv3.lz.dscc.99.com | 172.24.129.25 |

# 删除旧节点

在所有曾经安装过 kubernetes 的机器上，执行下列操作，删除旧节点信息

```bash
# 清理网卡信息
sudo ifconfig cni0 down
sudo ip link del cni0
sudo ifconfig flannel.1 down
sudo ip link del flannel.1
ifconfig # 查看网卡信息，是否已删除 cni0 与 flannel.1
ip link # 查看网卡信息，是否已删除 cni0 与 flannel.1
sudo rm -rf /etc/cni/net.d/*flannel.conflist
sudo rm -rf /run/flannel/subnet.env

# 清理用户目录
sudo rm -rf ~/.kube

# 清理 iptables，删除前注意保留非 kubernetes 的配置
sudo iptables -F && sudo iptables -X && sudo iptables -F -t nat && sudo iptables -X -t nat

# 重启 kubelet 与 docker 服务
sudo systemctl restart kubelet
sudo systemctl restart docker
```

# 安装前检查

## 检查 mac 地址唯一性

```bash
ifconfig -a
```

## 检查 product_uuid 唯一性

```bash
sudo cat /sys/class/dmi/id/product_uuid
```

## 允许 iptables 检查桥接流量

```bash
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

## 关闭 swap

```bash
# 临时关闭
sudo swapoff -a 
# 永久关闭需要删除 /etc/fstab 中 swap 那行
```

# 安装 docker

```bash
sudo apt-get update
sudo apt-get -y install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get -y update
sudo apt-get -y install docker-ce
```

修改 docker 的配置，修改 cgroupdriver 为 systemd：

```
{  
  "registry-mirrors": [
    "https://ebcsc6ia.mirror.aliyuncs.com",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "max-concurrent-uploads":1,
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "dns": ["192.168.9.35", "8.8.8.8"]  
}
```

增加 docker http/https 代理支持：

```
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/proxy.conf
[Service]
Environment="HTTP_PROXY=http://longzhou-hdpnn.lz.dscc.99.com:7890"
Environment="HTTPS_PROXY=http://longzhou-hdpnn.lz.dscc.99.com:7890"
Environment="NO_PROXY=localhost,127.0.0.1,.aliyun.com,.cn"
EOF
```

重启 docker 服务

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

# **安装 kubeadm、kubelet 和 kubectl**

1. 安装依赖
    
    ```bash
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl
    ```
    
2. 修改 `/etc/apt/sources.list.d/kubernetes.list` ，增加以下内容：
    
    ```bash
    deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
    ```
    
3. `apt-get update` 
    
    在更新时会出现缺少相应的 key 的报错，执行以下命令添加对应的 key，这里 `E084DAB9` 就是 key 的后八位
    
    ```bash
    gpg --keyserver keyserver.ubuntu.com --recv-keys E084DAB9
    gpg --export --armor E084DAB9 | sudo apt-key add -
    ```
    
4. 查看目前kubeadm、kubelet、kubectl 的版本：
    
    ```bash
    sudo apt-cache madison kubeadm
    sudo apt-cache madison kubelet
    sudo apt-cache madison kubectl
    ```
    
5. 锁定版本，防止 apt upgrade 升级版本：
    
    ```bash
    sudo apt-mark hold kubelet kubeadm kubectl
    ```
    
6. 安装默认版本，或指定版本：
    
    ```bash
    sudo apt-get install -y kubectl kubelet kubeadm
    sudo apt install -y kubelet=1.22.5-00 kubeadm=1.22.5-00 kubectl=1.22.5-00
    ```
    

# 预先拉取镜像

在 master 节点上拉取镜像。

```bash
kubeadm config images pull --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
```

# 安装 kube-vip 静态 pod

使用 ifconfig 命令确认网卡名为 `eth0` 。

kubeadm init 命令执行之前，在第一个 control-plane 上执行：

```bash
export VIP=192.168.181.235
export INTERFACE=eth0
KVVERSION=$(curl -sL https://api.github.com/repos/kube-vip/kube-vip/releases | jq -r ".[0].name")

alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:$KVVERSION"

kube-vip manifest pod \
    --interface $INTERFACE \
    --address $VIP \
    --controlplane \
    --services \
    --arp \
    --leaderElection | sudo tee /etc/kubernetes/manifests/kube-vip.yaml
```

执行后会在 `/etc/kubernetes/manifests` 目录下生成 kube-vip.yaml 配置文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-vip
  namespace: kube-system
spec:
  containers:
  - args:
    - manager
    env:
    - name: vip_arp
      value: "true"
    - name: port
      value: "6443"
    - name: vip_interface
      value: eth0
    - name: vip_cidr
      value: "32"
    - name: cp_enable
      value: "true"
    - name: cp_namespace
      value: kube-system
    - name: vip_ddns
      value: "false"
    - name: svc_enable
      value: "true"
    - name: vip_leaderelection
      value: "true"
    - name: vip_leaseduration
      value: "5"
    - name: vip_renewdeadline
      value: "3"
    - name: vip_retryperiod
      value: "1"
    - name: address
      value: 192.168.181.235
    - name : lb_enable
      value: "true"
    - name: lb_port
      value: "6443"
    image: ghcr.io/kube-vip/kube-vip:v0.4.1
    imagePullPolicy: Always
    name: kube-vip
    resources: {}
    securityContext:
      capabilities:
        add:
        - NET_ADMIN
        - NET_RAW
    volumeMounts:
    - mountPath: /etc/kubernetes/admin.conf
      name: kubeconfig
  hostAliases:
  - hostnames:
    - kubernetes
    ip: 127.0.0.1
  hostNetwork: true
  volumes:
  - hostPath:
      path: /etc/kubernetes/admin.conf
    name: kubeconfig
status: {}
```

注意，其中以下四行自动生成的配置里没有：

```yaml
    - name : lb_enable
      value: "true"
    - name: lb_port
      value: "6443"
```

是为了打开 loadbalancer 而手动加入的配置。

# 初始化第一个 control-plane

在 longzhou-vt-ctl3 上执行：

```yaml
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers --upload-certs --control-plane-endpoint 192.168.181.235
```

执行成功输出如下 ：
![](https://tva1.sinaimg.cn/large/008i3skNly1gy7ebh6xilj31730u0n1t.jpg)
根据提示，保存以下两行命令，分别用来加入新的 control-plane 节点与 node 节点：

```yaml
# 记录 kubeadm init 的输出
# control-plane 节点，**仅记录，不执行**
sudo kubeadm join 192.168.181.235:6443 --token os4paw.ph73f4bcatp3vi2g \
	--discovery-token-ca-cert-hash sha256:63d4a4c4447091ed636295b27b007b92b740de166936831f9192bde3d0ec5947 \
	--control-plane --certificate-key 0c46b1855ec22687adfabedfb816756816f0dd45bf88dae03dbaaa5537007709

# node 节点，**仅记录，不执行**
sudo kubeadm join 192.168.181.235:6443 --token os4paw.ph73f4bcatp3vi2g \
	--discovery-token-ca-cert-hash sha256:63d4a4c4447091ed636295b27b007b92b740de166936831f9192bde3d0ec5947
```

根据提示，继续执行：

```bash
# 待 kubeadm init 执行成功后
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

此时第一个 control-plane 已经初始化完毕。

# 加入其余的 control-plane

1. 在其余 control-plane 节点上， `telnet 192.168.181.235 6443` ，确保可以连接vip。
2. 将第一个节点的 kube-vip.yaml 复制到其余的 control-plane 节点的 `/etc/kubernetes/manifests` 目录下。
3. 在其余每个 control-plane 节点上分别执行：
    
    ```bash
    sudo kubeadm join 192.168.181.235:6443 --token os4paw.ph73f4bcatp3vi2g \
    	--discovery-token-ca-cert-hash sha256:63d4a4c4447091ed636295b27b007b92b740de166936831f9192bde3d0ec5947 \
    	--control-plane --certificate-key 0c46b1855ec22687adfabedfb816756816f0dd45bf88dae03dbaaa5537007709
    ```
    
4. 加入成功后，继续执行：
    
    ```bash
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
    

# 加入 node 节点

1. 检查各个 node 节点是否可以 `telnet 192.168.181.235 6443`
2. 执行 sudo kubeadm join
    
    ```bash
    sudo kubeadm join 192.168.181.235:6443 --token os4paw.ph73f4bcatp3vi2g \
    	--discovery-token-ca-cert-hash sha256:63d4a4c4447091ed636295b27b007b92b740de166936831f9192bde3d0ec5947
    ```
    

# 安装 cni 网络插件（flannel）

在任意一个 master 节点上执行：

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

> 这一步下载的镜像需要科学上网
> 

# 安装 kubernets-dashboard

1. 在任意一个 master 节点上执行：
    
    ```bash
    kubectl apply -f [https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml](https://raw.githubusercontent.com/kubernetes/dashboard/v2.4.0/aio/deploy/recommended.yaml) （注意是否有版本更新）
    ```
    
2. 使用如下命令查看 kubernets-dashboard 状态
    
    ```bash
    kubectl get deployment --namespace=kubernetes-dashboard kubernetes-dashboard
    
    kubectl describe deployment --namespace=kubernetes-dashboard kubernetes-dashboard
    
    kubectl get service --namespace=kubernetes-dashboard kubernetes-dashboard
    
    kubectl --namespace=kubernetes-dashboard get pod -o wide
    ```
    
3. 新建 dash-admin-user.yaml 文件，内容如下：
    
    ```yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: admin-user
      namespace: kubernetes-dashboard
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: admin-user
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: cluster-admin
    subjects:
    - kind: ServiceAccount
      name: admin-user
      namespace: kubernetes-dashboard
    ```
    
    执行命令 `kubectl apply -f dash-admin-user.yaml` 应用。
    
4. 创建登录令牌 `kubectl -n kubernetes-dashboard get secret $(kubectl -n kubernetes-dashboard get sa/admin-user -o jsonpath="{.secrets[0].name}") -o go-template="{{.data.token | base64decode}}"` 
    
    得到类似如下输出：
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7ed8vggyj32hk08u0xo.jpg)
    
    这里的 token，最后一位的 `%` 不要复制。
    
    或使用命令`kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')` 得到输出如下：
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7eewelpzj32hq0ia7aa.jpg)
    
5. 在 master 上执行 `kubectl proxy`命令，在 [localhost:8001](http://localhost:8001) 上启动 proxy。
6. 由于 kubernets-dashboard 只允许 [`http://localhost`](http://localhost) ， `https://` 开头的访问方式，我们需要在跳板机（自己的工作机，可以通过 ssh 登录 master）上利用 ssh 隧道来访问。
    
    在跳板机上上执行 `ssh -L localhost:8001:localhost:8001 -NT [linqili@longzhou-vt-ctl3.lz.dscc.99.com](mailto:linqili@longzhou-vt-ctl3.lz.dscc.99.com)` 
    
    然后在跳板机的浏览器上访问： [`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/) 
    
7. 在弹出的窗口上输入令牌，选择登录。
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7efgpyy9j30nj0b6wfh.jpg)
    

# 安装 kubernetes-metrics-server

1. 在任意一个 master 上执行`kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`
2. 默认情况下，metrics-server 是无法正常运行的，检查 pod 运行情况， `kubectl logs -f metrics-server-6c49cf6978-qdzwl -n kube-system` ，发现一下日志：
    
    ```prolog
    ...
    E0107 08:07:19.436094       1 scraper.go:139] "Failed to scrape node" err="Get \"https://172.24.129.25:10250/stats/summary?only_cpu_and_memory=true\": x509: cannot validate certificate for 172.24.129.25 because it doesn't contain any IP SANs" node="longzhou-lkv3.lz.dscc.99.com"
    E0107 08:07:19.449785       1 scraper.go:139] "Failed to scrape node" err="Get \"https://172.24.129.24:10250/stats/summary?only_cpu_and_memory=true\": x509: cannot validate certificate for 172.24.129.24 because it doesn't contain any IP SANs" node="longzhou-lkv2.lz.dscc.99.com"
    E0107 08:07:19.450141       1 scraper.go:139] "Failed to scrape node" err="Get \"https://192.168.181.84:10250/stats/summary?only_cpu_and_memory=true\": x509: cannot validate certificate for 192.168.181.84 because it doesn't contain any IP SANs" node="longzhou-vt-ctl4.lz.dscc.99.com"
    ...
    ```
    
3. 由于默认安装的 kubernetes 集群采用的是自签名证书，这些 ip 不属于证书 SAN 部分，因而无法与 apiserver 建立安全的 TLS 连接。解决办法是使用 apiserver 的证书来对 kubelet 节点进行签名。
4. 在 kubernetes dashboard 上编辑名为 `kubelet-config-1.23` 的 ConfigMap（namespace: kube-system），在 `kubelet` 键下，增加 `serverTLSBootstrap: true` 字段。
    
    ```yaml
    {
    	"kubelet": "apiVersion: kubelet.config.k8s.io/v1beta1
    		authentication:
    		  anonymous:
    		    enabled: false
    		  webhook:
    		    cacheTTL: 0s
    		    enabled: true
    		  x509:
    		    clientCAFile: /etc/kubernetes/pki/ca.crt
    		authorization:
    		  mode: Webhook
    		  webhook:
    		    cacheAuthorizedTTL: 0s
    		    cacheUnauthorizedTTL: 0s
    		cgroupDriver: systemd
    		clusterDNS:
    		- 10.96.0.10
    		clusterDomain: cluster.local
    		cpuManagerReconcilePeriod: 0s
    		evictionPressureTransitionPeriod: 0s
    		fileCheckFrequency: 0s
    		healthzBindAddress: 127.0.0.1
    		healthzPort: 10248
    		httpCheckFrequency: 0s
    		imageMinimumGCAge: 0s
    		kind: KubeletConfiguration
    		logging:
    		  flushFrequency: 0
    		  options:
    		    json:
    		      infoBufferSize: \"0\"
    		  verbosity: 0
    		memorySwap: {}
    		nodeStatusReportFrequency: 0s
    		nodeStatusUpdateFrequency: 0s
    		resolvConf: /run/systemd/resolve/resolv.conf
    		**rotateCertificates: true**
    		runtimeRequestTimeout: 0s
    		**serverTLSBootstrap: true**
    		shutdownGracePeriod: 0s
    		shutdownGracePeriodCriticalPods: 0s
    		staticPodPath: /etc/kubernetes/manifests
    		streamingConnectionIdleTimeout: 0s
    		syncFrequency: 0s
    		volumeStatsAggPeriod: 0s
    		"
    }
    ```
    
5. 在每一个节点上（包括 master 与 nodes）上，编辑 `/var/lib/kubelet/config.yaml` ，添加 `serverTLSBootstrap: true` 。并 `systemctl restart kubelet`重启。
    
    ```yaml
    apiVersion: kubelet.config.k8s.io/v1beta1
    authentication:
      anonymous:
        enabled: false
      webhook:
        cacheTTL: 0s
        enabled: true
      x509:
        clientCAFile: /etc/kubernetes/pki/ca.crt
    authorization:
      mode: Webhook
      webhook:
        cacheAuthorizedTTL: 0s
        cacheUnauthorizedTTL: 0s
    cgroupDriver: systemd
    clusterDNS:
    - 10.96.0.10
    clusterDomain: cluster.local
    cpuManagerReconcilePeriod: 0s
    evictionPressureTransitionPeriod: 0s
    fileCheckFrequency: 0s
    healthzBindAddress: 127.0.0.1
    healthzPort: 10248
    httpCheckFrequency: 0s
    imageMinimumGCAge: 0s
    kind: KubeletConfiguration
    logging:
      flushFrequency: 0
      options:
        json:
          infoBufferSize: "0"
      verbosity: 0
    memorySwap: {}
    nodeStatusReportFrequency: 0s
    nodeStatusUpdateFrequency: 0s
    resolvConf: /run/systemd/resolve/resolv.conf
    **rotateCertificates: true**
    runtimeRequestTimeout: 0s
    **serverTLSBootstrap: true**
    shutdownGracePeriod: 0s
    shutdownGracePeriodCriticalPods: 0s
    staticPodPath: /etc/kubernetes/manifests
    streamingConnectionIdleTimeout: 0s
    syncFrequency: 0s
    volumeStatsAggPeriod: 0s
    ```
    
6. 在以上涉及的配置文件中，还需确认 `rotateCertificates: true` 。此外，在 kubelet 启动的时候会默认添加参数 `--rotate-server-certificates true` ，来保证证书到期后自动轮换。详情查看[证书轮换](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/#certificate-rotation)。
7. 使用 `kubectl get csr` 命令查看当前集群中的证书：
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7eg03zx7j30ti04tq3t.jpg)
    
    使用 `kubectl certificate approve <csr-name>` 命令手动批准证书。
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7egjlzupj30vi0bqtay.jpg)
    
    可以看到，批准后，证书的 CONDITION 从 `Pending` 变为了 `Approved,Issued` 。更多关于证书的内容可以查看[这里](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/#certificate-rotation)。
    
    至此，kubernetes-metric-server 可以正常工作。使用 `kubectl top nodes` 命令查看节点资源信息。
    
    ![](https://tva1.sinaimg.cn/large/008i3skNly1gy7egzjij9j30js0563yw.jpg)
    

# 部署示例

1. 部署一个 deployment 和 service，程序清单如下 hello.yaml：
    
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      labels:
        app.kubernetes.io/name: load-balancer-example
      name: hello-world
    spec:
      replicas: 2
      selector:
        matchLabels:
          app.kubernetes.io/name: load-balancer-example
      template:
        metadata:
          labels:
            app.kubernetes.io/name: load-balancer-example
        spec:
          containers:
          - image: registry.cn-hangzhou.aliyuncs.com/aliyun_google/google-sample-node-hello:1.0
            name: hello-world
            ports:
            - containerPort: 8080
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: hello-world
    spec:
      type: NodePort
      ports:
      - port: 8080
        nodePort: 31704
        targetPort: 8080
      selector:
        app.kubernetes.io/name: load-balancer-example
    ```
    
2. 使用 kubectl apply -f hello.yaml 执行
3. 执行之后，在每一个 master 节点上都应更会启动一个端口的 Listening 监听端口。
4. `curl http://192.168.181.235:31704`，此时屏幕上返回 hello kubernetes!，则程序正常执行。