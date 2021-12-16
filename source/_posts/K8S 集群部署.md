---
title: K8S 集群部署
categories: 
- linux
- 应用部署
tags: 
- 未完成
---

## 在安装过程中没有找到 ebtables 或者其他类似的可执行文件
`yum install ebtables ethtool`

### 安装

```shell

# 修改docker使用systemd来管里容器的cgroup，而不是cgroupfs
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 国内节点
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
# 关闭swap
swapoff -a
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

kubeadm init --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```