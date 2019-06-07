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
