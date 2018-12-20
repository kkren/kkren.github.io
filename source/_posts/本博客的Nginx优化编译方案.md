---
title: 本博客的Nginx优化编译方案
date: 2018-10-25 19:02:52
tags: 技术
urlname: nginx-configuration
---

最近从 [Phantom网络加速](https://ssrfq.com/)白嫖了一台贵州的机器。因为没有备案，所以只能请大家用 https 访问了。在编译的过程中，参考了 [imququ](https://imququ.com/post/enable-tls-1-3.html) 等大佬的资料，在最新的 mainline Nginx 之上添加了 ngx_brotli 模块并使用最新的 OpenSSL 编译。<!-- more -->

## 编译 Nginx

我服务器的环境是  debian 9，在编译前先安装依赖。

```shell
sudo apt-get install build-essential libpcre3 libpcre3-dev zlib1g-dev unzip git
```

然后下载并解压 Nginx 和其他模块

```sh
# Nginx
wget -c http://nginx.org/download/nginx-1.15.6.tar.gz
tar zxf nginx-1.15.6.tar.gz
# OpenSSL
wget -c  https://github.com/openssl/openssl/archive/OpenSSL_1_1_1.tar.gz
tar xzf OpenSSL_1_1_1.tar.gz
mv openssl-OpenSSL_1_1_1 openssl
# ngx_brotli
git clone https://github.com/google/ngx_brotli.git
cd ngx_brotli
git submodule update --init
cd ../
```

开始编译

```shell
cd nginx-1.15.6
./configure --add-module=../ngx_brotli --with-openssl=../openssl --with-openssl-opt='enable-tls1_3' --with-http_v2_module --with-http_ssl_module --with-http_gzip_static_module
make
sudo make install
```

如果需要升级 Nginx ，不要执行 `sudo make install` ，执行下面的命令。

```shell
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.old
# 备份 Nginx
cp -f objs/nginx /usr/local/nginx/sbin/nginx
nginx -t
# 解决上一条命令出现的问题后继续
make upgrade
```

## 配置 Nginx

安装后 Nginx 和配置在 `/usr/local/nginx/` 下面。首先打开 `conf/nginx.conf`

```nginx
sendfile        on;
tcp_nopush     	on;
tcp_nodelay    	on;
```

这三行，原来注释掉的改成 on 。

```nginx
gzip			on;
gzip_vary     	on;
gzip_comp_level	6;
gzip_buffers   	16 8k;
gzip_min_length	1000;
gzip_proxied   	any;
gzip_disable   	"msie6";
gzip_http_version	1.0;
gzip_types    	text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;
    
brotli			on;
brotli_comp_level	6;
brotli_types	text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml;  
```

启用 gzip 和 brotli 压缩。然后创建一个网站的配置。

```nginx
ssl_protocols	TLSv1.3 TLSv1.2;
ssl_ciphers		TLS13-AES-256-GCM-SHA384:TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-128-GCM-SHA256:TLS13-AES-128-CCM-8-SHA256:TLS13-AES-128-CCM-SHA256:EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+ECDSA+AES128:EECDH+aRSA+AES128:RSA+AES128:EECDH+ECDSA+AES256:EECDH+aRSA+AES256:RSA+AES256:EECDH;
add_header		Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
add_header		X-Frame-Options deny;
add_header		X-Content-Type-Options nosniff;
```

重点就是加密算法的选择和 HSTS 。

## 其他的优化

博客的所有文件都会用 service worker 永久缓存在读者的电脑上。直接安装 [hexo-service-worker](https://github.com/zoumiaojiang/hexo-service-worker) 插件即可。

博客的图片放在 [sm.ms](https://sm.ms/) 图床上，然后使用 [hexo-lazyload](https://github.com/Robin-front/hexo-lazyload) 制作加载时的缩略图。放上一张首次加载首页的耗时图。![加载图](https://i.loli.net/2018/11/11/5be7d80a4ef45.png)