---
title: Docker安装步骤
categories: 
- linux
- 应用部署
tags: 
- 未完成
---

```shell
sudo yum remove docker \
                docker-client \
                docker-client-latest \
                docker-common \
                docker-latest \
                docker-latest-logrotate \
                docker-logrotate \
                docker-engine
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce-<VERSION_STRING> docker-ce-cli-<VERSION_STRING> containerd.io
```