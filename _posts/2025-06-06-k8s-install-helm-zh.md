---
layout    : post
title     : "k8s集群快速部署Helm"
date      : 2025-06-06
lastupdate: 2025-06-06
categories: k8s
---
# Helm
使用下面命令进行快速部署  
```sh
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```
  
使用tar包手动部署  
```sh
wget https://get.helm.sh/helm-v3.18.1-linux-amd64.tar.gz
# 命令中的3.18.1是版本号，可以根据需要进行修改
tar -xvzf helm-v3.18.1-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```
  
添加Helm的bash补全  
```sh
source <(helm completion bash)
helm completion bash >/etc/bash_completion.d/helm
bash
```
