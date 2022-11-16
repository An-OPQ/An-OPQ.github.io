---
title: Docker 与 K8S 离线安装（Kubeadm方式）
date: 2022-09-12 00:05:18
tags: 运维
cover: /img/wallhaven-z8l37o.jpg


---

# Docker 与 K8S 离线安装（Kubeadm方式）

<aside>
💡 本次安装的版本是：Docker-20.10   K8S-1.23.8   Centos7.9
</aside>



由于工作中许多环境都是离线的服务器，所以自己在学习过程中用于记录。

软件包地址：阿里云盘

「K8S-1.23.8安装包与镜像」[https://www.aliyundrive.com/s/ABZCZTsDSyT](https://www.aliyundrive.com/s/ABZCZTsDSyT)
点击链接保存，或者复制本段内容，打开「阿里云盘」APP ，无需下载极速在线查看，视频原画倍速播放。

- rpmPackage: K8S kubeadm、kubelet、kubectl 的安装包
- dockerImages: K8S组件的docker镜像
- calico：网络插件的docker镜像

建议可以尝试自己下载软件包与镜像，这样成功感会更大，同时挑战也就更大。

致谢：

本文大部分的笔记都源自于学习该位UP主的视频，大伙儿可以点点关注啥的（非笔者）。

[Linux-k8s开发的个人空间_哔哩哔哩_Bilibili](https://space.bilibili.com/510305915/?spm_id_from=333.999.0.0)

### 环境准备

- 服务器准备

笔者前置环境是：使用vmWare初始化了两台Centos7.9的服务器，配置如下：

内存：4G
CPU：2核

配置静态IP：

```bash
# 查看网卡使用的是那个配置文件
ls /etc/sysconfig/network-scripts/ifcfg-*
ip addr
# 修改配置文件
BOOTPROTO="dhcp" 改为：static
ONBOOT="no" 改为：yes
# 追加配置
IPADDR=192.168.101.251
NETMASK=255.255.255.0
GATEWAY=192.168.101.1

```

重启网络:

```bash
service network restart
```

静态IP配置成功之后，DNS配置一般都会消失，所以这时候ping域名就没法ping通。需要配置DNS

```bash
vim /etc/resolv.conf
# 添加配置文件
nameserver 114.114.114.114
nameserver 8.8.8.8
```

保存后，DNS配置是立即生效的，只要本地需要解析缓存区没有的域名，就需要读取一般DNS配置，所以这个配置是立即生效的。

- K8S安装前初始化

```bash
# 1.设置主机名
hostnamectl set-hostname master
hostnamectl set-hostname node1

# 2.添加hosts解析
cat >> /etc/hosts << EOF
192.168.101.251 master
192.168.101.252 node1
EOF

# 3.同步时间
yum -y install ntp
systemctl enable ntpd --now

# 4.永久关闭seLinux（需要重启系统生效）
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 5.永久关闭swap（需要重启系统生效）
swapoff -a  # 临时关闭
sed -i 's/.*swap.*/#&/g' /etc/fstab # 永久关闭

# 6.升级内核为5.4版本（需要重启系统生效）
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-5.el7.elrepo.noarch.rpm
yum --enablerepo=elrepo-kernel install -y kernel-lt
grub2-set-default 0
reboot #这里先重启再继续

# 7.关闭防火墙、清空iptables规则
systemctl disable firewalld && systemctl stop firewalld
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X && iptables -P FORWARD ACCEPT && service iptables save

# 8.加载IPVS
yum -y install ipset ipvsadm

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

modprobe -- nf_conntrack
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack

# 9.开启br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

### 安装Docker离线

[Index of linux/static/stable/x86_64/](https://download.docker.com/linux/static/stable/x86_64/)

1. 首先，访问上述地址，通过能访问外网的电脑下载软件包。通过scp 或则 ftp 上传到服务器上

```bash
#示例：scp
# 默认22 端口
scp <docker file >  <remoteUser>@<remoteIP>:<remotePath>
# 指定ssh端口（一般都是默认22端口，若您修改了ssh端口则需要指定，下面示例：10011端口）
# 默认22 端口
scp docker-20.10.0.tgz root@192.168.101.252:/home/admin
# -P(大写)
scp -P 10011  docker-20.10.0.tgz root@192.168.101.252:/home/admin 
```

2. 解压

```bash
tar zxvf docker-20.10.0.tgz
```

3. 将解压出来的Docker文件复制到/usr/bin目录下

```bash
sudo cp docker/* /usr/bin/
```

4. 创建docker.service文件

```bash
cat > /usr/lib/systemd/system/docker.service << 'EOF'
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

6. 启动docker

   ps：注意一定要先启动docker，再添加daemon.json文件。

```bash
systemctl enable docker --now
```

7. 添加daemon.json文件

```bash
cat > /etc/docker/daemon.json <<EOF
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
  "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
    ]
}
EOF
```

8. 重启docker

```bash
systemctl daemon-reload && systemctl restart docker
```

aliyun镜像地址：由于阿里云的docker镜像都是个人的，所以自己登陆aliyun复制下来即可

管控台：搜索→容器镜像服务

![2](image-20220912002246090.png)

### 安装k8s（kubeadm-1.23.8、kubelet-1.23.8、kubectl-1.23.8）

1. 准备安装包

由于是离线的服务器，所以需要提前准备安装包。

使用一台能访问外网的Centos服务器下载软件包，或则使用作者提供的软件包

（在每台服务器都需要 安装：kubeadm、kubelet、kubectl）

```bash
yum -y install --downloadonly --downloaddir=$PWD kubeadm-1.23.8-0 kubelet-1.23.8-0 kubectl-1.23.8-0
```

scp 或 ftp上传软件包

2. 添加源

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=kubernetes
baseurl=https://mirrors.tuna.tsinghua.edu.cn/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF
```

3. 安装

```bash
# 在软件包的目录下
rpm -ivh *.rpm --nodeps --force
# 查看是否安装成功
yum list installed |grep kube
# 启用
systemctl enable --now kubelet
```

4. 初始化集群

- 准备镜像

需要一台安装了kubeadm的服务器来拉镜像，或者使用作者提供的。

```bash
mkdir ~/kubeadm_init && cd ~/kubeadm_init

kubeadm config print init-defaults > kubeadm-init.yaml
```

```bash
cat > kubeadm-init.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.1.201 #修改自己的ip
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master
  taints:
  - effect: "NoSchedule"
    key: "node-role.kubernetes.io/master"
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.23.8
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16  
scheduler: {}
--- 
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration 
mode: ipvs 
--- 
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration 
cgroupDriver: systemd
EOF
```

```bash
# 预拉取镜像
kubeadm config images pull --config kubeadm-init.yaml
```

镜像拉取完成之后能看到下面七个镜像，那么本地存在镜像了，就需要打包出来上传到离线的服务器上。

```bash
registry.aliyuncs.com/google_containers/kube-apiserver            v1.23.8                  09d62ad3189b   2 months ago    135MB
registry.aliyuncs.com/google_containers/kube-controller-manager   v1.23.8                  2b7c5a039984   2 months ago    125MB
registry.aliyuncs.com/google_containers/kube-scheduler            v1.23.8                  afd180ec7435   2 months ago    53.5MB
registry.aliyuncs.com/google_containers/kube-proxy                v1.23.8                  db4da8720bcb   2 months ago    112MB
registry.aliyuncs.com/google_containers/etcd                      3.5.1-0                  25f8c7f3da61   10 months ago   293MB
registry.aliyuncs.com/google_containers/coredns                   v1.8.6                   a4ca41631cc7   11 months ago   46.8MB
registry.aliyuncs.com/google_containers/pause                     3.6                      6270bb605e12   12 months ago   683kB
```

打包镜像

共需要打包七份

```bash
# 示例 注意:打包的tar包，前面不能带镜像仓库和命令空间，会被误以为是目录，例如：
# 错误：registry.aliyuncs.com/google_containers/kube-apiserver.tar
# 正确：kube-apiserver.tar
docker save -o kube-apiserver.tar registry.aliyuncs.com/google_containers/kube-apiserver:v1.23.8
```

导入镜像

通过scp或则ftp上传到服务器之后，需要导入到docker中

```bash
# 示例：
docker load < etcd.tar
```

![3](image-20220912002224616.png)

- 初始化

将上述七个镜像都导入到离线服务器上之后，再离线安装

```bash
kubeadm init --config=kubeadm-init.yaml | tee kubeadm-init.log

# 若其他节点也需要kubectl的支持，就需要将配置文件复制过去
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

5. 安装k8s集群网络

还是需要提前下载yaml文件与镜像

`yaml：`

```bash
curl https://docs.projectcalico.org/manifests/calico.yaml -O

1 修改CIDR
- name: CALICO_IPV4POOL_CIDR
  value: "10.244.0.0/16"
2 指定网卡
# Cluster type to identify the deployment type
  - name: CLUSTER_TYPE
  value: "k8s,bgp"
# 下面添加
  - name: IP_AUTODETECTION_METHOD
    value: "interface=ens33"
    # ens33为本地网卡名字

#不指定网卡
#创建pod时会有如下报错
#创建pod时报Failed create pod sandbox

```

`镜像：`

```bash
calico/kube-controllers                                           v3.24.0   d4d0783ac017   3 weeks ago     71.3MB
calico/cni                                                        v3.24.0   45f84749206f   3 weeks ago     197MB
calico/node                                                       v3.24.0   c595bc026ccf   3 weeks ago     220MB

# 示例：
docker save -o kube-controllers.tar calico/kube-controllers:v3.24.0
docker load < kube-controllers.tar
```

`apply:`

将上述三个镜像都导入到离线服务上之后，再执行

```bash
# apply
kubectl apply -f calico.yaml
```

6. node节点加入k8s集群

(node节点不需要初始化，安装好下面三个组件直接join)

node节点也需要安装kubeadm、kubelet、kubectl

kubeadm join 可以直接在步骤4-初始化执行完的终端上看到提示，也可以看日志文件<kubeadm-init.log>

```bash
vim kubeadm-init.log
# 按下 G  ps:在vim快捷键中，大写 G 是跳转到vim的底部，双小写 gg 是到vim的顶部
#就可以看到 一堆kubeadm join ..................复制到node节点上执行即可。
```

```bash
kubeadm join ................
```

PS：kubeadm join 卡住不动或则失败的原因可能是：

- 服务器的时间不同步
- kubeadm 令牌过期
  - kubeadm的令牌仅仅只有一天。若过期则需要重新生成令牌
