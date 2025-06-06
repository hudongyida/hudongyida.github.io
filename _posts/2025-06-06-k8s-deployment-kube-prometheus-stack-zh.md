---
layout    : post
title     : "helm部署kube-prometheus-stack"
date      : 2025-06-06
lastupdate: 2025-06-06
categories: k8s
---
# 使用Helm部署 kube-Prometheus-stack监控套件
## 先决条件
- k8s 版本>1.19
- Helm 版本>3.0

## 配置Helm仓库信息
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 使用Helm部署
使用下面命令可以快速部署**但是不推荐该部署方式**
```sh
helm install [Helm部署名称] prometheus-community/kube-prometheus-stack
```
**推荐使用下面方式进行部署
```sh
#在k8s集群中创建名称空间
kubectl create ns monitoring

#使用Helm install在monitoring名称空间下进行部署
helm install -n monitoring prometheus-community/kube-prometheus-stack --generate-name
# --generate-name是随机名称，如果不需要随机名称可以手写，如下【假设名称为prometheus1】
# helm install -n monitoring prometheus-community/kube-prometheus-stack prometheus1
```

## 移除kube-prometheus-stack
```sh
helm uninstall [Helm部署名称]

# 如果是使用上文的部署形式可以使用下面命令查看Helm部署名称
helm list -n monitoring

# 如果想要完全移除，请继续执行下面命令，Helm默认不会移除CRD
kubectl delete crd alertmanagerconfigs.monitoring.coreos.com
kubectl delete crd alertmanagers.monitoring.coreos.com
kubectl delete crd podmonitors.monitoring.coreos.com
kubectl delete crd probes.monitoring.coreos.com
kubectl delete crd prometheusagents.monitoring.coreos.com
kubectl delete crd prometheuses.monitoring.coreos.com
kubectl delete crd prometheusrules.monitoring.coreos.com
kubectl delete crd scrapeconfigs.monitoring.coreos.com
kubectl delete crd servicemonitors.monitoring.coreos.com
kubectl delete crd thanosrulers.monitoring.coreos.com
```

## Web访问【在实验环境中】
由于Helm部署使用的是Cluster IP 类型，只能在集群内相互访问，如果要实现对外的访问则需要使用其他手段，比如Nginx ingress,下面为了演示效果使用NodePort的方式进行暴露  
使用命令查看svc
```sh
[root@master-01 linux-amd64]# kubectl get svc -n monitoring 
NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                                       ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   24m
kube-prometheus-stack-1749-alertmanager                     ClusterIP   10.68.96.212    <none>        9093/TCP,8080/TCP            24m
kube-prometheus-stack-1749-operator                         ClusterIP   10.68.221.167   <none>        443/TCP                      24m
kube-prometheus-stack-1749-prometheus                       ClusterIP   10.68.254.233   <none>        9090/TCP,8080/TCP            24m
kube-prometheus-stack-1749200426-grafana                    ClusterIP   10.68.191.93    <none>        80/TCP                       24m #需要修改
kube-prometheus-stack-1749200426-kube-state-metrics         ClusterIP   10.68.33.94     <none>        8080/TCP                     24m
kube-prometheus-stack-1749200426-prometheus-node-exporter   ClusterIP   10.68.125.1     <none>        9100/TCP                     24m
prometheus-operated                                         ClusterIP   None            <none>        9090/TCP                     24m
```
修改 grafana
```sh
kubectl edit svc -n monitoring kube-prometheus-stack-1749200426-grafana
```
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    meta.helm.sh/release-name: kube-prometheus-stack-1749200426
    meta.helm.sh/release-namespace: monitoring
  creationTimestamp: "2025-06-06T09:00:47Z"
  labels:
    app.kubernetes.io/instance: kube-prometheus-stack-1749200426
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 12.0.0-security-01
    helm.sh/chart: grafana-9.2.1
  name: kube-prometheus-stack-1749200426-grafana
  namespace: monitoring
  resourceVersion: "26207"
  uid: a4db2c82-fa9a-4ec8-a373-6bb3cf49ea31
spec:
  clusterIP: 10.68.191.93
  clusterIPs:
  - 10.68.191.93
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: http-web
    port: 30003                             #修改端口到30000以上
    protocol: TCP
    targetPort: 3000
    nodePort: 30003                         #添加该参数
  selector:
    app.kubernetes.io/instance: kube-prometheus-stack-1749200426
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort                            #修改type类型为NodePort 
status:
  loadBalancer: {}
```
在浏览器中输入 计算节点的IP地址:30003访问grafana  
使用下面命令获取账户密码
```sh
# 查看grafana秘钥文件
[root@master-01 linux-amd64]# kubectl get -n monitoring secrets | grep grafana
kube-prometheus-stack-1749200426-grafana                                              Opaque               3      54m

# 使用下面命令获取秘钥中的管理员用户信息，并且使用base64进行解码
[root@master-01 linux-amd64]# kubectl get secret -n monitoring kube-prometheus-stack-1749200426-grafana   -o jsonpath='{.data.admin-user}' | base64 --decode
admin

# 使用下面命令获取秘钥中的管理员用户密码，并且使用base64进行解码
[root@master-01 linux-amd64]# kubectl get secret -n monitoring kube-prometheus-stack-1749200426-grafana   -o jsonpath='{.data.admin-password}' | base64 --decode
prom-operator
```
