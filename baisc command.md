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
 

   
 












