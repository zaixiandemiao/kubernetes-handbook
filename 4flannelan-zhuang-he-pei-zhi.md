# Flannel安装和配置

## 安装Flannel
直接使用yum进行安装
```
$ yum list flanneld
$ yum install -y flanneld
```
## 配置Flannel
使用yum安装后，会生成/usr/lib/systemd/system/flanneld.service配置文件
```
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/flanneld
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=/usr/bin/flanneld-start $FLANNEL_OPTIONS
ExecStartPost=/usr/libexec/flannel/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
```
可以看到flannel环境变量配置文件在/etc/sysconfig/flanneld
```
# Flanneld configuration options  

# etcd url location.  Point this to the server where etcd runs
FLANNEL_ETCD_ENDPOINTS="https://172.20.0.113:2379,https://172.20.0.114:2379,https://172.20.0.115:2379"

# etcd config key.  This is the configuration key that flannel queries
# For address range assignment
FLANNEL_ETCD_PREFIX="/kube-centos/network"

# Any additional options that you want to pass
FLANNEL_OPTIONS="-etcd-cafile=/etc/etcd/ssl/ca.pem -etcd-certfile=/etc/etcd/ssl/etcd.pem -etcd-keyfile=/etc/etcd/ssl/etcd-key.pem"
```
* etcd的地址FLANNEL_ETCD_ENDPOINT
* etcd查询的目录，包含docker的IP地址段配置。FLANNEL_ETCD_PREFIX, 需要在etcd集群中有对应的路径
* FLANNEL_OPTIONS配置了TLS证书

## 在etcd中创建网络配置
执行下面的命令为docker分配IP地址段，用于启动容器时分配ip
```
etcdctl mkdir /kube-centos/network
etcdctl mk /kube-centos/network/config "{ \"Network\": \"10.254.0.0/16\", \"SubnetLen\": 24, \"Backend\": { \"Type\": \"vxlan\" } }"
```
ip地址需要为10开头的，kubernetes集群安装kube-dns时，配置为别的曾经出错
## 启动flannel
```
$ systemctl daemon-reload
$ systemctl enable flanneld
$ systemctl start  flanneld
$ systemctl status flanneld
```
## flannel 子网段
Flannel启动后，应当有`/run/flannel/subnet.env`文件, source使之生效
```
$ cat /run/flannel/subnet.env
FLANNEL_NETWORK=10.254.0.0/16
FLANNEL_SUBNET=10.254.46.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=false
$ source /run/flannel/subnet.env
```

## 查询Etcd中的内容
```
$etcdctl ls /kube-centos/network/subnets
/kube-centos/network/subnets/10.254.14.0-24
/kube-centos/network/subnets/10.254.38.0-24
/kube-centos/network/subnets/10.254.46.0-24
$etcdctl get /kube-centos/network/config
{ "Network": "10.254.0.0/16", "SubnetLen": 24, "Backend": { "Type": "vxlan" } }
$etcdctl get /kube-centos/network/subnets/10.254.14.0-24
{"PublicIP":"10.254.0.114","BackendType":"vxlan","BackendData":{"VtepMAC":"56:27:7d:1c:08:22"}}
$etcdctl get /kube-centos/network/subnets/10.254.38.0-24
{"PublicIP":"10.254.0.115","BackendType":"vxlan","BackendData":{"VtepMAC":"12:82:83:59:cf:b8"}}
$etcdctl get /kube-centos/network/subnets/10.254.46.0-24
{"PublicIP":"10.254.0.113","BackendType":"vxlan","BackendData":{"VtepMAC":"e6:b2:fd:f6:66:96"}}
```
## 查询ip
使用`ip addr`,此时docker和flannel应当在同一网段中
```
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN 
    link/ether 02:42:da:bf:83:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.254.38.1/24 brd 172.30.38.255 scope global docker0
       valid_lft forever preferred_lft forever
7: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN 
    link/ether 9a:29:46:61:03:44 brd ff:ff:ff:ff:ff:ff
    inet 10.254.38.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
```