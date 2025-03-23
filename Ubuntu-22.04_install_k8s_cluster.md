集群配置要求：
一主两从模式，使用kubeadm方式安装
k8s版本：v1.28.2
    CRI:containerd.io
    CNI:CoreOS Flannel

网络环境：
    master:   11.0.1.21/24
    worker-1: 11.0.1.22/24
    worker-2: 11.0.1.23/24
    gateway:  11.0.1.2/24
    DNS1:8.8.8.8
    DNS2:223.5.5.5
    Pod-cidr: 10.244.0.0/12
    Service-cidr: 10.96.0.0/12


一、ubuntu安装配置
1.1 基础配置 
1.1.1 设置root密码并开启免密切换root
 

关闭apt自动更新
sudo systemctl stop unattended-upgrades
sudo systemctl disable unattended-upgrades

开启IPV4转发
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sysctl --system
sudo modprobe br_netfilter
echo '1' | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
sudo nano /etc/sysctl.conf
在文件的末尾添加以下两行配置
net.bridge.bridge-nf-call-iptables = 1 
net.ipv4.ip_forward = 1
sudo sysctl -p
sudo nano /etc/modules-load.d/br_netfilter.conf
在文件中添加以下内容：
br_netfilter


1.1.2 配置主机名解析
所有节点：
cat >> /etc/hosts << EOF
11.0.1.21 master kubeapi.hebin.com k8sapi.hebin.com kubeapi 
11.0.1.22 worker-1 k8s-worker-1.hebin.com
11.0.1.23 worker-2 k8s-worker-2.hebin.com
EOF

1.1.3 配置网卡
master:
#nano /etc/netplan/01-network-manager-all.yaml
# Let NetworkManager manage all devices on this system
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 11.0.1.21/24
      gateway4: 11.0.1.2
      nameservers:
        addresses:
          - 8.8.8.8
          - 223.5.5.5
# netplan apply 

worker节点修改好对应的IP即可

1.2 关闭防火墙和swap(所有节点操作)
#systemctl stop ufw && systemctl disable ufw --now 
#swapoff -a  
root@master:~# cp /etc/fstab /etc/fstab.bak
root@master:~# sed -n '/swap/s/^/#/gp' /etc/fstab
#/swapfile                                 none            swap    sw              0       0
root@master:~# sed -i '/swap/s/^/#/' /etc/fstab

1.3 配置sources源，这里使用阿里云源
直接使用阿里云官方文档即可

# apt upgrade
# apt update 

1.4 安装必要服务
1.4.1 配置必要的目录，所有节点
root@master:~# mkdir -pv /opt/k8s_cluster_install/{yamls,scripts,packages}


1.4.2 master节点搭建samba服务器
安装samba服务
# apt install samba -y

配置Samba
#vim /etc/samba/smb.conf
#在文件末尾添加以下内容来创建一个共享目录：
[shared]
   comment = Shared Folder
   path = /src/to/shared
   browseable = yes
   writable = yes
   read only = no
   create mask = 0777
   directory mask = 0777
   public = yes



设置samba用户
# smbpasswd -a username

重启服务
systemctl restart smbd && systemctl enable smbd



1.5 安装配置containerd.io
1.5.1 安装containerd.io 
#apt install -y containerd.io=1.6.26-1
#apt-mark hold containerd.io    #防止意外升级
#修改配置文件 
# containerd config default | sudo tee /etc/containerd/config.toml
# vim /etc/containerd/config.toml
添加配置：
  1. 修改containerd使用SystemdCgroup
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
          SystemdCgroup = true
  2. 配置Containerd使用国内Mirror站点上的pause镜像及指定的版本
      [plugins."io.containerd.grpc.v1.cri"]
        sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"

  3. 配置镜像地址
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.1ms.run"]
          endpoint = ["https://docker.1ms.run"]
        
        [plugins."io.containerd.grpc.v1.cri".registry.configs."docker.1ms.run".tls]
          insecure_skip_verify = true
      
        [plugins."io.containerd.grpc.v1.cri".registry.configs."docker.1ms.run".auth]
          username = "19951769865"
          password = "hebin199075"

        [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.k8s.io"]
          endpoint = ["https://registry.aliyuncs.com/google_containers"]

1.5.2 安装crictl工具
由于containerd.io使用的ctr作为CLI命令，用于直接和containerd交互，crictl作为kubernetes社区提供的工具，
可很好的通过CRI接口与容器运行时交互，所以要单独安装该工具
VERSION="v1.28.0"
wget https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-$VERSION-linux-amd64.tar.gz

sudo tar -zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin

crictl --version

sudo mkdir -p /etc/crictl
sudo tee /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

#检查是否正确连接到containerd
crictl info


2.0  配置节点免密
./free_login.sh

2.2 安装chrony服务
# apt install -y chrony

修改配置文件，添加阿里云的同步地址
master:
#vim /etc/chrony/chrony.conf
#注释默认NTP地址，添加阿里云同步地址
server ntp.aliyun.com iburst
server ntp1.aliyun.com iburst
server ntp2.aliyun.com iburst
#允许worker节点同步自己
allow 11.0.1.0/24

#systemctl daemon-reload && systemctl restart chrony

检查同步时间是否正常
root@master:~# chronyc sources -v


worker节点： 
#vim /etc/chrony/chrony.conf
#注释默认NTP地址，只同步master地址时间
server 11.0.1.21 iburst

检查同步时间是否正常
root@worker-2:~# systemctl daemon-reload 
root@worker-2:~# systemctl restart chrony
root@worker-2:~# chronyc sources -v

3.0  安装kubeadm,kubelet,kubectl工具
配置阿里云源 
apt-get update && apt-get install -y apt-transport-https
curl -fsSL https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/Release.key |
    gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://mirrors.aliyun.com/kubernetes-new/core/stable/v1.28/deb/ /" |
    tee /etc/apt/sources.list.d/kubernetes.list
apt-get update

#查看可安装版本
apt-cache madison kubeadm kubelet kubectl
 kubeadm | 1.28.2-1.1   kubelet | 1.28.2-1.1    kubectl | 1.28.2-1.1
#安装指定版本
sudo apt install -y kubeadm=1.28.2-1.1 kubelet=1.28.2-1.1 kubectl=1.28.2-1.1

#锁定版本
sudo apt-mark hold kubelet kubeadm kubectl

#systemctl enable kubelet 


3.1 集群安装 
3.1.1 查看并预拉所需镜像，仅在master节点操作
查看待拉取镜像信息：
root@master:/opt/k8s_cluster_install/packages# kubeadm config images list --image-repository=registry.aliyuncs.com/google_containers


3.1.2 拉取镜像：
root@master:~# kubeadm config images pull  --image-repository=registry.aliyuncs.com/google_containers


3.2初始化集群
sudo kubeadm init \
    --control-plane-endpoint="11.0.1.21" \
    --kubernetes-version=v1.28.2 \
    --pod-network-cidr=10.244.0.0/16 \
    --service-cidr=10.96.0.0/12 \
    --token-ttl=0 \
    --upload-certs \
    --image-repository registry.aliyuncs.com/google_containers    


Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 11.0.1.21:6443 --token 7lq3ad.ouohi0lyjf4dj034 \
        --discovery-token-ca-cert-hash sha256:c573d208f477f4571962f5b3dcb4bbcb568120445f55f7508c4bceb3daa78400 \
        --control-plane --certificate-key aba0c919137e94fd6afc2b32c1264e483817c2ebcdab2826569e63aa4e44ec8b

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 11.0.1.21:6443 --token 7lq3ad.ouohi0lyjf4dj034 \
        --discovery-token-ca-cert-hash sha256:c573d208f477f4571962f5b3dcb4bbcb568120445f55f7508c4bceb3daa78400 

3.3 加入集群
master节点执行：
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

export KUBECONFIG=/etc/kubernetes/admin.conf


worker节点执行：
kubeadm join 11.0.1.21:6443 --token 7lq3ad.ouohi0lyjf4dj034 \
        --discovery-token-ca-cert-hash sha256:c573d208f477f4571962f5b3dcb4bbcb568120445f55f7508c4bceb3daa78400 

3.4 安装CNI
3.4.1 安装网络插件flannel:
wget https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

3.4.2 将镜像地址替换为国内镜像地址：
image: ghcr.io/flannel-io/flannel:v0.26.5
image: ghcr.io/flannel-io/flannel-cni-plugin:v1.6.2-flannel1
image: ghcr.io/flannel-io/flannel:v0.26.5

image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/flannel-io/flannel:v0.26.5
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/flannel-io/flannel-cni-plugin:v1.6.2-flannel1
image: swr.cn-north-4.myhuaweicloud.com/ddn-k8s/ghcr.io/flannel-io/flannel:v0.26.5

3.4.3 部署CNI插件
kubectl create -f kube-flannel.yaml 


4.0 设置命令补全
apt install bash-completion -y 
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc



5.0 配置git

apt install -y git 

设置用户名，邮箱
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"

配置代理：（只配置ubuntu的代理是无法git clone的）
git config --global http.proxy http://your-proxy-server:port
git config --global https.proxy http://your-proxy-server:port

Git clone所需项目：
# git clone https://github.com/iKubernetes/learning-k8s.git
# cd /opt/k8s_cluster_install/git_clone_repostries/learning-k8s

git后使用现有资源搭建一套wordpress+mysql站点环境，所需yaml文件在
/opt/k8s_cluster_install/git_clone_repostries/learning-k8s/wordpress目录下，
直接执行kubectl -f mysql-ephemeral/  和 kubectl -f wordpress-apache-ephemeral/
注意要修改两个deployment里的image镜像地址和pull策略，然后预拉镜像到本地。

PS：ctr pull 镜像可以直接看到进度，但是默认拉取到的镜像是在default命名空间，而crictl 使用的是k8s.io命名空间
下的镜像，因此在拉取镜像时需注意。
可以使用ctr -n k8s.io 指定命名空间的方式直接将镜像拉取到指定k8s.io下

导出/导入镜像：
ctr images export mysql-8.0.tar docker.1ms.run/mysql:8.0

ctr -n k8s.io images import mysql-8.0.tar
