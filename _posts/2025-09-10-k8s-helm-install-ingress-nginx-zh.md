---
layout    : post
title     : "使用helm安装ingress-nginx"
date      : 2025-09-10
lastupdate: 2025-09-10
categories: k8s
---
# 使用Helm部署ingress-nginx【社区版本非F5 ingress-nginx企业版】

## 快速部署直接使用下面命令
**前提可以直接拉取外网的镜像**
```shell
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```
  
## 下载helm包进行手动配置
修改配置版,下载helm包，并修改valuse.yaml文件  
```shell
[root@easzlab ~]# helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
"ingress-nginx" has been added to your repositories
[root@easzlab ~]# helm repo update 
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ingress-nginx" chart repository
Update Complete. ⎈Happy Helming!⎈
[root@easzlab ~]#
[root@easzlab ~]#
[root@easzlab ~]#
[root@easzlab ~]# helm pull ingress-nginx/ingress-nginx 
[root@easzlab ~]# ls
anaconda-ks.cfg  ezdown  ingress-nginx-4.13.2.tgz
[root@easzlab ~]# tar -xf ingress-nginx-4.13.2.tgz 
[root@easzlab ~]# 
[root@easzlab ~]# ls
anaconda-ks.cfg  ezdown  ingress-nginx  ingress-nginx-4.13.2.tgz
[root@easzlab ~]#
[root@easzlab ~]#
[root@easzlab ingress-nginx]# cp values.yaml values-2.yaml
[root@easzlab ingress-nginx]# sed -n "5,8p" values-2.yaml
global:
  image:
    # -- Registry host to pull images from.
    registry: k8s.nju.edu.cn
[root@easzlab ingress-nginx]# sed -n "225,226p" values-2.yaml
  # -- Use a `DaemonSet` or `Deployment`
  kind: DaemonSet
[root@easzlab ingress-nginx]# sed -n "125,131p" values-2.yaml 
  # IngressClasses are immutable and cannot be changed after creation.
  # We do not support namespaced IngressClasses, yet, so a ClusterRole and a ClusterRoleBinding is required.
  ingressClassResource:
    # -- Name of the IngressClass
    name: nginx
    # -- Create the IngressClass or not
    enabled: true
```
配置文件修改好后使用helm install进行安装  
```shell
[root@master ~]# ls
anaconda-ks.cfg  ezdown  ingress-nginx  ingress-nginx-4.13.2.tgz
[root@master ~]# ls ingress-nginx
changelog  Chart.yaml  ci  cloudbuild.yaml  OWNERS  README.md  README.md.gotmpl  templates  tests  values-2.yaml  values.yaml
[root@master ~]# helm upgrade --install ingress-nginx ./ingress-nginx --namespace ingress-nginx --create-namespace --values ./ingress-nginx/values-2.yaml 
```
查看部署结果  
```
[root@master ~]# kubectl get po,svc -n ingress-nginx 
NAME                                 READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-5rwjm   1/1     Running   0          8m50s
pod/ingress-nginx-controller-s8l4l   1/1     Running   0          8m50s

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.68.171.158   <pending>     80:32530/TCP,443:32533/TCP   8m50s
service/ingress-nginx-controller-admission   ClusterIP      10.68.91.232    <none>        443/TCP                      8m50s
[root@master ~]# kubectl get po,svc,deamonset -n ingress-nginx 
error: the server doesn't have a resource type "deamonset"
[root@master ~]# kubectl get daemonsets.apps,po,svc -n ingress-nginx 
NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/ingress-nginx-controller   2         2         2       2            2           kubernetes.io/os=linux   9m18s

NAME                                 READY   STATUS    RESTARTS   AGE
pod/ingress-nginx-controller-5rwjm   1/1     Running   0          9m18s
pod/ingress-nginx-controller-s8l4l   1/1     Running   0          9m18s

NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/ingress-nginx-controller             LoadBalancer   10.68.171.158   <pending>     80:32530/TCP,443:32533/TCP   9m18s
service/ingress-nginx-controller-admission   ClusterIP      10.68.91.232    <none>        443/TCP                      9m18s
[root@master ~]# poweroff

```
