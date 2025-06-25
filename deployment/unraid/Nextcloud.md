## 拉取镜像并安装

直接在应用市场中拉取Nextcloud的官方镜像template并安装即可。

## 数据库准备

参见[[PostgerSQL数据库]]

## 初始设置

打开Nextcloud的web界面后，它会提示你创建管理员账号并设置数据目录与数据库。数据目录默认即可，数据库选择PostgreSQL并把配置好的账号、密码、数据库名和数据库访问url输入，即可安装。

## 反向代理

在进行[[反向代理与证书自动化]]中的内容之前，首先要更改nextcloud中的配置文件`/mnt/user/appdata/nextcloud/config/config.php`，修改：
```php
'trusted_domains' =>
array (
0 => '192.168.1.20:8666',
1 => 'nextcloud.boldwinds.top',
),
'trusted_proxies' => ['172.17.0.1'], 
'overwritehost' => 'nextcloud.boldwinds.top',
'overwriteprotocol' => 'https',
'overwritewebroot' => '/',
'overwrite.cli.url' => 'https://nextcloud.boldwinds.top',
```

## 软链接

机器中存储了很多媒体资源，电影电视剧等，将其挂载到Nextcloud的数据目录下，这样可以通过webdav来直接访问。
- 电影资源：`/mnt/user/media/movies`
- 电视剧资源：`/mnt/user/media/series`
- nextcloud数据目录：`/mnt/user/data/`

修改添加两个挂载点：
- /var/www/html/data/boldwinds/files/movies:/mnt/user/media/movies/
- /var/www/html/data/boldwinds/files/series:/mnt/user/media/series/
注意上面的boldwinds为对应的用户名。

挂载完成后，进入nextcloud的容器，执行：
```bash
cd /var/www/html
php occ files:scan --path="boldwinds/files/series"
php occ files:scan --path="boldwinds/files/movies"
```

等待一段时间后再打开nextcloud就会发现电影等资源出现在nextcloud的目录中

## 挂载webdav

直接把nextcloud中提供的webdav的url：https://nextcloud.boldwinds.top/remote.php/dav/files/boldwinds复制到Windows或者Macos的磁盘挂载处，输入nextcloud的用户和密码作为凭据即可使用了。

