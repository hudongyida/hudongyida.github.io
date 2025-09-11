---
layout    : post
title     : "k8s部署postgreSQL+pgadmin"
date      : 2025-09-11
lastupdate: 2025-09-11
categories: k8s
---
# 在k8s集群中部署postgreSQL数据+pgadmin web管理界面

## 先决条件

- Kubernetes 1.23+
- Helm 3.8.0+

## postgreSQL

### 添加Helm仓库

添加helm仓库信息，并且更新
**请注意在写稿的时候broadcom公司收购了bitnami，我也不确定这个镜像能用多久。**
[详细信息](https://github.com/bitnami/containers/issues/83267)
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```  
  
拉取chart包,有环境的可以使用search搜一个包信息，直接拉取问题也不大
```shell
helm pull bitnami/postgresql
    #直接拉取包
helm search repo bitnami/postgresql
    #搜索包信息
helm pull bitnami/postgresql --version 1.49.0
    #下载指定的包信息，截止写稿时这个版本是最新的
```  
执行后会得到下面内容  
```shell
[root@master ~]# ls
postgresql-16.7.27.tgz
```  
#### 配置
解压包，并且编辑文件
```shell
tar -xf postgresql-16.7.27.tgz
    #解压包
cp postgresql/values.yaml postgresql/values-1.yaml
    #复制文件
vim postgresql/values-1.yaml
    #编辑文件，编辑内容见下
```  
```yaml
global: 
  postgresql:
    auth:
      postgresPassword: "123456"    #配置密码

primary:
  service:
    type: ClusterIP                 #因为待会会使用web界面管理，所以无需设置端口暴露
  persistence:
    size: 15Gi                       #大小，默认是8Gi
    
      #修改完成后保存退出，
```  
创建PV文件，因为这里制作演示且是单节点所以使用hostPath  
创建`pv-postgreslq.yaml`文件
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-postgresql  # PV名称，自定义
  labels:
    type: local
    app: postgresql
spec:
  capacity:
    storage: 15Gi  # 匹配values.yaml中的size
  accessModes:
    - ReadWriteOnce  # 匹配accessMode
  persistentVolumeReclaimPolicy: Retain  # 删除PVC后保留PV
  hostPath:
    path: /var/postgreSQL_data  # 您的本地路径

      #注意这里并没有使用storageClass:
      #如果要使用也请在values-1.yaml文件中编辑primary.persistence.storageClass
```  
文件编辑好了后先`chmod`权限再`kubectl apply -f`命令创建资源
```shell
    #请一定要先在work节点chmod 777 /var/postgreSQL_data
chmod 777 /var/postgreSQL_data
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #很重要，不然可能会出权限问题，导致服务器起不来，别我怎么知道的
kubectl apply -f pv-postgreslq.yaml
```  
使用命令查看
```shell
root@master ~]# kubectl get pv
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                    STORAGECLASS    VOLUMEATTRIBUTESCLASS   REASON   AGE
pv-postgresql          15Gi       RWO            Retain           Available                                                        <unset>                          92m
```  
`Available`表示没有可用，没有被绑定
### 部署
```shell
kubectl create ns postgresql
helm install postgre bitnami/postgresql -n postgresql -f ./values-1.yaml
```  
出现下面结果表示成功
```shell
[root@master gitlab]# kubectl get po -n postgresql 
NAME                                READY   STATUS    RESTARTS   AGE
postgre-postgresql-0                1/1     Running   0          96m
```

## pgadmin
### pgadmin介绍【废话】
PgAdmin是PostgreSQL最受欢迎的开源图形化管理工具，提供直观的Web和桌面界面，用于数据库的创建、管理、查询及维护，极大提升DBA和开发者的工作效率。

### 添加仓库
```shell
helm repo add runix https://helm.runix.net
helm repo update
```  
拉取chart包,有环境的可以使用search搜一个包信息，直接拉取问题也不大
```shell
helm pull runix/pgadmin4
    #直接拉取包
helm search repo runix/pgadmin4
    #搜索包信息
helm pull runix/pgadmin4 --version 1.49.0
    #下载指定的包信息，截止写稿时这个版本是最新的
```  
#### 配置
解压包，并且编辑文件
```shell
tar -xf pgadmin4-1.49.0.tgz
    #解压包
cp pgadmin4/values.yaml pgadmin4/values-1.yaml
    #复制文件
vim pgadmin4/values-1.yaml
    #编辑文件，编辑内容见下
```  
```yaml
---
    #环境变量。用于登入
env: 
  email: admin@aliruanjianyuan.com  #名称随意，用于登入
  password: "123456"                #密码随意，记住就行
    #PV设置用于持久化存储

    # Ingress 配置 - 无 SSL

#service:
#  type: NodePort  # 修改为 NodePort
#  port: 80
#  # nodePort: 30080  # 可以指定固定的 NodePort 端口（可选，范围 30000-32767）

ingress:
  enabled: true
#  enabled: false       #当service.type的值为NodePort的时候enabled: false禁用ingress
  className: "nginx"
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: "postgresqladmin.aliruanjianyuan.com" #在浏览器输入的域名
      paths:
        - path: /
          pathType: Prefix

persistentVolume:
  enabled: true
  accessMode: ReadWriteOnce
  size: 10Gi
    # 服务器自动配置
serverDefinitions: 
  enabled: true
  resourceType: ConfigMap
  servers:
    k8s-postgre:            # postgreSQL服务器名称，随意
      Name: "k8s-postgre"   # postgreSQL服务器名称，随意
      Group: "Kubernetes Servers"   # postgreSQL服务器组，随意
      Username: "postgres"  #默认为postgres，超级管理员
      Host: "postgre-postgresql.postgresql.svc.cluster.local"
        # <服务名称>.<命名空间>.svc.cluster.local
        # 使用kubectl get svc -n postgresql命令查看
        #NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
        #postgre-postgresql      ClusterIP   10.68.113.129   <none>        5432/TCP   116m
        #其中postgre-postgresql是我的svc名称，postgresql是pod所在的命名空间
        #svc.cluster.local 集群内部的固定域名格式
      Port: 5432            #postgreSQL服务端口默认
      SSLMode: "prefer"     #默认即可
      MaintenanceDB: "postgres" #默认即可

networkPolicy:
  enabled: true
```  
创建PV，同上，使用hostPath,创建`kubectl apply -f pv-pgadmin.yaml`文件
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-pgadmin  # PV名称，自定义
  labels:
    type: local
    app: postgresql
spec:
  capacity:
    storage: 10Gi  # 匹配values.yaml中的size
  accessModes:
    - ReadWriteOnce  # 匹配accessMode
  persistentVolumeReclaimPolicy: Retain  # 删除PVC后保留PV
  hostPath:
    path: /var/pgadmin_data  # 您的本地路径

      #注意这里并没有使用storageClass:
      #如果要使用也请在values-1.yaml文件中编辑persistentVolume.storageClass
```  
先修改全程`chmod 777 /var/pgadmin_data`再创建资源
```shell
    #请一定要先在work节点chmod 777 /var/pgadmin_data
chmod 777 /var/pgadmin_data
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #一定要做，一定要做
    #计算节点，不是控制节点。work节点，不是Master节点
    #很重要，不然可能会出权限问题，导致服务器起不来，别我怎么知道的
kubectl apply -f pv-pgadmin.yaml
```
### 部署
使用helm命令部署pgadmin
```shell
helm install pgadmin runix/pgadmin4 -n postgresql -f values-1.yaml
```  
查看部署结果
```shell
[root@master gitlab]# kubectl get -n postgresql po,ingress
NAME                                    READY   STATUS    RESTARTS   AGE
pod/pgadmin-pgadmin4-69c9849cf9-lxfb2   1/1     Running   0          89m
pod/postgre-postgresql-0                1/1     Running   0          125m

NAME                                         CLASS   HOSTS                                 ADDRESS   PORTS   AGE
ingress.networking.k8s.io/pgadmin-pgadmin4   nginx   postgresqladmin.aliruanjianyuan.com             80      89m
[root@master gitlab]# 
```  
我这里使用的是ingress做的方向代理，所以我使用域名访问`http://postgresqladmin.aliruanjianyuan.com`
```
[root@master pv]# curl -I http://postgresqladmin.aliruanjianyuan.com
HTTP/1.1 302 FOUND
Date: Thu, 11 Sep 2025 14:24:03 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 213
Connection: keep-alive
Location: /login?next=/
Vary: Accept-Encoding
X-Frame-Options: SAMEORIGIN
Content-Security-Policy: default-src ws: http: data: blob: 'unsafe-inline' 'unsafe-eval';
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
Set-Cookie: pga4_session=7bf90158-ea6c-4e7c-b65f-eb03294b49d6!7pXDB29y/FeWR/r0EZAQvi+Rublc89ENo/ty9/YCjUM=; Expires=Fri, 12 Sep 2025 14:24:03 GMT; HttpOnly; Path=/; SameSite=Lax

[root@master pv]# 
```  
如果你使用的NodePort的端口暴露方式请使用节点IP直接访问  
- 获取SVC运行信息，查看端口
```shell 
kubectl get svc -n postgresql | grep pgadmin
```  
结果类型下面,32456就是端口
```
NAME                TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
pgadmin-pgadmin4   NodePort   10.96.123.456   <none>        80:32456/TCP   5m
```
- 获取所有节点 IP 地址
```shell
kubectl get nodes -o wide
```
结果如下
```
NAME        STATUS                     ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                      KERNEL-VERSION                 CONTAINER-RUNTIME
master      Ready,SchedulingDisabled   master   28h   v1.32.3   10.0.0.10     <none>        Rocky Linux 9.5 (Blue Onyx)   5.14.0-503.14.1.el9_5.x86_64   containerd://2.0.4
worker-01   Ready                      node     28h   v1.32.3   10.0.0.11     <none>        Rocky Linux 9.5 (Blue Onyx)   5.14.0-503.14.1.el9_5.x86_64   containerd://2.0.4
```
- 获取pod运行节点信息
```shell
kubectl get po -n postgresql -o wide
```
结果如下,我的pgadmin-pgadmin4工作在worker-01,所以我的访问地址是`http://10.0.0.11:32456`
```
NAME                                READY   STATUS    RESTARTS   AGE    IP              NODE        NOMINATED NODE   READINESS GATES
pgadmin-pgadmin4-69c9849cf9-lxfb2   1/1     Running   0          97m    172.20.171.18   worker-01   <none>           <none>
postgre-postgresql-0                1/1     Running   0          134m   172.20.171.20   worker-01   <none>           <none>
```


2025-06-06-k8s-postgreSQL+pgadmin-dashboard-zh.md
