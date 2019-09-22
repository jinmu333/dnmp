# DNMP

(Docker + Nginx + MySQL + PHP7/5) **LNMP一键安装程序**

## 1.目录结构

```conf
/
├── conf                        配置文件目录
│   ├── conf.d                  Nginx用户站点配置目录
│   ├── nginx.conf              Nginx默认配置文件
│   ├── mysql.cnf               MySQL用户配置文件
│   ├── php-fpm.conf            PHP-FPM配置文件（部分会覆盖php.ini配置）
│   └── php.ini                 PHP默认配置文件
├── Dockerfile                  PHP镜像构建文件
├── extensions                  PHP扩展源码包
├── log                         日志目录
├── mysql                       MySQL数据目录
├── docker-compose-sample.yml   Docker 服务配置示例文件
├── env.smaple                  环境配置示例文件
└── www                         PHP代码目录
```

结构示意图：

![Demo Image](./dnmp.png)

## 2.快速使用

本地安装`git`、`docker`和`docker-compose`(**需要1.7.0及以上版本**)

```bash
git clone https://github.com/yeszao/dnmp.git
# docker 权限
# sudo gpasswd -a ${USER} docker
cd dnmp
docker build -f Dockerfile -t dnmp_php72:1.0 .
docker-compose -f docker-compose.yml up
```

> 访问在浏览器中访问：[http://localhost](http://localhost)，PHP代码：`./www/localhost/index.php`文件。

## 2.PHP和扩展

### 2.1 切换Nginx使用的PHP版本

默认情况下，我们同时创建 **PHP5.6和PHP7.2** 2个PHP版本的容器，

切换PHP仅需修改相应站点 Nginx 配置的`fastcgi_pass`选项，

例如，示例的 [http://localhost](http://localhost) 用的是PHP7.2，Nginx 配置：

```conf
fastcgi_pass   php72:9000;
# 要改用PHP5.6，修改为：
fastcgi_pass   php56:9000;
# 再重启 Nginx 生效。
docker exec -it nginx nginx -s reload
```

### 2.2 安装PHP扩展

PHP的很多功能都是通过扩展实现，而安装扩展是一个略费时间的过程，
所以，除PHP内置扩展外，在`env.sample`文件中我们仅默认安装少量扩展，
如果要安装更多扩展，请打开你的`.env`文件修改如下的PHP配置，
增加需要的PHP扩展：

```bash
PHP72_EXTENSIONS=pdo_mysql,opcache,redis       # PHP 7.2要安装的扩展列表，英文逗号隔开
PHP56_EXTENSIONS=opcache,redis                 # PHP 5.6要安装的扩展列表，英文逗号隔开
```

然后重新build PHP镜像。

```bash
docker build -f Dockerfile -t dnmp_php72:1.0 .
docker-compose -f docker-compose.yml up --build
```

可用的扩展请看同文件的`PHP extensions`注释块说明。

## 5.使用Log

Log文件生成的位置依赖于conf下各log配置的值。

### 5.1 Nginx日志

Nginx日志是我们用得最多的日志，所以我们单独放在根目录`log`下。

`log`会目录映射Nginx容器的`/var/log/nginx`目录，所以在Nginx配置文件中，需要输出log的位置，我们需要配置到`/var/log/nginx`目录，如：

```conf
error_log  /var/log/nginx/nginx.localhost.error.log  warn;
```

### 5.2 PHP-FPM日志

大部分情况下，PHP-FPM的日志都会输出到Nginx的日志中，所以不需要额外配置。

另外，建议直接在PHP中打开错误日志：

```php
error_reporting(E_ALL);
ini_set('error_reporting', 'on');
ini_set('display_errors', 'on');
```

如果确实需要，可按一下步骤开启（在容器中）。

进入容器，创建日志文件并修改权限：

```bash
docker exec -it dnmp_php_1 /bin/bash
mkdir /var/log/php
cd /var/log/php
touch php-fpm.error.log
chmod a+w php-fpm.error.log
```

主机上打开并修改PHP-FPM的配置文件`conf/php-fpm.conf`，找到如下一行，删除注释，并改值为：

```conf
php_admin_value[error_log] = /var/log/php/php-fpm.error.log
```

重启PHP-FPM容器

### 5.3 MySQL日志

因为MySQL容器中的MySQL使用的是`mysql`用户启动，它无法自行在`/var/log`下的增加日志文件。所以，我们把MySQL的日志放在与data一样的目录，即项目的`mysql`目录下，对应容器中的`/var/lib/mysql/`目录。

```bash
slow-query-log-file     = /var/lib/mysql/mysql.slow.log
log-error               = /var/lib/mysql/mysql.error.log
```

以上是mysql.conf中的日志文件的配置。

### 7.1 phpMyAdmin

phpMyAdmin容器映射到主机的端口地址是：`8080`，所以主机上访问phpMyAdmin的地址是：
[http://localhost:8080](http://localhost:8080)

MySQL连接信息：

- host：(本项目的MySQL容器网络)
- port：`3306`
- username：（手动在phpmyadmin界面输入）
- password：（手动在phpmyadmin界面输入）

## 8.在正式环境中安全使用

要在正式环境中使用，请：

1. 在php.ini中关闭XDebug调试
2. 增强MySQL数据库访问的安全策略
3. 增强redis访问的安全策略

## 9.常见问题

### 9.1 如何在PHP代码中使用curl

参考这个issue：[https://github.com/yeszao/dnmp/issues/91](https://github.com/yeszao/dnmp/issues/91)

### 9.2 Docker使用cron定时任务

[Docker使用cron定时任务](https://www.awaimai.com/2615.html)

### 9.3 Docker容器时间

容器时间在.env文件中配置`TZ`变量，所有支持的时区请看[时区列表·维基百科](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)或者[PHP所支持的时区列表·PHP官网](https://www.php.net/manual/zh/timezones.php)。

### 9.4 Mysql5.7版本与mysql8.0版本切换问题

mysql8.0采用了default_authentication_plugin=mysql_native_password加密方式,8.0以前版本是用caching_sha2_password,所以如果用5.7版本的,切换到conf目录,
修改mysql.conf配置文件

```sh
[mysqld]
default_authentication_plugin=mysql_native_password
```

然后在 mysql 下执行以下命令来修改密码：

```bash
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '新密码';
```
