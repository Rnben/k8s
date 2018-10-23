# -*- mode: ruby -*-
# vi: set ft=ruby :

ENV["LC_ALL"] = "en_US.UTF8"

Vagrant.configure("2") do |config|
  (1..3).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = "bento/centos-7.4"
      node.vm.hostname = "node#{i}"
      node.ssh.username = "root"
      node.ssh.password = "vagrant"
      node.ssh.insert_key = true
      node.vm.box_check_update = false
      node.vm.network "private_network", ip: "11.11.11.11#{i}"
      node.vm.provider "virtualbox" do |vb|
              vb.customize ["modifyvm", :id, "--name", "node#{i}", "--memory", "1024"]
      end
      node.vm.provision "shell", inline: <<-SHELL
# basic
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
vm.swappiness=0
EOF
sysctl --system
cat <<EOF > /etc/hosts
11.11.11.111 node1
11.11.11.112 node2
11.11.11.113 node3
EOF
swapoff -a && sed -i 's/.*swap.*/#&/' /etc/fstab
sed -i 's/SELINUX=permissive/SELINUX=disabled/' /etc/sysconfig/selinux && setenforce 0
# 开启forward
# Docker从1.13版本开始调整了默认的防火墙规则,禁用了iptables filter表中FOWARD链.会引起Kubernetes集群中跨Node的Pod无法通信
iptables -P FORWARD ACCEPT
yum install -y yum-utils device-mapper-persistent-data lvm2 epel-release
# docker 
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install vim docker-ce
mkdir -p /usr/k8s/bin/
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/k8s/bin/cfssl
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/k8s/bin/cfssljson
wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/k8s/bin/cfssl-certinfo
chmod +x /usr/k8s/bin/
#systemctl start docker && systemctl enable docker
# kubeadm
#cat <<EOF > /etc/yum.repos.d/kubernetes.repo
#[kubernetes]
#name=Kubernetes
#baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
#enabled=1
#gpgcheck=1
#repo_gpgcheck=1
#gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
#EOF
#yum install -y kubectl-1.11.2 kubelet-1.11.2 kubeadm-1.11.2
#cat >/etc/sysconfig/kubelet<<EOF
#KUBELET_EXTRA_ARGS="--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.1"
#EOF
echo "Hello,enjoy!!"
      SHELL
      end
  end
end
