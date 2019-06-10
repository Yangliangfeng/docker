### docker操作常见指令

* 查看docker网络
docker network ls

* 查看各个容器的IP
docker network inspect bridge

* 删除所有未使用的容器网络
docker network prune

* 删除自定义容器网络
docker network rm + 容器名

* 创建自定义容器网络
```
语法：docker network create [选项] 网络名
选项：
   -d <drive_type>   指定网络设备类型，默认是"bridge"
示例：
	docker network create -d bridge my-network
```
* 停止所有的容器
docker stop $(docker ps -aq)

* 删除所有的容器
docker rm $(docker ps -aq)

* 删除所有的镜像
docker rmi $(docker images -q)

* 复制容器的文件
docker cp myweb:/usr/local/apache2/conf/httpd.conf /home/shenyi/conf/

* docker-compose编排命令
```
1. 启动容器
docker-compose up -d

2.停止所有容器
docker-compose stop

3.停止其中某个容器
docker-compose stop xxx

4.取别名启动容器
docker-compose -p php up -d

5.取别名停止容器
docker-compose -p php stop

6.删除所有的容器
docker-compose rm

7. 检查某个容器的状态
docker inspect web(web是容器名)
```
* 以文件形式存储镜像以及安装
```
1. 以为渐渐形式保存镜像
docker save -o centos-nginx-image.tar centos:nginx

2. 解压文件形式的镜像
docker load -i centos-nginx-image.tar
```
