---
title: Deploy kubernetes in Fedora 36
date: 2022-08-02T01:16:00+08:00
slug: deploy_k8s_in_fedora
categories:
tags:
    - k8s
---

> [Fedora 36 : Kubernetes](https://www.server-world.info/en/note?os=Fedora_36&p=kubernetes&f=1)

# 配置网络

```bash
# dnf install langpacks-zh_CN glibc-all-langpacks -y

cat <<EOF | sudo tee /etc/sysctl.d/99-k8s-cri.conf
net.bridge.bridge-nf-call-iptables=1  
net.ipv4.ip_forward=1  
net.bridge.bridge-nf-call-ip6tables=1
EOF
```

```bash
# 开启所需网络模块 overlay br_netfilter
echo -e overlay\\nbr_netfilter > /etc/modules-load.d/k8s.conf

# nftables切换到 iptables-legacy, 不切换发现也没问题
sudo dnf install iptables-legacy
sudo alternatives --config iptables

# 关闭 swap
sudo touch /etc/systemd/zram-generator.conf

# 原文有切换到 cgroup1 目前没发现不切换有什么问题.

# 关闭防火墙
sudo systemctl disable --now firewalld

# Set SELinux in permissive mode (effectively disabling it)
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

# 关闭 systemd-resolved 防止解析 dns 回环
# [docker - Kubernetes CoreDNS in CrashLoopBackOff - Stack Overflow](https://stackoverflow.com/questions/53559291/kubernetes-coredns-in-crashloopbackoff)
sudo systemctl disable --now systemd-resolved

sudo vi /etc/NetworkManager/NetworkManager.conf
# conf
# [main]
# dns=default
sudo unlink /etc/resolv.conf
sudo touch /etc/resolv.conf

sudo reboot
```

# 安装必要软件

使用 `containerd`

```bash
# https://github.com/containerd/containerd/releases
sudo tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system
sudo mv containerd.service /usr/local/lib/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
# https://github.com/opencontainers/runc/releases
install -m 755 runc.amd64 /usr/local/sbin/runc
# https://github.com/containernetworking/plugins/releases
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz

sudo mkdir /etc/containerd
containerd config default | cat | sudo tee /etc/containerd/config.toml &>/dev/null

# 修改 config 文件 SystemdCgroup = true
sudo systemctl restart containerd
```

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

# 初始化及后续配置

```bash
sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

sudo kubeadm init --pod-network-cidr=10.1.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


# 如果忘了 join 命令可以使用如下命令获得.
kubeadm token create --print-join-command

# 配置网络容器
wget https://docs.projectcalico.org/manifests/calico.yaml
#           - name: IP
#              value: "autodetect"
#           - name: IP_AUTODETECTION_METHOD
#              value: "interface=eth.*"
kubectl apply -f calico.yaml

# 其它集群将 kubeadm init 替换成 join 命令就可以了.

# 在需要操作容器的节点安装 nerdctl
# https://github.com/containerd/nerdctl/releases
sudo tar Cxzvf /usr/local/bin nerdctl-0.22.0-linux-amd64.tar.gz
mkdir -p $HOME/.local/bin
chmod 700 $HOME/.local/bin
cp /usr/local/bin/nerdctl $HOME/.local/bin
sudo chown root $HOME/.local/bin/nerdctl
sudo chmod +s $HOME/.local/bin/nerdctl
exec zsh
# 配置补全
nerdctl completion zsh | cat | sudo tee /usr/share/zsh/site-functions/_nerdctl &>/dev/null
exec zsh
```