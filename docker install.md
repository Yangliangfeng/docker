CentOS7的docker安装
---------------------

1. 安装一些基本依赖软件
```
yum install -y yum-utils device-mapper-persistent-data lvm2
```
2. 设置即将安装的是稳定版仓库
```
  yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

3. 这一步是可选的(edge月更新仓库, Edge gives you new features every month)
```
yum-config-manager --enable docker-ce-edge
```

4. 默认安装
```
yum install docker-ce -y
```

5. 启动(并开机启动)
```
systemctl start docker
systemctl enable docker
```

6. 加入dock用户组
  docker安装时默认创建了docker用户组，将普通用户加入docker用户组就可以不使用sudo来操作docker
```
sudo usermod -aG docker +"登陆的普通用户名"  并重新登陆
```

7. 阿里云的镜像源
  https://dev.aliyun.com/search.html
  
8. 执行以下命令
 ```
 sudo mkdir -p /etc/docker     #创建一个文件夹叫做docker
```
```
sudo tee /etc/docker/daemon.json <<-'EOF'   #利用tee 命令把下面的配置写入 daemon.json
{
  "registry-mirrors": ["https://14lbe7a4.mirror.aliyuncs.com"]  #这里要改成自己阿里云镜像加速器的地址
}
EOF
```
```
sudo systemctl daemon-reload  #重载所有修改过的配置文件,扫描新的或有变动的单元
sudo systemctl restart docker  # 重启docker
```

9. 下载一个镜像(本文是下载阿里云的镜像:https://dev.aliyun.com/search.html)   

    镜像地址(选取你要下载的镜像源的地址，本文是下载的一个PHP的镜像源):
```
  docker pull registry.cn-hangzhou.aliyuncs.com/lxepoo/apache-php5
```

10. 启动容器
```
docker run -d -p 8080:80 名称或者ID
```
* run :把我们的镜像放入容器中（只在第一次运行)
* -d 启动容器后台运行，并返回ID；
* -p 把容器的80端口映射到宿主机的8080

11. 操作命令
```
docker images   #查看镜像
docker ps -a    #列出所有运行中的容器
docker start + ID  #启动ID的镜像
docker stop + ID   #停止ID的镜像
```
12. 启动nginx+php环境
```
1.创建网络
docker network create --driver=bridge --subnet=192.138.0.0/16 mynginx 

2.先要启动fpm
docker run -d --rm --name fpm --network mynginx --ip 192.138.0.2 \
-v /home/yang/php:/php php:7.3-fpm-alpine3.9 

3.再启动nginx
docker run -d --rm --name nginx --network mynginx -v /home/yang/php:/usr/share/nginx/html 
-v /home/yang/conf/nginx.conf:/etc/nginx/nginx.conf -p 80:80 nginx:1.16.0-alpine
```
13. docker可视化监控插件
```
https://github.com/google/cadvisor
```






   
