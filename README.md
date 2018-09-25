# k8s

# 安装手册
>https://github.com/Rnben/k8s

## 集群环境

|主机名|IP|安装软件|
|---|---|---|
|node1|11.11.11.111|kubeadm、kubelet、kubectl|
|node2|11.11.11.112|kubeadm、kubelet|

使用vagrant创建虚拟机

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV["LC_ALL"] = "en_US.UTF8"

Vagrant.configure("2") do |config|
  (1..2).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = "bento/centos-7.4"
      node.vm.hostname = "node#{i}"
      node.ssh.username = "root"
      node.ssh.password = "vagrant"
      node.ssh.insert_key = true
      node.vm.box_check_update = false
      node.vm.network "private_network", ip: "11.11.11.11#{i}"
      node.vm.provider "virtualbox" do |vb|
              vb.customize ["modifyvm", :id, "--name", "node#{i}", "--memory", "2048"]
      end
      node.vm.provision "shell", inline: <<-SHELL
        echo "Hello,enjoy!!"
      SHELL
      end
  end
end
```
## 主机初始化

>所有主机操作

```
# 关闭swap
swapoff -a && sed -i 's/.*swap.*/#&/' /etc/fstab

# 禁用selinux
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux && setenforce 0

# 开启forward
iptables -P FORWARD ACCEPT

# 配置转发相关参数
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF
sysctl --system

# hosts配置
cat <<EOF > /etc/hosts
11.11.11.111 node1
11.11.11.112 node2
EOF
```

## 安装docker

```
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install vim docker-ce
systemctl start docker && systemctl enable docker
```

## 安装Kubeadm 
使用阿里镜像源安装
```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubectl-1.11.2 kubelet-1.11.2 kubeadm-1.11.2

cat <<EOF >/etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS="--cgroup-driver=$DOCKER_CGROUPS --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
EOF
```

> 官方镜像导出 `kubeadm config images list --kubernetes-version=v1.11.2`

## 配置master节点

*kubeadm-master.config*
```
apiVersion: kubeadm.k8s.io/v1alpha2
kind: MasterConfiguration
kubernetesVersion: v1.11.2
api:
  advertiseAddress: 11.11.11.111
# controlPlaneEndpoint: 11.11.11.110:8443
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
networking:
  podSubnet: 10.244.0.0/16
kubeProxy:
  config:
    mode: iptables
```

*拉取镜像* 
`kubeadm config images pull --config kubeadm-master.config`

*初始化*
`kubeadm init --config kubeadm-master.config`
*保存最后输出的 join 命令*
`kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>`

>查看或新建token
>`kubeadm token list`
>`kubeadm token create`
>
>*-discovery-token-ca-cert-hash*
>openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin >-outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

*配置kubectl*
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

查看节点状态
`kubectl get nodes`

移除master节点的污点
`kubectl taint nodes --all node-role.kubernetes.io/master-`

## 配置网络插件
`kubectl create -f kube-flannel.yaml`

## 添加node节点
`kubeadm join ... `



## 参考链接
https://www.kubernetes.org.cn/4256.html
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#join-nodes
https://github.com/kubernetes/dashboard
https://jimmysong.io/kubernetes-handbook/practice/dashboard-addon-installation.html
