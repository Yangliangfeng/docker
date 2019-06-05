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
