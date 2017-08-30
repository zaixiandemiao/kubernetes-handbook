## 安装Docker
使用[阿里云加速器](https://cr.console.aliyun.com)安装  
1. 确保本机没有旧版本的docker  
`sudo yum remove docker docker-engine docker.io`
1. 安装docker-engine, docker-ce  
`curl -sSL http://acs-public-mirror.oss-cn-hangzhou.aliyuncs.com/docker-engine/internet | sh -`  
3. 添加镜像
```
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://8difvy5w.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker
```