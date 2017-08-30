# Kubernetes安装手册

本文档介绍Kubernetes1.6.x集群的安装

# 集群详情

* Kubernetes 1.6.0
* Docker 1.12.5（使用yum安装）
* Etcd 3.1.5
* Flanneld 0.7 vxlan 网络
* TLS 认证通信 \(所有组件，如 etcd、kubernetes master 和 node\)
* RBAC 授权
* kublet TLS BootStrapping
* kubedns、dashboard、
* heapster\(influxdb、grafana\)、监控模块
* EFK\(elasticsearch、fluentd、kibana\) 日志采集模块
* 私有docker镜像仓库harbor

# 参考博客

本安装手册主要参考了开源书[Kubernetes handbook](https://rootsongjc.gitbooks.io/kubernetes-handbook/content/)中的最佳实践部分，在此对原作者表示感谢。在实践中，结合自己在安装中遇到的一些坑，进行了适当的修改，得到这篇手册，以方便自己日后的使用


