### docker配置samba共享:windows开发,在linux中运行(无密码模式)

* 拉取镜像
https://hub.docker.com/r/dperson/samba/

* 开放端口
sudo iptables -I INPUT -p tcp --dport 139 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 445 -j ACCEPT

sudo iptables -I INPUT -p udp --dport 137 -j ACCEPT
sudo iptables -I INPUT -p udp --dport 138 -j ACCEPT

* 运行容器
```
docker run -it -p 139:139 -p 445:445  --name smb -d --rm  \
 -v /home/yang/html/web:/mount \      //web是网站目录 mount是挂载到容器的mount目录下
 dperson/samba \
 -u "yang;123" \  //用户名和密码
 -s "yang;/mount/;yes;no;yes;all;all;all" \  //权限
 -w "WORKGROUP" \   //window下的工作组
 -g "force user= yang" \   //配置在window下修改文件的权限
 -g "guest account= yang" 

```
