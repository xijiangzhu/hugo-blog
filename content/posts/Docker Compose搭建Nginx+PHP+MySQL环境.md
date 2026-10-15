---
title: Docker Compose搭建Nginx+PHP+MySQL环境
date: 2026-10-12 10:00:00
tags:
  - Docker
  - Docker Compose
  - Nginx
  - PHP
  - MySQL
categories:
  - 技术
  - 运维
description: 使用 Docker Compose 编排 Nginx、PHP-FPM 和 MySQL 三个容器,包括镜像选择、Dockerfile、目录结构、compose 配置、启动停止以及 PHP 连接 MySQL 的验证。
---

本文通过一个 demo,使用 Docker Compose 一键编排 **Nginx + PHP-FPM + MySQL** 三个容器,搭建一套常见的 PHP 运行环境,并验证 PHP 能够连接 MySQL。

<!-- more -->

> 版本说明:本文基于较早的版本编写(Compose 1.19.0、PHP 7.2、MySQL 5.7),这些版本都已停止维护。思路和写法仍然适用,实际使用时建议换成较新的镜像版本,文末「使用提示」里有具体说明。

## 一、镜像选择

### 1. Nginx

[nginx 官方镜像](https://hub.docker.com/_/nginx/)使用 `latest` tag。

### 2. PHP

PHP 版本较多,可以在 [PHP 官方镜像](https://hub.docker.com/_/php/)中选一个 tag。本文使用 `7.2-fpm`(作者使用 `latest` 无法正常启动)。

**PHP 的 Dockerfile**

要让 PHP 连接 MySQL,需要在 Docker 官方的 PHP 镜像上额外安装 `mysqli` 扩展,所以用 Dockerfile 构建一个新镜像。

新建 `php-mysqli` 目录:

```bash
mkdir php-mysqli
cd php-mysqli
vi Dockerfile
```

编辑 `Dockerfile`,内容如下:

```dockerfile
FROM php:7.2-fpm
RUN apt-get update \
    && docker-php-ext-install mysqli
```

- `FROM php:7.2-fpm`:表示基于 `php:7.2-fpm` 镜像。
- `RUN` 后面的指令:在构建镜像时执行,用来安装扩展。

这里不需要手动 `build` 镜像,后面使用 `docker-compose` 时会自动构建。

关于 Dockerfile,可以参考 [《Docker — 从入门到实践》——使用 Dockerfile 定制镜像](https://bingohuang.gitbooks.io/docker_practice/content/image/build.html)。

### 3. MySQL

[MySQL 官方镜像](https://hub.docker.com/_/mysql/),本文选择 `5.7` tag。

## 二、安装 Docker Compose

可参考[官方文档](https://docs.docker.com/compose/)。

**1. 下载(本文使用 1.19.0)**

```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.19.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

可以到 [Compose GitHub](https://github.com/docker/compose/releases) 查看所有发布版本。

**2. 设置执行权限**

```bash
sudo chmod +x /usr/local/bin/docker-compose
```

**3. 测试是否安装成功**

```bash
docker-compose --version
```

输出示例:

```text
docker-compose version 1.19.0, build 1719ceb
```

## 三、构建目录结构

新建如下目录结构:

```text
php-nginx-compose/
├── conf.d
│   └── default.conf
├── docker-compose.yml
├── html
│   └── index.php
└── php-mysqli
    └── Dockerfile
```

其中 `php-mysqli` 就是上面创建的目录,直接放在 `php-nginx-compose` 目录中。

### 1. docker-compose.yml

```yaml
version: '3'
services:
    nginx:
        image: nginx:latest
        # 端口映射
        ports:
            - "80:80"
        # 依赖关系:先启动 php
        depends_on:
            - "php"
        # 数据卷
        volumes:
            # 映射主机 ./conf.d 目录到容器 /etc/nginx/conf.d 目录
            - "$PWD/conf.d:/etc/nginx/conf.d"
            - "$PWD/html:/usr/share/nginx/html"
        networks:
            - app_net
        # 容器名称
        container_name: "compose-nginx"
    php:
        build: ./php-mysqli
        # image 指定 build 生成镜像的名称,名字可以自定义
        image: php:7.2-fpm-mysqli
        ports:
            - "9000:9000"
        volumes:
            - "$PWD/html:/usr/share/nginx/html"
        networks:
            - app_net
        container_name: "compose-php"
    mysql:
        image: mysql:5.7
        ports:
            - "3306:3306"
        # 环境变量
        environment:
            # MySQL root 密码,换成你自己的密码
            - MYSQL_ROOT_PASSWORD=你的密码
        networks:
            app_net:
                # 固定子网 IP,必须在下面配置的子网 10.10.*.* 范围内
                ipv4_address: 10.10.10.1
        container_name: "compose-mysql"
networks:
    # 配置 docker network
    app_net:
        driver: bridge
        ipam:
            config:
                # 子网
                - subnet: 10.10.0.0/16
```

关于 Compose file 的配置,可以看[官方详细说明](https://docs.docker.com/compose/compose-file/)。网上还有一篇中文说明[《如何写 docker-compose.yml,Docker compose file 参考文档》](https://deepzz.com/post/docker-compose-file.html)。建议先看官方文档,看不懂再找翻译。

### 2. conf.d/default.conf

```nginx
server {
    listen       80;
    server_name  localhost;
    root   /usr/share/nginx/html;
    index  index.html index.htm index.php;

    location ~ \.php$ {
        fastcgi_pass   php:9000;
        fastcgi_index  index.php;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    }
}
```

`fastcgi_pass php:9000` 中的 `php` 就是 `docker-compose.yml` 里的服务名,同一个网络内的容器可以直接用服务名互相访问。

### 3. html/index.php

```php
<?php
phpinfo();
```

## 四、启动服务

在 `php-nginx-compose` 目录下执行:

```bash
docker-compose up -d
```

输出示例:

```text
Creating compose-php ... done
Creating compose-nginx ... done
Creating compose-mysql ...
Creating compose-nginx ...
```

查看是否启动成功:

```bash
docker ps
```

输出示例:

```text
CONTAINER ID   IMAGE                COMMAND                  CREATED          STATUS          PORTS                    NAMES
aeb1f626516b   nginx:latest         "nginx -g 'daemon of…"   5 seconds ago    Up 3 seconds    0.0.0.0:80->80/tcp       compose-nginx
4e681d94c146   php:7.2-fpm-mysqli   "docker-php-entrypoi…"   7 seconds ago    Up 5 seconds    0.0.0.0:9000->9000/tcp   compose-php
fcdf6ac20bac   mysql:5.7            "docker-entrypoint.s…"   7 seconds ago    Up 5 seconds    0.0.0.0:3306->3306/tcp   compose-mysql
```

`nginx`、`php`、`mysql` 三个容器都启动了。如果没有全部启动,可以再执行一次 `docker-compose up -d`;如果还是不成功,就排查一下配置是否有误。

**验证**:浏览器访问 `http://服务器IP/index.php`,如果看到 PHP 的 `phpinfo()` 信息页面,说明 `nginx` 与 `php` 容器已经关联成功。

也可以用 Navicat 等数据库工具连接 MySQL:账号 `root`,密码是 `MYSQL_ROOT_PASSWORD` 环境变量的值,端口 `3306`。

## 五、停止与重启服务

停止服务有两种方式:

### 1. 使用 docker-compose 停止

```bash
docker-compose stop
```

输出示例:

```text
Stopping compose-nginx ... done
Stopping compose-php   ... done
Stopping compose-mysql ... done
```

### 2. 使用 docker 命令停止

```bash
docker stop 容器名/容器ID
```

### 3. 重启服务

直接执行下面的命令即可。即使只停止了部分容器,它也会把没有启动的容器重新启动:

```bash
docker-compose up -d
```

## 六、PHP 连接 MySQL

在 `php-nginx-compose/html/` 目录下新建 `connect_mysql.php`:

```php
<?php
// mysqli_connect 参数:主机、账号、密码、数据库、端口
$link = mysqli_connect("10.10.10.1", "root", "你的密码", "mysql", 3306);

if (!$link) {
    echo "Error: Unable to connect to MySQL." . PHP_EOL . "</br>";
    echo "Debugging errno: " . mysqli_connect_errno() . PHP_EOL . "</br>";
    echo "Debugging error: " . mysqli_connect_error() . PHP_EOL . "</br>";
    exit;
}

echo "Success: A proper connection to MySQL was made! The my_db database is great." . PHP_EOL . "</br>";
echo "Host information: " . mysqli_get_host_info($link) . PHP_EOL;

mysqli_close($link);
?>
```

这里的 IP 就是 `docker-compose.yml` 中配置的子网 IP `ipv4_address: 10.10.10.1`。因为三个容器在同一个网络里,也可以直接把主机写成服务名 `mysql`。`mysqli_connect` 的用法可以参考 [PHP 官方文档](https://www.php.net/manual/en/function.mysqli-connect.php)。

**验证**:浏览器访问 `http://服务器IP/connect_mysql.php`,如果页面输出 `Success: A proper connection to MySQL was made!` 以及 `Host information: 10.10.10.1 via TCP/IP`,说明连接 MySQL 成功。

## 七、使用提示

1. **Compose 版本**:新版 Docker 已把 Compose 做成内置插件,命令是 `docker compose`(中间是空格,不是连字符),安装 Docker 时通常已经自带,不需要再单独下载 `docker-compose`。旧的 `docker-compose` 命令(v1)已停止维护。
2. **`version` 字段**:新版 Compose 中顶层的 `version: '3'` 已经是过时字段,可以删掉,保留也不会影响使用。
3. **镜像版本**:PHP 7.2、MySQL 5.7 均已停止维护,新项目建议换成较新的稳定版本,并注意 MySQL 8 默认使用 `caching_sha2_password` 认证,老的客户端或 PHP 版本可能需要额外配置。
4. **不要暴露不必要的端口**:示例中把 PHP-FPM 的 `9000` 和 MySQL 的 `3306` 映射到了宿主机,仅用于方便调试。生产环境中,同一网络内的容器之间可以直接互访,不需要映射这两个端口,尤其不要把 `3306` 暴露到公网。
5. **数据持久化**:示例没有给 MySQL 配置数据卷,删除容器后数据会丢失。实际使用时,请给 MySQL 增加数据卷,例如 `- "$PWD/mysql-data:/var/lib/mysql"`。
6. **密码安全**:`MYSQL_ROOT_PASSWORD` 不要使用弱密码,也不要把真实密码提交到公开仓库,可以改用 `.env` 文件或 Docker secrets。
