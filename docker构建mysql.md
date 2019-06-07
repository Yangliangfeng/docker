### docker构建mysql运行环境

* 下载mysql镜像
https://hub.docker.com/_/mysql/
docker pull mysql:5.7

* mysql的简单运行
```
1.运行
docker run --name mysql --rm \
 -p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

-e 设置环境变量
MYSQL_ROOT_PASSWORD 是root的密码

2. 登陆到容器的mysql客户端
docker exec -it mysql mysql -uroot -p123456

3. mysql容器中的配置文件
1) 配置文件在
/etc/mysql/my.cnf

2) 数据目录在
/var/lib/mysql

4. 在本机下新建mysql文件夹，里面包含两个子文件夹
1) conf
2) data

5. 特别说明
官方的my.cnf 加载了conf.d里面的文件。我们只要创建个带有关键配置部分的配置文件，映射 conf.d文件夹即可

于是我们在conf文件下，随便创建个文件。譬如 abc.cnf (后缀必须是cnf)

6. abc.conf配置文件的内容
[mysqld]
server-id = 1 #服务Id唯一
port = 3306
log-error	= /var/log/mysql/error.log
#只能用IP地址
skip_name_resolve 
#数据库默认字符集
character-set-server = utf8mb4
#数据库字符集对应一些排序等规则 
collation-server = utf8mb4_general_ci
#设置client连接mysql时的字符集,防止乱码
init_connect='SET NAMES utf8mb4'
#最大连接数
max_connections = 300

7. 启动mysql服务器容器
docker run --name mysql --rm \
 -v /home/shenyi/mysql/conf:/etc/mysql/conf.d \
-v /home/shenyi/mysql/data:/var/lib/mysql \
-p 3306:3306 \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7

```
* 导入数据到容器中
```
1. 拷贝本机文件到容器中
docker cp /home/yang/mysql/test.sql mysql:/tmp

2. 进入mysql容器的客户端
docker exec -it mysql mysql -uroot -p123456

3. 导入数据
source /tmp/test.sql
```
* 加入镜像源的MySQL客户端的运行
```
1. 常用的镜像
默认镜像源可能比较慢，常用的有：
中科大镜像源：http://mirrors.ustc.edu.cn/alpine/
阿里云镜像源：http://mirrors.aliyun.com/alpine/

2. 利用Dockerfil修改镜像源
FROM alpine:3.9
RUN echo http://mirrors.ustc.edu.cn/alpine/v3.9/main > /etc/apk/repositories
RUN echo http://mirrors.ustc.edu.cn/alpine/v3.9/community >> /etc/apk/repositories
RUN apk update && apk upgrade
RUN apk add  mysql-client
ENTRYPOINT ["mysql"]

3. 利用Dockerfile文件构建镜像
docker build -t mytool:1.0 /home/yang/mydocker/Dockerfile

4. 用自建的镜像以交互的方式运行mysql客户端
docker run -it --name tmp --rm mytool:1.0 -h 182.92.225.115 -uroot -p123456

5. msyql常用的备份命令
mysqldump -h182.92.225.115 -uroot -p123456 test > test.sql  //备份主机182.92.225.115 上test数据库

6. mytool:1.1 版本Dockerfile
FROM mytool:1.0
RUN mkdir data
ENTRYPOINT mysqldump -h192.168.222.135 -uroot -p123456 test > /data/test.sql

7. 构造mytool:1.1版本的镜像
docker build -t mytool:1.1 /home/yang/mydocker/Dockerfile

8. mytool:1.2 版本的Dockerfile
FROM mytool:1.1
ENV mysql_user root
ENV mysql_pass 123456
ENV mysql_host  192.168.222.135
ENV mysql_db test
ENTRYPOINT mysqldump -h$mysql_host  -u$mysql_user -p$mysql_pass $mysql_db > /data/$mysql_db.sql

9. 构建mytool:1.2镜像
docker build -t mytool:1.2

10. 运行备份的文件
docker run -it --name bakup --rm \
-v /home/shenyi/mysqlbak:/data \
-e mysql_pass=123123 \
-e mysql_host=192.168.222.1 \
-e mysql_db=shenyi \
-e mysql_user=shenyi \
mytool:1.2
```
* 构建自动备份数据的容器
```
1. 编辑数据库备份的脚本bakup.sh
#!/bin/sh
if [ ! -d "/data" ]; then
  mkdir /data
fi
mysqldump -h$mysql_host  -u$mysql_user -p$mysql_pass $mysql_db > /data/$mysql_db-$(date +%Y%m%d_%H%M%S).sql

2. 构建数据库自动备份的Dockerfile
FROM mytool:1.1
ENV mysql_user root
ENV mysql_pass 123456
ENV mysql_host  192.168.222.135
ENV mysql_db test
COPY ./bakup.sh /
RUN chmod +x bakup.sh
ENV cron_conf  "*       *       *       *       *  "
RUN echo "$cron_conf /bakup.sh" >>  /var/spool/cron/crontabs/root  //定时任务计划
ENTRYPOINT ["crond","-f"] //crond 以f的方式运行

3. 构建新镜像
docker build -t mysqlbackup:1.0 .

4. 运行备份数据库的容器

docker run -d --name bakup  \
-v /home/yang/mysqlbak:/data \
-e mysql_pass=123123 \
-e mysql_host=192.168.222.1 \
-e mysql_db=yang \
-e mysql_user=yang \
mysqlbakup:1.0

```
