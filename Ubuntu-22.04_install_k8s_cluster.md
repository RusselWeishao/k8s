集群配置要求：
一主两从模式，使用kubeadm方式安装。
k8s版本：v1.28.2
    CRI:containerd.io
    CNI:CoreOS Flannel
Docker版本：Docker 24.0

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
    $sudo passwd root   ##密码123456
    $sudo visudo
    文末添加：
    hebin ALL=(ALL) NOPASSWD: ALL

    测试免密：
    sudo -i 

    允许root身份ssh登录
    #vim /etc/ssh/sshd_config
    找到 #PermitRootLogin prohibit-password 或 PermitRootLogin no，将其改为：
    PermitRootLogin yes

    sudo systemctl restart ssh

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
#cp /etc/apt/sources.list /etc/apt/sources.list.bak 
#echo > /etc/apt/sources.list
#vim /etc/apt/sources.list
#添加如下内容
deb https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-security main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-updates main restricted universe multiverse

# deb https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse
# deb-src https://mirrors.aliyun.com/ubuntu/ jammy-proposed main restricted universe multiverse

deb https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse
deb-src https://mirrors.aliyun.com/ubuntu/ jammy-backports main restricted universe multiverse

# apt upgrade
# apt update 

1.4 配置阿里云docker-ce源
1.4.1 配置必要的目录，所有节点
root@master:~# mkdir -pv /opt/k8s_cluster_install/{yamls,scripts,packages}
mkdir: created directory '/opt/k8s_cluster_install'
mkdir: created directory '/opt/k8s_cluster_install/yamls'
mkdir: created directory '/opt/k8s_cluster_install/scripts'
mkdir: created directory '/opt/k8s_cluster_install/packages'

1.4.2 master节点搭建samba服务器
安装samba服务
# apt install samba -y

配置Samba
#vim /etc/samba/smb.conf
#在文件末尾添加以下内容来创建一个共享目录：
[shared]
   comment = Shared Folder
   path = /opt/k8s_cluster_install/packages
   browseable = yes
   writable = yes
   read only = no
   create mask = 0777
   directory mask = 0777
   public = yes

添加用户并对共享目录授权，若用户组不存在则手动创建
sudo groupadd nobody
sudo usermod -g nobody nobody
sudo mkdir -p /srv/samba/shared
sudo chmod -R 777 /srv/samba/shared
sudo chown -R nobody:nogroup /opt/k8s_cluster_install/packages

设置samba用户
# smbpasswd -a username

重启服务
systemctl restart smbd && systemctl enable smbd

访问方式：
Windows：打开文件资源管理器，输入\\<Ubuntu的IP地址>\shared
\\11.0.1.21\shared

Linux：使用smbclient或挂载共享目录：
smbclient //<Ubuntu的IP地址>/shared -U username

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

1.6 设置代理脚（可选）
#设置代理
#apt install -y expect
#cat set_porxy.sh
#!/bin/bash

# 定义日志文件
LOG_FILE="set_proxy_output.log"

# 清空日志文件
> "$LOG_FILE"

# 定义代理地址
PROXY_1="192.168.2.11:3213"
PROXY_2="172.16.38.20:3213"

# 提示用户输入代理地址
read -p "请输入代理地址（192.168.2.11:3213 或 172.16.38.20:3213）: " proxy_address

# 检查代理地址是否有效
if [[ "$proxy_address" != "$PROXY_1" && "$proxy_address" != "$PROXY_2" ]]; then
  echo "错误：代理地址无效！" | tee -a "$LOG_FILE"
  exit 1
fi

# 提示用户输入操作（on 或 off）
read -p "请输入操作（on 开启代理，off 关闭代理）: " operation

# 检查操作是否有效
if [[ "$operation" != "on" && "$operation" != "off" ]]; then
  echo "错误：操作无效！" | tee -a "$LOG_FILE"
  exit 1
fi

# 根据用户输入设置或取消代理
if [[ "$operation" == "on" ]]; then
  # 删除 ~/.bashrc 中已有的代理配置
  sed -i '/export http_proxy=/d' ~/.bashrc
  sed -i '/export https_proxy=/d' ~/.bashrc

  # 在 ~/.bashrc 文件末尾添加代理配置
  echo "export http_proxy=http://$proxy_address" >> ~/.bashrc
  echo "export https_proxy=http://$proxy_address" >> ~/.bashrc

  echo "代理已永久开启：$proxy_address" | tee -a "$LOG_FILE"
  echo "请运行以下命令使配置立即生效：" | tee -a "$LOG_FILE"
  echo "source ~/.bashrc" | tee -a "$LOG_FILE"
else
  # 删除 ~/.bashrc 中已有的代理配置
  sed -i '/export http_proxy=/d' ~/.bashrc
  sed -i '/export https_proxy=/d' ~/.bashrc

  echo "代理已永久关闭" | tee -a "$LOG_FILE"
  echo "请运行以下命令使配置立即生效：" | tee -a "$LOG_FILE"
  echo "source ~/.bashrc" | tee -a "$LOG_FILE"
fi

# 脚本正常退出
exit 0

2.0  配置节点免密
#apt install -y expect ansible 
#ansible-2.10.8版本使用apt安装不会生成默认配置文件，为避免后续在做节点免密时手动确认密钥的操作，需添加参数
root@master:/opt/k8s_cluster_install/scripts# ansible --version
ansible 2.10.8
  config file = None
  configured module search path = ['/root/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.10.12 (main, Feb  4 2025, 14:57:36) [GCC 11.4.0]
root@master:/opt/k8s_cluster_install/scripts# vim /etc/ansible/ansible.cfg
[defaults]
inventory = /etc/ansible/hosts
host_key_checking = False

root@master:/opt/k8s_cluster_install/scripts# cat /etc/ansible/hosts 
[all]
localhost
11.0.1.22
11.0.1.23

[workers]
11.0.1.22
11.0.1.23

[local]
localhost

2.1 编写免密脚本
#!/bin/bash

# 创建密钥对
echo "Generating SSH key pair..."
ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa -q

# 定义主机列表
k8s_host_list=("master" "worker-1" "worker-2")

# 从外部文件读取密码
read -sp "Enter the root password for all nodes: " mypasswd
echo

# 配置免密登录
# cat > free_login.sh << EOF
for i in "${k8s_host_list[@]}"; do
    echo "Configuring password-free login for $i..."
    expect -c "
    set timeout 20
    spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@$i
    expect {
        \"*yes/no*\" { send \"yes\r\"; exp_continue }
        \"*password:*\" { send \"$mypasswd\r\"; exp_continue }
        timeout { puts \"Timeout while waiting for password prompt on $i\"; exit 1 }
    }
    "
    if [ $? -eq 0 ]; then
        echo "Successfully configured password-free login for $i."
    else
        echo "Failed to configure password-free login for $i."
    fi
done
EOF 

# 验证连接
for i in "${k8s_host_list[@]}"; do
    echo "Testing connection to $i..."
    ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@$i "echo Connection successful"
    if [ $? -eq 0 ]; then
        echo "Connection test passed for $i."
    else
        echo "Connection test failed for $i."
    fi
done

#chmod +x free_login.sh 
#./free_login.sh 

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

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /             'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* 203.107.6.88                  2   6    17    12   -662us[ -525us] +/-   22ms
^+ 8.149.241.96                  2   6    17    13   +261us[ +398us] +/-   23ms

worker节点： 
#vim /etc/chrony/chrony.conf
#注释默认NTP地址，只同步master地址时间
server 11.0.1.21 iburst

检查同步时间是否正常
root@worker-2:~# systemctl daemon-reload 
root@worker-2:~# systemctl restart chrony
root@worker-2:~# chronyc sources -v

  .-- Source mode  '^' = server, '=' = peer, '#' = local clock.
 / .- Source state '*' = current best, '+' = combined, '-' = not combined,
| /             'x' = may be in error, '~' = too variable, '?' = unusable.
||                                                 .- xxxx [ yyyy ] +/- zzzz
||      Reachability register (octal) -.           |  xxxx = adjusted offset,
||      Log2(Polling interval) --.      |          |  yyyy = measured offset,
||                                \     |          |  zzzz = estimated error.
||                                 |    |           \
MS Name/IP address         Stratum Poll Reach LastRx Last sample               
===============================================================================
^* master                        3   6    17     3  +3918ns[  +14us] +/-   26ms


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
I0320 16:36:43.042012    4798 version.go:256] remote version is much newer: v1.32.3; falling back to: stable-1.28
registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.15
registry.aliyuncs.com/google_containers/kube-controller-manager:v1.28.15
registry.aliyuncs.com/google_containers/kube-scheduler:v1.28.15
registry.aliyuncs.com/google_containers/kube-proxy:v1.28.15
registry.aliyuncs.com/google_containers/pause:3.9
registry.aliyuncs.com/google_containers/etcd:3.5.9-0
registry.aliyuncs.com/google_containers/coredns:v1.10.1

3.1.2 拉取镜像：
root@master:~# kubeadm config images pull  --image-repository=registry.aliyuncs.com/google_containers
I0320 11:03:54.532171   43759 version.go:256] remote version is much newer: v1.32.3; falling back to: stable-1.28
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-apiserver:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-controller-manager:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-scheduler:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/kube-proxy:v1.28.15
[config/images] Pulled registry.aliyuncs.com/google_containers/pause:3.9
[config/images] Pulled registry.aliyuncs.com/google_containers/etcd:3.5.9-0
[config/images] Pulled registry.aliyuncs.com/google_containers/coredns:v1.10.1

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
