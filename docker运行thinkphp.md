### docker中运行thinkphp

* 网站目录

/home/yang/html/web      //存放代码的目录

/home/yang/html/compose  //存放docker-compose文件目录

* docker-compose文件
```
version: "3"
services:
  fpm:
   image: php:7.3-fpm-alpine3.9
   container_name: fpm
   volumes:
      - /home/yang/html/web:/php
   networks:
      mywebnet:
       ipv4_address: 192.138.0.2
  httpd:
   image: httpd:2.4-alpine
   container_name: httpd
   ports:
      - 8081:80
   volumes:
      - /home/yang/html/web/public:/usr/local/apache2/htdocs/
      - /home/yang/conf/httpd.conf:/usr/local/apache2/conf/httpd.conf
   networks:
      mywebnet:
       ipv4_address: 192.138.0.3
networks:
  mywebnet:
     driver: bridge
     ipam:
       config:
         - subnet: 192.138.0.0/16

```
* url重写
```
1. 开启apache重写模块
LoadModule rewrite_module modules/mod_rewrite.so

2. 在网站配置相关节点加入  
<VirtualHost *:80>
    ServerAdmin shenyi@com.cn
    DocumentRoot "/usr/local/apache2/htdocs"
    ServerName localhost
    <Directory "/usr/local/apache2/htdocs">
     Options None
     AllowOverride all
     Require all granted
    </Directory>
    ProxyRequests Off
    ProxyPassMatch ^/(.*\.php) fcgi://192.168.0.2:9000/php/public/$1
</VirtualHost>

3. 重新构建容器
docker-compose restart
```
* 在容器中配置mysql
```
1. mysql加入docker-compose配置文件中
mysql:
   image: mysql:5.7
   container_name: mysqld
   ports:
     - 3306:3306
   volumes:
     - /home/yang/mysql/conf:/etc/mysql/conf.d
     - /home/shenyi/mysql/data:/var/lib/mysql
   environment:
     - MYSQL_ROOT_PASSWORD=123456
   networks:
     - mywebne
2. 启动mysql
docker-compose up -d mysql

3. 获取容器中的mysql的IP地址
docker network inspect compose_mywebnet

4.配置tp框架中的数据库配置文件
```
* 在容器中安装php核心扩展
```
1. 登入容器查看php配置模块
docker exec -it fpm sh

2. 查看当前容器的镜像源
cat /etc/apk/repositories

3. 查看当前 alpine的版本
cat /etc/issue

4. 设置阿里云的镜像源
echo http://mirrors.ustc.edu.cn/alpine/v3.9/main > /etc/apk/repositories && \
echo http://mirrors.ustc.edu.cn/alpine/v3.9/community >> /etc/apk/repositories

5. 镜像源的更新
apk update && apk upgrade

6. 安装php扩展
docker-php-ext-install pdo_mysql

7. 重启fpm
docker-compose restart fpm

注释：刚才是在容器中单独安装扩展；如果容器删除了，下次重新启动新的容器，还是没有新安装的扩展

8. 在Dockerfile中配置扩展
在docker-compose.yml的配置文件目录下，新建build文件夹，在build文件夹下，新建名为phpfpm的Dockerfile文件
FROM php:7.2.0-fpm-alpine3.6
RUN echo http://mirrors.ustc.edu.cn/alpine/v3.6/main > /etc/apk/repositories && \
    echo http://mirrors.ustc.edu.cn/alpine/v3.6/community >> /etc/apk/repositories
RUN apk update && apk upgrade
RUN docker-php-ext-install pdo_mysql

9. 在docker-compose.yml配置文件中加入build
fpm:
   build:
    context: ./build
    dockerfile: phpfpm

   image: php:7.2.0-fpm-alpine3.6
   container_name: fpm
   volumes:
      - /home/yang/html/web:/php
   networks:

10. 生成新的php:fpm镜像
docker-compose build fpm

11. 删除并重新启动fpm容器
docker-compose stop fpm

docker-compose rm fpm

docker-compose up -d fpm

```
* php安装redis扩展以及安装redis容器
```
1. 拉取基于apline的redis
docker pull redis:apline

2. redis配置文件
下载url：https://redis.io/topics/config 
在网站的目录下新建conf文件夹，进入conf目录：
wget https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf
修改如下：
bind 0.0.0.0
dir /data

3. 在docker-compose.yml中加入redis服务
redis:
    image: redis:alpine
    container_name: redis
    ports:
     - 6379:6379
    volumes:
     - /home/yang/html/conf/redis.conf:/usr/local/etc/redis/redis.conf
     - /home/yang/html/redisdata:/data
    networks:
      mywebnet:
       ipv4_address: 192.138.0.10

4. 启动redis服务器
docker-compose up -d redis

5. fpm安装redis扩展
在build目录中创建phpredis的Dockerfile
FROM php:7.2.0-fpm-alpine3.6
RUN apk add autoconf gcc g++ make
RUN pecl install redis-4.0.1 \
    && docker-php-ext-enable redis

6. 在docker-compose.yml配置文件加入扩展
fpm:
   build:
    context: ./build
    dockerfile: phpredis

7. 生成新的fpm镜像
docker-compose build fpm

8. 删除并重新启动 fpm容器
docker-compose stop fpm
docker-compose rm fpm
docker-compose up -d fpm

```
* 解决容器内tp上传文件没有权限的问题
```
1. 添加权限
本机的代码目录挂载到容器中比如/php目录下，用户和用户组是本机的，而容器中运行的php的默认用户却跟本机的不一致。
容器中运行php的默认用户是www-data，因此，要把www-data用户和用户组的ID必须保持一致。
usermod -u 1000 www-data
groupmod -g 1000 www-data  //但是，usermod在容器中默认不存在

2. 安装usermod
apk update && apk upgrade
apk add shadow
执行 usermod和groupmod命令

3. 重启fpm容器
docker-compose restart fpm
```
* docker的健康检查
```
1. 健康检查的docker-compose配置文件
healthcheck:(健康检查)
  test: ["CMD", "curl", "-s", "-f", "http://localhost:80"] //-f fail表示出现的故障   -s slient 表示在当前的cmd不是输出
  interval: 5s
  timeout: 5s
  retries: 3
 
 2. 命令行下的健康检查
--health-cmd:检查的命令
--health-interval：两次健康检查的间隔
--health-timeout：健康检查命令运行超时时间，超过代表失败
--health-retries：当连续失败指定次数后，则将容器状态视为 unhealthy
--health-start-period:容器启动的初始化时间，此时健康检查失效不会计入次数

我们把命令改一下：
docker run  --name web1 -d -p 8080:80 --privileged -v /home/shenyi/nginx/web1/:/var/www/html/ \
--health-cmd="curl --silent --fail http://localhost:80/ || exit 1" --health-interval=3s --health-retries=3 \
--health-timeout=5s \
 centos:jdk

```

