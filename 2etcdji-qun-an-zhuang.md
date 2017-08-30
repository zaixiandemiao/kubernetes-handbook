# 2、Etcd集群安装
## 环境信息
Centos 7
```
192.168.202.131 node1
192.168.202.132 node2
192.168.202.133 node3
```
## TLS密钥和证书
部署的etcd集群使用TLS证书对集群中节点间通信进行加密，并开启基于CA根证书签名的双向数字证书认证。本文档使用[cfssl](https://github.com/cloudflare/cfssl)来生成CA证书以及其他需要的证书。生成的证书列表如下：  
* ca.pem
* etcd.pem
* etcd-key.pem  
下面介绍使用cfssl生成所需要的私钥和证书.
## 安装[cfssl](https://github.com/cloudflare/cfssl)
### 方式一：直接使用二进制包安装  
```
$ wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ chmod +x cfssl_linux-amd64
$ sudo mv cfssl_linux-amd64 /root/local/bin/cfssl

$ wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x cfssljson_linux-amd64
$ sudo mv cfssljson_linux-amd64 /root/local/bin/cfssljson

$ wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
$ chmod +x cfssl-certinfo_linux-amd64
$ sudo mv cfssl-certinfo_linux-amd64 /root/local/bin/cfssl-certinfo

$ export PATH=/root/local/bin:$PATH
```
### 方式二：使用go命令安装
如果系统中安装过Go的话，可以直接使用命令安装
```
$ go get -u github.com/cloudflare/cfssl/cmd/...
$ echo $GOPATH
/usr/local
$ ls /usr/local/bin/cfssl*
cfssl cfssl-bundle cfssl-certinfo cfssljson cfssl-newkey cfssl-scan
```
## 创建CA证书
### 创建CA的配置文件ca-config.json
```
$ mkdir /root/ssl
$ cd /root/ssl
$ cfssl print-defaults config > ca-config.json
$ cfssl print-defaults csr > ca-csr.json
$ cat ca-config.json
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "frognew": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```
ca-config.json中可以定义多个profile，分别设置不同的expiry和usages等参数。如上面的ca-config.json中定义了名称为frognew的profile，这个profile的expiry 87600h为10年，useages中：
* signing表示此CA证书可以用于签名其他证书，ca.pem中的CA=TRUE
* server auth表示TLS Server Authentication, 即client可以用该 CA 对server提供的证书进行验证
* client auth表示TLS Client Authentication，即server可以用该CA对client提供的证书进行验证
### 创建CA证书签名请求配置ca-csr.json：
```
{
  "CN": "frognew",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "frognew",
      "OU": "cloudnative"
    }
  ]
}
```
下面使用cfssl生成CA证书和私钥:
```
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ ls ca*
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem
```
## Etcd证书和私钥
创建etcd证书签名请求配置etcd-csr.json：
```
{
    "CN": "frognew",
    "hosts": [
      "127.0.0.1",
      "192.168.202.131",
      "192.168.202.132",
      "192.168.202.133",
      "node1",
      "node2",
      "node3"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "ST": "BeiJing",
            "L": "BeiJing",
            "O": "frognew",
            "OU": "cloudnative"
        }
    ]
}
```
注意上面配置hosts字段中制定授权使用该证书的IP和域名列表，因为现在要生成的证书需要被etcd集群各个节点使用，所以这里指定了各个节点的IP和hostname。

下面生成etcd的证书和私钥：
```
$ cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=frognew etcd-csr.json | cfssljson -bare etcd

$ ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem
```
对生成的证书可以使用cfssl或者openssl查看：
```
$ cfssl-certinfo -cert etcd.pem

$ openssl x509  -noout -text -in  etcd.pem
```
* 确认 Issuer 字段的内容和 ca-csr.json 一致；
* 确认 Subject 字段的内容和 etcd-csr.json 一致；
* 确认 X509v3 Subject Alternative Name 字段的内容和 etcd-csr.json 一致；
* 确认 X509v3 Key Usage、Extended Key Usage 字段的内容和 ca-config.json 中 profile 一致；
## 安装etcd
Etcd可以使用二进制安装和yum源安装两种方式
### 二进制安装
```
wget https://github.com/coreos/etcd/releases/download/v3.1.6/etcd-v3.1.6-linux-amd64.tar.gz
```
解压缩etcd-v3.1.6-linux-amd64.tar.gz，将其中的etcd和etcdctl两个可执行文件复制到各节点的/usr/bin目录。
### yum源安装
```
$ yum list etcd
$ yum install -y etcd
```

安装完成之后，在各节点创建etcd的数据目录
```
mkdir -p /var/lib/etcd
```
使用systemctl启动和管理etcd服务，在每个节点上创建etcd的systemd unit文件/usr/lib/systemd/system/etcd.service，注意替换ETCD_NAME和INTERNAL_IP变量的值：
```
export ETCD_NAME=node1
export INTERNAL_IP=192.168.202.131
cat > /usr/lib/systemd/system/etcd.service <<EOF
[Unit]
Description=etcd server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name ${ETCD_NAME} \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-peer-urls https://${INTERNAL_IP}:2380 \
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://${INTERNAL_IP}:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster node1=https://192.168.202.131:2380,node2=https://192.168.202.132:2380,node3=https://192.168.202.133:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
* --data-dir指定了etcd的工作目录和数据目录是/var/lib/etcd
* --cert-file和--key-file分别指定etcd的公钥证书和私钥
* --peer-cert-file和--peer-key-file分别指定了etcd的Peers通信的公钥证书和私钥。
* --trusted-ca-file指定了客户端的CA证书
* --peer-trusted-ca-file指定了Peers的CA证书
* --initial-cluster-state new表示这是新初始化集群，--name指定的参数值必须在--initial-cluster中  

**注意**：在etcd.pem生成时hosts配置了Ip地址列表和hostname列表，在etcd的service(/usr/lib/systemd/system/etcd.service)文件中，所有ip不能代替为未包含的hostname，如master
## 启动Etcd
在各节点上启动etcd：
```
$ systemctl daemon-reload
$ systemctl enable etcd
$ systemctl start etcd
$ systemctl status etcd
```
在启动etcd的时候，可以开启另一个命令窗口，查看启动日志，确保没有报错
```
journalctl -f
```
* 如果出现了形如 `unkown flag`的字段，表示启动参数错误，不识别,说明该参数拼写错误(*如--keyfile应当为--key-file*),可以到官方配置文档[Configuration flags](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/configuration.md)查看该参数的写法，确保正确。
* 如果出现`Failed to find member fXXXXXX`的错误，这说明之前启动的etcd时，标识号出现错误，此时删除`/var/lib/etcd/member`目录，让etcd重新为每个节点分配标识号, `/var/lib/etcd`为etcd启动配置工作目录

如果日志一切正常，可以使用`etcdctl`检查集群是否健康，在任一节点执行：
```
etcdctl \
  --ca-file=/etc/etcd/ssl/ca.pem \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --endpoints=https://node1:2379,https://node2:2379,https://node3:2379 \
  cluster-health
  
2017-04-24 19:53:40.545148 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-04-24 19:53:40.546127 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
member 4f2f99d70000fc19 is healthy: got healthy result from https://192.168.202.132:2379
member 99a756f799eb4163 is healthy: got healthy result from https://192.168.202.131:2379
member a9aff19397de2e4e is healthy: got healthy result from https://192.168.202.133:2379
cluster is healthy
```
确保输出`cluster is healthy`的信息。
上面的命令使用证书访问，返回正常信息，若未添加证书，使用`etcdctl member list`访问，应当报错，否则，TLS(安全认证)未生效，即使用http访问etcd集群。

## etcdctl配置
由于使用了TLS安全认证，etcdctl 查询时需要在命令行中指定证书和endpoints，会使得一条命令变得很长，可以预先创建一个etcdctl配置文件，进行相应的配置.
1. 创建etcdctl配置文件
```
$ vi /etc/etcd/etcdctl
$ cat /etc/etcd/etcdctl
ETCDCTL_ENDPOINT="https://node1:2379,https://node2:2379,https://node3:2379"
ETCDCTL_CACERT=/etc/etcd/ssl/ca.pem
ETCDCTL_CERT=/etc/etcd/ssl/etcd.pem
ETCDCTL_KEY=/etc/etcd/ssl/etcd-key.pem
```
2. 使配置文件生效
```
$ source /etc/etcd/etcdctl
```
3. 查看集群状态
```
$ etcdctl cluster-health

2017-04-24 19:53:40.545148 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
2017-04-24 19:53:40.546127 I | warning: ignoring ServerName for user-provided CA for backwards compatibility is deprecated
member 4f2f99d70000fc19 is healthy: got healthy result from https://192.168.202.132:2379
member 99a756f799eb4163 is healthy: got healthy result from https://192.168.202.131:2379
member a9aff19397de2e4e is healthy: got healthy result from https://192.168.202.133:2379
cluster is healthy
```
etcdctl配置的本质是定义ETCDCTL_ENDPOINT常量，etcdctl运行时读取该常量值，进行连接，具体的常量名称可以参考官方的配置说明[etcdctl config](https://github.com/coreos/etcd/tree/master/etcdctl)

## 参考
* [Clustering Guide](https://github.com/coreos/etcd/blob/master/Documentation/op-guide/clustering.md)
* [etcdctl config](https://github.com/coreos/etcd/tree/master/etcdctl)
* [cloudflare/cfssl](https://github.com/cloudflare/cfssl)