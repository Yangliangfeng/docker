### docker 构建扩展

* 安装curl
```
1. cat /etc/apk/repositories 查看其内核

2. 修改repositories
  echo http://mirrors.ustc.edu.cn/alpine/v3.7/main > /etc/apk/repositories && \
  echo http://mirrors.ustc.edu.cn/alpine/v3.7/community >> /etc/apk/repositories

3. apk add curl
```
