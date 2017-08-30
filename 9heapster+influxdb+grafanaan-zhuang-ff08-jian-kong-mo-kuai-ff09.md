# Heapster+InfluxDB+Grafana安装
## 下载压缩包
到 [heapster release 页面](https://github.com/kubernetes/heapster/releases) 下载 heapster。
```bash
$ wget https://github.com/kubernetes/heapster/archive/v1.3.0.zip
$ unzip v1.3.0.zip
$ mv v1.3.0.zip heapster-1.3.0
```
文件目录：`heapster-1.3.0/deploy/kube-config/influxdb`  
```bash
$ cd heapster-1.3.0/deploy/kube-config/influxdb
$ ls *.yaml
grafana-deployment.yaml  grafana-service.yaml  heapster-deployment.yaml  heapster-service.yaml  influxdb-deployment.yaml  influxdb-service.yaml heapster-rbac.yaml
```
由于kubernetes集群中使用了RBAC认证，因此需要创建heapster的RBAC配置`heapster-rbac.yaml`  
已经修改好的yaml文件: [heapster](https://github.com/zaixiandemiao/kubernetes-install/tree/master/manifest/heapster),也可以结合kubernetes源码中的yaml配置文件：[kubernetes heapster](https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/cluster-monitoring/influxdb)
## 配置 grafana-deployment

```bash
$ diff grafana-deployment.yaml.orig grafana-deployment.yaml
16c16
<         image: gcr.io/google_containers/heapster-grafana-amd64:v4.0.2
---
>         image: registry.cn-hangzhou.aliyuncs.com/lczean/heapster-grafana-amd64:v4.0.2
40,41c40,41
<           # value: /api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/
<           value: /
---
>           value: /api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/
>           #value: /
```
* image可以暂时使用[阿里云](https://cr.console.aliyun.com/#/imageSearch),确认可以使用后，再上传到私有仓库中,为保证不是镜像下载时出现问题，可以先docker pull到本地，确保镜像完整下载
* 如果后续使用 kube-apiserver 或者 kubectl proxy 访问 grafana dashboard，则必须将 GF_SERVER_ROOT_URL 设置为`/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/`，否则后续访问grafana时访问时提示找不到`http://10.64.3.7:8086/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana/api/dashboards/home` 页面

## 配置heapster-deployments.yaml

```bash
$ diff heapster-deployment.yaml.orig heapster-deployment.yaml
16c16
>         image: sz-pg-oam-docker-hub-001.tendcloud.com/library/heapster-amd64:v1.3.0-beta.1
>         - --source=kubernetes:http://192.168.202.131:8080?inClusterConfig=false
```

* --source指定的是kube-apiserver的地址，使用8080 insecure端口进行访问

## 配置 influxdb-deployment
influxdb 官方建议使用命令行或 HTTP API 接口来查询数据库，从 v1.1.0 版本开始默认关闭 admin UI，将在后续版本中移除 admin UI 插件。

开启镜像中 admin UI的办法如下：先导出镜像中的 influxdb 配置文件，开启 admin 插件后，再将配置文件内容写入 ConfigMap，最后挂载到镜像中，达到覆盖原始配置的目的：

注意：manifests 目录已经提供了 [修改后的 ConfigMap 定义文件](https://github.com/opsnull/follow-me-install-kubernetes-cluster/blob/master/manifests/heapster/influxdb-cm.yaml), 可以直接将其放在heapster目录下，一起创建, 确保[admin]中的`enable = true`

```bash
$ # 将 ConfigMap 中的配置文件挂载到 Pod 中，达到覆盖原始配置的目的
$ diff influxdb-deployment.yaml.orig influxdb-deployment.yaml
16c16
<         image: grc.io/google_containers/heapster-influxdb-amd64:v1.1.1
---
>         image: sz-pg-oam-docker-hub-001.tendcloud.com/library/heapster-influxdb-amd64:v1.1.1
19a20,21
>         - mountPath: /etc/
>           name: influxdb-config
22a25,27
>       - name: influxdb-config
>         configMap:
>           name: influxdb-config
```

## 配置 monitoring-influxdb Service

```bash
 diff influxdb-service.yaml.orig influxdb-service.yaml
12a13
>   type: NodePort
15a17,20
>     name: http
>   - port: 8083
>     targetPort: 8083
>     name: admin
```

* 定义端口类型为 NodePort，额外增加了 admin 端口映射，用于后续浏览器访问 influxdb 的 admin UI 界面

## 执行所有定义文件

```bash
$ ls *.yaml
grafana-service.yaml      heapster-rbac.yaml     influxdb-cm.yaml          influxdb-service.yaml
grafana-deployment.yaml  heapster-deployment.yaml  heapster-service.yaml  influxdb-deployment.yaml
$ kubectl create -f  .
deployment "monitoring-grafana" created
service "monitoring-grafana" created
deployment "heapster" created
serviceaccount "heapster" created
clusterrolebinding "heapster" created
service "heapster" created
configmap "influxdb-config" created
deployment "monitoring-influxdb" created
service "monitoring-influxdb" created
```

## 检查结果

检查 Deployment
```bash
$ kubectl get deployments -n kube-system | grep -E 'heapster|monitoring'
heapster               1         1         1            1           2m
monitoring-grafana     1         1         1            1           2m
monitoring-influxdb    1         1         1            1           2m
```
检查 Pods
```bash
$ kubectl get pods -n kube-system | grep -E 'heapster|monitoring'
heapster-110704576-gpg8v                1/1       Running   0          2m
monitoring-grafana-2861879979-9z89f     1/1       Running   0          2m
monitoring-influxdb-1411048194-lzrpc    1/1       Running   0          2m
```
检查 kubernets dashboard 界面，看是显示各 Nodes、Pods 的 CPU、内存、负载等利用率曲线图

## 访问 grafana

1. 通过 kube-apiserver 访问：

    获取 monitoring-grafana 服务 URL

    ``` bash
    $ kubectl cluster-info
    Kubernetes master is running at https://192.168.202.131:6443
    Heapster is running at https://192.168.202.131:6443/api/v1/proxy/namespaces/kube-system/services/heapster
    KubeDNS is running at https://192.168.202.131:6443/api/v1/proxy/namespaces/kube-system/services/kube-dns
    kubernetes-dashboard is running at https://192.168.202.131:6443/api/v1/proxy/namespaces/kube-system/services/kubernetes-dashboard
    monitoring-grafana is running at https://192.168.202.131:6443/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana
    monitoring-influxdb is running at https://192.168.202.131:6443/api/v1/proxy/namespaces/kube-system/services/monitoring-influxdb
    To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
    ```

    浏览器访问 URL： `http://192.168.202.131:8080/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana`

2. 通过 kubectl proxy 访问：

    创建代理

    ``` bash
    $ kubectl proxy --address='192.168.202.131' --port=8086 --accept-hosts='^*$'
    Starting to serve on 192.168.202.131:8086
    ```

    浏览器访问 URL：`http://192.168.202.131:8086/api/v1/proxy/namespaces/kube-system/services/monitoring-grafana`

### grafana显示问题

grafana的图像是根据influxdb中的数据绘制出的，刚启动的数据库中不包含数据，所以说要稍微等待一下。

grafana中cluster只能获取到一个节点以及该节点的pod信息。这就要回到教程最初了，最开始希望的是可以通过配置iptables规则来实现访问，但是配置失败了，导致grafana只能获取到一个node的数据，这时只需要关闭防火墙就好了。

## 访问 influxdb admin UI

获取 influxdb http 8086 映射的 NodePort

```bash
$ kubectl get svc -n kube-system|grep influxdb
monitoring-influxdb    10.254.22.46    <nodes>       8086:32201/TCP,8083:30269/TCP   9m
```

通过 kube-apiserver 的**非安全端口**访问 influxdb 的 admin UI 界面： `http://192.168.202.131:8080/api/v1/proxy/namespaces/kube-system/services/monitoring-influxdb:8083/`

在页面的 “Connection Settings” 的 Host 中输入 node IP， Port 中输入 8086 映射的 nodePort 如上面的 32201，点击 “Save” 即可（我的集群中的地址是192.168.202.131:32201）：

![kubernetes-influxdb-heapster](./images/kubernetes-influxdb-heapster.jpg)
