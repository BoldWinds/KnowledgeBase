## 拉取镜像运行容器

设置好superuser的用户名、密码、初始数据库和postgresql的访问端口。
安装完成后进入容器终端，输入命令连接数据库：
```bash
psql -U username
\l
```
此时显示所有数据库则说明容器正常运行。

## 创建新的数据库

在这里以创建nextcloud要用的数据库为例。
首先创建新的用户：`CREATE USER nextcloud WITH PASSWORD 'nextcloud';`
之后创建该用户所需要的数据库：`CREATE DATABASE nextcloud;`
最后更改数据库所有者：`ALTER DATABASE nextcloud OWNER TO nextcloud;`

