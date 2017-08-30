# Harbor安装

## 前言

Harbor的安装，[官方](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)提供了比较详尽的手册，基本可以参考完成整个安装过程。本文档采用离线安装方法，使用http访问私有仓库，后期探索证书认证方法。安装Harbor前需要的环境：
* Python >= 2.7
* Docker >= 1.10，推荐使用[阿里云加速器](https://cr.console.aliyun.com)安装 
* Docker Compose >= 1.60(*Harbor使用docker-compose以容器的形式运行*)，直接下载[二进制文件](https://github.com/docker/compose/releases/download/1.16.0-rc1/docker-compose-Linux-x86_64)，加入path即可

## 1. 下载离线安装包
到Harbor的[发布页面](https://github.com/vmware/harbor/releases)下载，尽量下载没有rc的后缀的版本，相对比较稳定一些，我下载的是[1.1.2](https://github.com/vmware/harbor/releases/download/v1.1.2/harbor-offline-installer-v1.1.2.tgz)版本的离线包,下载完成后将压缩包解压  
```
$ tar xvf harbor-offline-installer-<version>.tgz
```  

## 2. 配置harbor.cfg

Harbor的安装配置都在harbor.cfg文件中，其中的属性分为required, optional, 详细信息可以参考[官方配置说明](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)，因为只使用Http访问私有仓库，这里需要改动的参数很少，修改情况如下：
* **hostname** 访问Harbor的地址（UI和registry service），可以被配置为ip地址或者域名，不可以配置成`localhost`或者`127.0.0.1`  
* **ui_url_protocol:** (http 或https，默认为http)，此处使用http，https的配置参考[https配置](https://github.com/vmware/harbor/blob/master/docs/configure_https.md)  
* **harbor_admin_password:** 第一次登陆时，admin的密码，登陆后必须修改，默认为Harbor12345,Harbor的密码必须包含大写，小写，数字三种字符
* **self_registration:** (on 或 off.默认为on), 是否支持自主注册，如果设为Off，则用户只能被admin用户创建  
* **token_expiration:** token的时效，单位为分钟，默认为30分钟
  
## 3. installation & starting 

执行脚本文件进行安装  
```
./install.sh
```  
如果执行顺利的话，则可以访问http://hostname 进行访问

## 上传镜像到私有仓库
若要上传镜像的电脑可以访问外网，则可以使用docker先将镜像pull到本地，然后push到私有仓库上。
```
$ docker pull nginx
$ docker login hostname
$ docker tag nginx hostname/library/nginx
$ docker push hostname/library/nginx
```
上面的四条命令将DockerHub上的nginx先Pull到本地，然后打标签后，上传到私有仓库中， hostname可以是Harbor所在的ip地址，library是一个所有用户都可以访问到的project名称。admin用户可以push，普通用户只可以pull  
如果`docker login`出错，则原因是，docker配置中，默认需要加密访问，可以到docker的配置文件中添加`--insecure-registry hostname`, 配置文件路径在`/etc/sysconfig/docker`,然后执行下面的命令  
``` 
$ docker -D login hostname
$ docker tag
$ docker push
```  

## 管理Harbor 
停止Harbor  
```
docker-compose stop
```  
运行Harbor  
```
docker-compose start
```  
修改harbor.cfg, 重启Harbor
```
$ sudo docker-compose down -v
$ vim harbor.cfg
$ sudo prepare
$ sudo docker-compose up -d
```