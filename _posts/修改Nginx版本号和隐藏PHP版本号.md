---
title: 修改Nginx版本号和隐藏PHP版本号
date: 2018-1-26 20:26:28
tags:
- nginx
- 修改Nginx版本号
- 隐藏PHP版本号
categories:
- 服务器
- nginx
---

> 在实践中通常需要主动隐藏或篡改软件版本号，防止攻击者通过特定版本号的已知漏洞进行针对性攻击。下面演示如何修改nginx版本号，以及隐藏php版本号。

![](/images/Hide-Nginx-Version-Number.png)

<!-- more -->
# 修改nginx版本号

## 方法一：修改源码

> 思路：软件版本号肯定记录在源码中，只要找到对应配置文件进行修改再安装即可。下面演示完整的操作过程。

```bash
# 下载软件包
wget -c https://nginx.org/download/nginx-1.12.2.tar.gz
wge -c https://zlib.net/zlib-1.2.11.tar.gz
wget -c https://www.openssl.org/source/openssl-1.0.2n.tar.gz

# 分别进行解压
tar -zxvf nginx-1.12.2.tar.gz
tar -zxvf zlib-1.2.11.tar.gz
tar -zxvf openssl-1.0.2n

# 通常下面的语句查找，可以发现只有一个文件“src/core/nginx.h”符合我们的要求。
grep -rn '1.12.2' nginx-1.12.2

# 编辑文件并修改里面的版本号宏定义语句
vim nginx-1.12.2/src/core/nginx.h
#define NGINX_VERSION      "tianmingxing"
```

上面的动作算完成了版本号修改，接下来开始手动源码安装nginx。

```bash
# 进入nginx解压出来的目录
cd nginx-1.12.2

# 构建编译配置信息，其中/home/xxx需要变成你下载软件的目录。
./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-http_ssl_module --with-openssl=/home/xxx/openssl-1.0.2n --without-http_gzip_module --with-zlib=/home/xxx/zlib-1.2.11

# 编译并安装
make && make install

# nginx会安装在指定目录“/usr/local/nginx”中
# 此时我们来验证一下nginx的版本是否已经应用上面的修改
/usr/local/nginx/sbin/nginx -v
nginx version: nginx/tianmingxing
# 可以看到版本号已经成功修改
```

## 方法二：修改配置

1. 找到nginx主配置文件
    ```bash
    vim /etc/nginx/nginx.conf
    ```
1. 在 `http` 模块中增加一行 `server_tokens off;`
    ```nginx
    http {
        ...
    
        server_tokens off;
    
        include /etc/nginx/mime.types;
        ...
        
        server {}
    }
    ```
1. 重启服务 `systemctl restart nginx`

# 隐藏php版本号

可以很容易的通常修改配置文件实现，修改完重启php-fpm进程就应用了。

```bash
# 查看php配置文件，如果你明确知道配置文件在哪里，可以跳过这一步
php -i | grep "Loaded Configuration File"
# 输出信息
Loaded Configuration File => /etc/php.ini

# 编辑文件ini文件
vim /etc/php.ini
# 搜索到expose_php参数，并修改成下面形式
expose_php = Off
```
