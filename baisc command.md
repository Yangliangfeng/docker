# 容器的基本操作命令

#### 1.docker run 命令操作的几个参数
* -i: 打开stdin，用于和容器进行交互，通常与 -t 同时使用
* -t: 容器创建虚拟终端，我们就可以登录终端了通常与 -i 同时使用
* --name xxoo: 为容器指定一个名称为xxoo

```
docker run -i -t --name myos1 centos
```
这时你会发现，突然进入了一个新的终端.请按ctrl+D 退出 ,这时我们会发现容器停止运行了

#### 2.再次启动容器
```
docker ps -a     #查看当前所有容器
docker start + ID(或容器名字)   #启动容器
docker attach + ID(或容器名字)  #进入容器,即与容器进行交互
docker start + ID(或容器名字) -a -i  #进入容器,即与容器进行交互
  * -a:打开容器的输出流
  * -i:打开容器的输入流
```

#### 3.在运行中的容器中执行命令
```
docker exec myos1 cat test   #不需要进入容器，就可以立即输出cat命令执行的结果
```

# 简单的利用Dockerfile创建一个镜像

  Dockerfile是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。官方文档：
  https://docs.docker.com/engine/reference/builder/#usage

* 1.新建一个空文件夹(build)，cd 进入build文件夹，在文件夹下新建Dockerfile文件，并插入如下内容：
```
FROM  centos:latest        #下载镜像的REPOSITORY:TAG,以centos:latest为镜像
RUN yum -y install httpd   #安装httpd
RUN  systemctl enable httpd.service  #开机启动httpd
EXPOSE 80                   #暴露80端口
```
* 2.读取Dockerfile创建镜像
```
docker build -t centos:httpd .
  * -t:表示命名镜像的标签是httpd
  * .:"."表示的是当前文件夹
 
 指定Dockerfile
 docker build -f /path/to/a/Dockerfile .
```
* 3.启动容器
```
docker run  -d -p 8080:80 --name myhttpd centos:httpd
#容器启动不了
那么，删除容器
docker rm + ID
交互式方式启动容器
docker run -i -t  -p 8080:80 --name myhttpd centos:httpd
#容器启动
```
* 4.解决错误
```
docker exec -i -t myhttpd /bin/bash    #进入容器
systemctl start httpd        #启动httpd，但是，出现了如下错误:
Failed to get D-Bus connection: Operation not permitted
停止容器并删除:
docker stop myhttpd
docker rm myhttpd
解决方法:
docker run --privileged -d -p 8080:80 --name myhttpd centos:httpd /usr/sbin/init
  * --privileged:给容器加特权,否则交互式方式进入容器无法操作一些譬如修改内核、修改系统参数、甚至启动服务等
  * /usr/sbin/init:在启动容器时执行此命令
 进入容器:
 docker exec -i -t myhttpd /bin/bash
 开启httpd:
 systemctl start httpd
```
* 5.终极Dockerfile配置文件内容
```
FROM centos:latest
RUN yum -y install httpd   
RUN  systemctl enable httpd.service 
CMD /usr/sbin/init   
```
* 6.网站文件塞入容器

  通过yum安装的apache，默认的配置文件在/etc/httpd/conf/httpd.conf，根据默认的配置文件，网站在/var/www/html
 ```
  docker run --privileged -d -p 8080:80 --name  myhttpd -v /home/yang/myweb:/var/www/html centos:httpd /usr/sbin/init
  * 在容器中设置了一个挂载点/var/www/html 
  * 将主机上的 /home/shenyi/myweb目录中的映射到/var/www/html下，我们容器中操作该目录或在主机中操作，两者均是实时同步的
  * 访问网站的目录，其实就是访问/home/yang/myweb目录
```
* 7.把JDK加入镜像
 
 * 创建一个空文件夹，叫做build-jdk,然后把你的现成JDK文件拷贝到此目录,我们开始写Dockerfile
 ```
 FROM centos:httpd    #注意这里，我们使用前面做好的httpd的镜像
COPY jdk1.8 /usr/local/jdk1.8.0_121   #把当前文件夹下的jdk文件拷贝到/usr/local/jdk1.8.0_121目录下
ENV JAVA_HOME=/usr/local/jdk1.8.0_121   #加入环境变量
ENV PATH $JAVA_HOME/bin:$PATH           
ENV CLASSPATH .:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
CMD /usr/sbin/init    #容器启动后执行此命令
```
 * 生成镜像
 ```
 docker build -t centos:jdk   #生成标签为jdk的镜像
 ```
 * 启动容器
```
docker run --privileged -d -p 8080:80 --name  myjdk -v /home/shenyi/myweb:/var/www/html centos:jdk
```
* 进入容器
```
docker exec -it myjdk /bin/bash
```
* 8.docker配置远程连接

  build是Docker守护进程运行的，不是CLI(客户端命令)，运行时会把整个的context（上下文）发送给Daemon守护进程
  
  两个重要概念:“Docker客户端和Docker守护进程
  
  Docker 服务端提供了一系列REST API（Docker Remote API),当我们敲入docker命令时实际上是通过API和Docker服务端进行交互的
  
  官网提供的三种连接方式:
   * unix:///var/run/docker/sock（默认连接方式）
   * tcp://host:port
   * fd://socketfd
   
   查看连接方式的命令:
   `
    ps -ef | grep docker
    `
    
   * 配置远程访问
    
 ```
    sudo vi /usr/lib/systemd/system/docker.service 
    ExecStart=/usr/bin/dockerd    #注释这一行
    ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock     #加入这一行
    重启Docker:
    systemctl daemon-reload
    systemctl restart docker
    查看结果:
    ps -ef |grep docker
```
# Docker管理工具portainer

 * 拉取镜像
 
 `
 docker pull portainer/portainer
`
 * 运行容器
 
 `
 docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock -v /opt/portainer:/data --name mydocker portainer/portainer
`
 * 访问
 
 `
 http://192.168.1.30:9000
 `
 * 基于portainer的mysql安装
   在通过portainer安装时，启动mysql出现了问题。要根据阿里云镜像源平台mysql官方的给出的docker启动的命令进行启动。
   
   `
   docker run --name mysql -p 3307:3306 -e MYSQL_ROOT_PASSWORD=123 -d mysql
  `
  
# 关于docker的几个案例 
  
### 1.手工搭建Centos+Nginx容器、commit提交、容器主机文件互拷
  
 * 搭建nginx 镜像
  
```
  docker run -it --privileged --name tmp centos /usr/sbin/init      #创建临时容器，此时，界面处于停止状态，重新打开终端
  docker exec -it tmp /bin/bash      #进入容器
  rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
  yum install nginx -y    #下载并安装nginx
  systemctl enable nginx   #设置开机启动
  systemctl start nginx    #启动nginx
```
  
 * 提交修改好的镜像并产生新的镜像
  `
  docker commit -c 'CMD ["/usr/sbin/init"] ' -c "EXPOSE 80" tmp centos:nginx
  `
  
 * 拷贝一份配置文件
  
  `
  docker cp tmp:/etc/nginx/nginx.conf /home/yang/nginx/conf/
  `
  
 * 启动nginx镜像
  
  `
  docker run --name mynginx --privileged -p 9090:80 -v /home/yang/nginx/conf/nginx.conf:/etc/nginx/nginx.conf -d centos:nginx
  `
  
  ### 2.Docker Compose入门：启动多个web容器
  
  准备工作:创建两个文件夹分别是web1、web2，里面分别新建index.html，并写入不同的内容
  
  * 安装
  
  ```
  sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose       #在/usr/local/bin/下出现 docker-compose目录
  sudo chmod +x /usr/local/bin/docker-compose      #赋予权限
  docker-compose --version      #查看版本
  ```
  
  * 配置文件 docker-compose.yml
  
  ```
  1.创建一个空文件夹，composetest
  2.创建一个docker-compose.yml 文件
  3.写入如下内容
  services: 
  web1: 
    container_name: web1
    image:"centos:jdk"
    ports: 
      - "8080:80"
    privileged: true
    volumes: 
      - "/home/shenyi/nginx/web1/:/var/www/html/"
  web2: 
    container_name: web2
    image: "centos:jdk"
    ports: 
      - "8081:80"
    privileged: true
    volumes: 
      - "/home/shenyi/nginx/web2/:/var/www/html/"
version: "3"
   4.相关yaml检查的在线工具 检查语法
   http://www.yamllint.com/
```
* 创建并启动
 
 `
 docker-compose up -d
`
### 3.Docker network入门、简配容器网络、容器间互相访问

 * 创建网络并设置子网地址
 
 ```
 docker-compose stop    #停止运行的容器
 docker network create -d bridge --subnet=192.128.0.0/16 mynginx  #创建网络并设置子网地址
 docker network ls      #查看创建的网络
 docker network inspect mynginx     #查看创建的mynginx网络配置
 ```
 
 * 修改docker-compose的配置文件
 
 ```
services:
  web1:
    container_name: web1
    image: "centos:jdk"
    ports:
      - "8080:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/web1/:/var/www/html/"
    networks:
      - "mynginx-net"
  web2:
    container_name: web2
    image: "centos:jdk"
    ports:
      - "8081:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/web2/:/var/www/html/"
    networks:
      - "mynginx-net"
  nginx:
    container_name: mynginx
    image: "centos:nginx"
    ports:
      - "9000:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/conf/nginx.conf:/etc/nginx/nginx.conf"
    networks:
      - "mynginx-net"
networks:
  mynginx-net:
    external:
      name: "mynginx"
version: "3"
```
 * 启动容器
 
 `
 docker-compose start
 `
 
 * 进入容器查看
 
 ```
 cat /etc/hosts         #查看网络配置
 ```
 
 ### 4.Docker compose创建网络、指定容器IP、启动简单nginx负载均衡
 
  * 自动创建网络
```
  docker-compose stop      #停止运行的容器
  docker rm +ID            #删除之前创建的容器
  编辑配置文件:
services:
  web1:
    container_name: web1
    image: "centos:jdk"
    ports:
      - "8080:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/web1/:/var/www/html/"
    networks:
      app_net:
       ipv4_address: ${web1_addr}
  web2:
    container_name: web2
    image: "centos:jdk"
    ports:
      - "8081:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/web2/:/var/www/html/"
    networks:
      app_net:
       ipv4_address: 192.158.0.6
  nginx:
    container_name: mynginx
    image: "centos:nginx"
    ports:
      - "9000:80"
    privileged: true
    volumes:
      - "/home/yang/nginx/conf/nginx.conf:/etc/nginx/nginx.conf"
    networks:
      app_net:
       ipv4_address: 192.158.0.3
networks:
  app_net:
    driver: bridge
    ipam:
     config:
      - subnet: 192.158.0.0/16
version: "3"
```
 * 创建并启动
 
```
 docker-compose up -d   
```
 * 环境变量
```
 https://docs.docker.com/compose/environment-variables/#the-env-file    #参考此文档
 在.yml的配置目录下，新建“.env”文件，在文件中写入需要替换的变量的值，在.yml文件，用${变量名}进行替换
 ```
 * nginx的负载均衡的基本配置
 
 ```
 upstream mydocker {
             server 192.156.0.6;
             server 192.156.0.3;
         }
      server {
        listen       80;
        server_name  mydocker;
        location / {
             proxy_set_header Host $host;
             proxy_set_header X-Real-IP $remote_addr;
             proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
             proxy_buffering off;
             proxy_pass http://mydocker;     
        }
     }
```


 
 

 


 

   
 












