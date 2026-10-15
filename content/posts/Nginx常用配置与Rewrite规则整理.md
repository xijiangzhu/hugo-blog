---
title: Nginx常用配置与Rewrite规则整理
date: 2026-10-12 10:00:00
tags:
  - Nginx
categories:
  - Linux
description: 整理 Nginx 常用的正则匹配、rewrite 指令、全局变量,以及防盗链、域名跳转、HTTPS 跳转等常见配置实例。
---

本文整理 Nginx 日常配置中最常用的内容:正则匹配符、文件判断、rewrite 指令与 flag、常见规则实例、全局变量,以及与 Apache 规则的对应关系。

<!-- more -->

## 一、正则表达式匹配

| 符号 | 含义 |
| --- | --- |
| `~` | 区分大小写匹配 |
| `~*` | 不区分大小写匹配 |
| `!~` | 区分大小写不匹配 |
| `!~*` | 不区分大小写不匹配 |

## 二、文件及目录匹配

| 符号 | 含义 |
| --- | --- |
| `-f` / `!-f` | 判断文件是否存在 / 不存在 |
| `-d` / `!-d` | 判断目录是否存在 / 不存在 |
| `-e` / `!-e` | 判断文件或目录是否存在 / 不存在 |
| `-x` / `!-x` | 判断文件是否可执行 / 不可执行 |

## 三、rewrite 指令的 flag 标记

rewrite 指令的最后一个参数是 flag 标记,常用的有四种:

| flag | 说明 |
| --- | --- |
| `last` | 相当于 Apache 里的 `[L]` 标记,表示本条规则执行完后,用重写后的 URI 重新发起请求(重新匹配 `server{}` 中的 location) |
| `break` | 本条规则匹配完成后,终止匹配,不再匹配后面的规则 |
| `redirect` | 返回 302 临时重定向,浏览器地址栏会显示跳转后的 URL |
| `permanent` | 返回 301 永久重定向,浏览器地址栏会显示跳转后的 URL |

> 使用 `last` 和 `break` 实现 URI 重写时,浏览器地址栏不变,但两者有细微差别:
>
> - 使用 `alias` 指令时,必须用 `last` 标记。
> - 使用 `proxy_pass` 指令时,需要用 `break` 标记。
> - `last` 在本条 rewrite 规则执行完毕后,会对其所在的 `server{...}` 重新发起请求;`break` 则在本条规则匹配完成后直接终止匹配。

例如,将 `/photo/123456` 重定向到 `/path/to/photo/12/1234/123456.png`:

```nginx
rewrite "/photo/([0-9]{2})([0-9]{2})([0-9]{2})" /path/to/photo/$1/$1$2/$1$2$3.png;
```

## 四、Rewrite 规则相关指令

### 1. break 指令

- 使用环境:`server`、`location`、`if`
- 作用:完成当前的规则集,不再处理后续的 rewrite 指令。

### 2. if 指令

- 使用环境:`server`、`location`
- 作用:检查条件是否成立,成立则执行大括号内的语句。

注意:`if` 指令**不支持嵌套**,也**不支持多个条件的 `&&` 和 `||` 处理**。

### 3. return 指令

- 语法:`return code;`
- 使用环境:`server`、`location`、`if`
- 作用:结束规则的执行,并返回状态码给客户端。

示例:访问的 URL 以 `.sh` 或 `.bash` 结尾时,返回 403:

```nginx
location ~ .*\.(sh|bash)?$ {
    return 403;
}
```

### 4. rewrite 指令

- 语法:`rewrite regex replacement flag;`
- 使用环境:`server`、`location`、`if`
- 作用:根据正则表达式重定向 URI 或修改字符串,指令按配置文件中的顺序执行。

注意:重写表达式只对**相对路径**有效。如果要匹配主机名,应使用 `if` 语句,示例:

```nginx
if ($host ~* www\.(.*)) {
    set $host_without_www $1;
    rewrite ^(.*)$ http://$host_without_www$1 permanent;
}
```

### 5. set 指令

- 语法:`set $variable value;`
- 默认值:无
- 使用环境:`server`、`location`、`if`
- 作用:定义一个变量并赋值。变量的值可以是文本、变量,或两者的组合。

示例:

```nginx
set $varname "hello world";
```

### 6. uninitialized_variable_warn 指令

- 语法:`uninitialized_variable_warn on|off;`
- 使用环境:`http`、`server`、`location`、`if`
- 作用:开启或关闭未初始化变量的警告信息,默认开启。

## 五、Rewrite 规则编写实例

### 1. 文件或目录不存在时,重定向到某个 php 文件

```nginx
if (!-e $request_filename) {
    rewrite ^/(.*)$ /index.php last;
}
```

### 2. 目录对换

`/123456/xxxx` 变为 `/xxxx?id=123456`:

```nginx
rewrite ^/(\d+)/(.+)/ /$2?id=$1 last;
```

### 3. IE 浏览器重定向到 /ie 目录

```nginx
if ($http_user_agent ~ MSIE) {
    rewrite ^(.*)$ /ie/$1 break;
}
```

### 4. 禁止访问多个目录

```nginx
location ~ ^/(cron|templates)/ {
    deny all;
    break;
}
```

### 5. 禁止访问以 /data 开头的文件

```nginx
location ~ ^/data {
    deny all;
}
```

### 6. 禁止访问 .sh、.flv、.mp3 后缀的文件

```nginx
location ~ .*\.(sh|flv|mp3)$ {
    return 403;
}
```

### 7. 设置某些类型文件的浏览器缓存时间

```nginx
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$ {
    expires 30d;
}

location ~ .*\.(js|css)$ {
    expires 1h;
}
```

### 8. 给 favicon.ico 和 robots.txt 设置过期时间

`favicon.ico` 缓存 99 天,`robots.txt` 缓存 7 天,并且不记录 404 错误日志:

```nginx
location ~ (favicon.ico) {
    log_not_found off;
    expires 99d;
    break;
}

location ~ (robots.txt) {
    log_not_found off;
    expires 7d;
    break;
}
```

### 9. 设定某个文件的过期时间

这里设为 600 秒,并且不记录访问日志:

```nginx
location ^~ /html/scripts/loadhead_1.js {
    access_log off;
    root /opt/lampp/htdocs/web;
    expires 600;
    break;
}
```

### 10. 文件防盗链并设置过期时间

说明:

- `return 412`:自定义 HTTP 状态码(默认是 403),方便找出真正的盗链请求。
- `rewrite ^/ http://img.linuxidc.net/leech.gif;`:显示一张防盗链提示图片。
- `access_log off;`:不记录访问日志,减轻压力。
- `expires 3d;`:所有文件设置 3 天的浏览器缓存。

```nginx
location ~* ^.+\.(jpg|jpeg|gif|png|swf|rar|zip|css|js)$ {
    valid_referers none blocked *.linuxidc.com *.linuxidc.net localhost 208.97.167.194;
    if ($invalid_referer) {
        rewrite ^/ http://img.linuxidc.net/leech.gif;
        return 412;
        break;
    }

    access_log off;
    root /opt/lampp/htdocs/web;
    expires 3d;
    break;
}
```

### 11. 只允许固定 IP 访问网站,并加上密码

```nginx
root /opt/htdocs/www;
allow 208.97.167.194;
allow 222.33.1.2;
allow 231.152.49.4;
deny all;
auth_basic "C1G_ADMIN";
auth_basic_user_file htpasswd;
```

### 12. 将多级目录下的文件转成一个文件,增强 SEO 效果

`/job-123-456-789.html` 指向 `/job/123/456/789.html`:

```nginx
rewrite ^/job-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /job/$1/$2/jobshow_$3.html last;
```

### 13. 文件和目录不存在时,转发到后端

```nginx
if (!-e $request_filename) {
    proxy_pass http://127.0.0.1;
}
```

### 14. 将根目录下某个文件夹指向二级目录

需求:`/shanghaijob/` 指向 `/area/shanghai/`。

```nginx
rewrite ^/([0-9a-z]+)job/(.*)$ /area/$1/$2 last;
```

> 如果把 `last` 改成 `permanent`,浏览器地址栏会显示 `/area/shanghai/`。

**问题一**:上面的写法在访问 `/shanghaijob`(末尾没有斜杠)时不会匹配。补充一条规则:

```nginx
rewrite ^/([0-9a-z]+)job$ /area/$1/ last;
rewrite ^/([0-9a-z]+)job/(.*)$ /area/$1/$2 last;
```

这样 `/shanghaijob` 也能访问了。

**问题二**:页面中的相对链接会失效。例如 `./list_1.html` 的真实地址是 `/area/shanghai/list_1.html`,但因为地址栏仍是 `/shanghaijob`,浏览器会把它解析成 `/list_1.html`,导致无法访问。

想用下面的 `-d` 判断来自动补斜杠跳转,也行不通:

```nginx
if (-d $request_filename) {
    rewrite ^/(.*)([^/])$ http://$host/$1$2/ permanent;
}
```

原因是 `-d` 要求 `$request_filename` 必须是**真实存在的目录**,而这里的路径是 rewrite 出来的虚拟路径,条件不会为真。

**解决方法**:手动写一条跳转规则,补上末尾斜杠:

```nginx
rewrite ^/([0-9a-z]+)job$ /$1job/ permanent;
rewrite ^/([0-9a-z]+)job/(.*)$ /area/$1/$2 last;
```

### 15. 域名跳转

```nginx
server {
    listen 80;
    server_name jump.linuxidc.com;
    index index.html index.htm index.php;
    root /opt/lampp/htdocs/www;
    rewrite ^/ http://www.linuxidc.com/;
    access_log off;
}
```

### 16. 多域名转向

```nginx
server_name www.linuxidc.com www.linuxidc.net;
index index.html index.htm index.php;
root /opt/lampp/htdocs;

if ($host ~ "linuxidc\.net") {
    rewrite ^(.*) http://www.linuxidc.com$1 permanent;
}
```

### 17. 80 端口转 443

原笔记中的写法如下,但它是把非 80 端口的请求跳转回 **http**,并不能实现跳转到 HTTPS:

```nginx
# 原写法(有误,仅供对照)
if ($server_port != '80') {
    rewrite ^/(.*)$ http://$host/$1 permanent;
}
```

正确的 80 转 443 写法见文末「HTTP 强制跳转到 HTTPS」一节。

## 六、Nginx 全局变量

| 变量 | 说明 |
| --- | --- |
| `$arg_PARAMETER` | GET 请求中参数 `PARAMETER` 的值 |
| `$args` | 请求行中(GET 请求)的参数,如 `foo=123&bar=blahblah` |
| `$binary_remote_addr` | 二进制格式的客户端地址 |
| `$body_bytes_sent` | 响应时送出的 body 字节数,即使连接中断,这个数据也是精确的 |
| `$content_length` | 请求头中的 Content-Length 字段 |
| `$content_type` | 请求头中的 Content-Type 字段 |
| `$cookie_COOKIE` | Cookie 变量 `COOKIE` 的值 |
| `$document_root` | 当前请求在 `root` 指令中指定的值 |
| `$document_uri` | 与 `$uri` 相同 |
| `$host` | 请求的主机头字段,否则为服务器名称 |
| `$hostname` | 机器的主机名,即 gethostname 的返回值 |
| `$http_HEADER` | 任意请求头的值,如 `$http_user_agent` |
| `$is_args` | 有 `$args` 参数时为 `?`,否则为空 |
| `$http_user_agent` | 客户端 agent 信息 |
| `$http_cookie` | 客户端 cookie 信息 |
| `$limit_rate` | 可用于限制连接速率 |
| `$query_string` | 与 `$args` 相同 |
| `$request_body_file` | 客户端请求主体信息的临时文件名 |
| `$request_method` | 客户端请求的动作,通常为 GET 或 POST |
| `$remote_addr` | 客户端的 IP 地址 |
| `$remote_port` | 客户端的端口 |
| `$remote_user` | 经过 Auth Basic 模块验证的用户名 |
| `$request_completion` | 请求结束时设置为 OK;请求未结束,或不是请求链的最后一个时为空 |
| `$request_filename` | 当前请求的文件路径,由 `root` 或 `alias` 指令与 URI 生成 |
| `$request_uri` | 包含请求参数的原始 URI,不含主机名,如 `/foo/bar.php?arg=baz`,不能修改 |
| `$scheme` | 请求使用的协议,如 http、https |
| `$server_protocol` | 请求使用的协议版本,通常是 HTTP/1.0 或 HTTP/1.1 |
| `$server_addr` | 服务器地址,在完成一次系统调用后可以确定这个值 |
| `$server_name` | 服务器名称 |
| `$server_port` | 请求到达服务器的端口号 |

## 七、Apache 和 Nginx 规则的对应关系

| Apache | Nginx |
| --- | --- |
| `RewriteCond` | `if` |
| `RewriteRule` | `rewrite` |
| `[R]` | `redirect` |
| `[P]` | `last` |
| `[R,L]` | `redirect` |
| `[P,L]` | `last` |
| `[PT,L]` | `last` |

### 示例:只允许指定域名访问,其他域名一律转向 www.linuxidc.net

**Apache:**

```apache
RewriteCond %{HTTP_HOST} !^(.*?)\.aaa\.com$ [NC]
RewriteCond %{HTTP_HOST} !^localhost$
RewriteCond %{HTTP_HOST} !^192\.168\.0\.(.*?)$
RewriteRule ^/(.*)$ http://www.linuxidc.net [R,L]
```

**Nginx:**

```nginx
if ($host ~* ^(.*)\.aaa\.com$) {
    set $allowHost '1';
}

if ($host ~* ^localhost) {
    set $allowHost '1';
}

if ($host ~* ^192\.168\.1\.(.*?)$) {
    set $allowHost '1';
}

if ($allowHost !~ '1') {
    rewrite ^/(.*)$ http://www.linuxidc.net redirect;
}
```

### 示例:所有页面都显示同一张图片

```nginx
rewrite ^(.*)$ /1/1.png break;
```

### 示例:访问的链接不存在时,重定向到某个页面

```nginx
location / {
    if (!-f $request_filename) {
        rewrite ^/(.+)$ /1/1.png last;
    }
}
```

### 示例:不带 www 跳转到带 www 的域名

在 `server{}` 里添加:

```nginx
if ($host != 'www.test.com') {
    rewrite ^/(.*)$ http://www.test.com/$1 permanent;
}
```

### 示例:禁止 IE 访问

```nginx
location / {
    if ($http_user_agent ~ MSIE) {
        return 503;
    }
}
```

### 示例:防盗链

```nginx
location ~* \.(gif|jpg|png|bmp)$ {
    valid_referers none blocked *.baidu.com server_names ~\.google\. ~\.baidu\.;
    if ($invalid_referer) {
        return 403;
        #rewrite ^/ http://www.ttlsa.com/403.jpg;
    }
}
```

## 八、HTTP 跳转到 HTTPS

### Apache

把下面三句规则放到 `VirtualHost` 的 80 端口配置里,即可实现所有 HTTP 请求跳转到 HTTPS:

```apache
RewriteEngine on
RewriteCond %{SERVER_PORT} !^443$
RewriteRule ^(.*)?$ https://%{SERVER_NAME}$1 [L,R]
```

### Nginx

HTTP 强制跳转到 HTTPS:

```nginx
if ($scheme = http) {
    rewrite ^(.*)$ https://$host$1 permanent;
}
```

更推荐的写法是单独为 80 端口建一个 `server` 块,直接用 `return`,效率更高,也避免使用 `if`:

```nginx
server {
    listen 80;
    server_name www.example.com;
    return 301 https://$host$request_uri;
}
```

## 小结

1. `rewrite` 要注意 flag 的选择:地址栏需要变化用 `redirect` / `permanent`,不需要变化用 `last` / `break`。
2. `if` 不支持嵌套和多条件,复杂逻辑可以借助 `set` 变量拆分,或改用 `location`、`map`。
3. 能用 `return` 解决的跳转,优先用 `return`,比 `rewrite` 更简洁高效。
