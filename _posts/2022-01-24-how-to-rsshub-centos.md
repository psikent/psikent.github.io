# CentOS 7系统部署rsshub

Tags: app-rsshub
修改时间: March 11, 2021 3:31 PM
创建时间: February 7, 2021 9:49 AM
笔记本: 生活

## 开启本机bbr

[CentOS 7 开启Google BBR - 简书](https://www.notion.so/CentOS-7-Google-BBR-05c11a12ccbd4c658898d60cb665dab9) 

## 安装docker和docker-compose

[CentOS 7系统安装docker和docker-compose](https://www.notion.so/CentOS-7-docker-docker-compose-620b8e8c0a5b4e5d81907a1572bb2c41) 

## RSSHUB部署

创建rsshub文件夹

```bash
mkdir ~/rsshub && cd ~/rsshub
```

下载 docker-compose.yml

```bash
wget https://raw.githubusercontent.com/DIYgod/RSSHub/master/docker-compose.yml
```

```bash
vi docker-compose.yml
```

添加环境变量

```bash
ACCESS_KEY: logitech
REQUEST_RETRY: 2
YOUTUBE_KEY: AIzaSyDR_H2Bt6fGkRHc9B8UUzD8Z450CzcyytQ
TELEGRAM_TOKEN: 1624077904:AAE4uFFGPBfyGH08Rs-glznmYLCYgiTvVR4
GITHUB_ACCESS_TOKEN: e3f5b2843e42bd95154b7f21eb6d609e96e93af3
```

创建 volume 持久化 Redis 缓存

```bash
docker volume create redis-data
```

启动

```bash
docker-compose up -d
```

停止

```bash
docker-compose down
```

## 通过nginx实现https

### 通过acme.sh完成证书签发

[CentOS 7 系统 Trojan 安装及维护](https://www.notion.so/CentOS-7-Trojan-880dbe85f39146289bd45cba8f9c265d) 

安装证书：

```bash
acme.sh --install-cert -d rss.sysview.me \
--key-file /usr/local/etc/certfiles/private.key \
--fullchain-file /usr/local/etc/certfiles/certificate.crt
```

### 生成dhparam

```bash
openssl dhparam -out /usr/local/etc/certfiles/dhparam.pem 2048
```

### 下载nginx并完成配置

[CentOS 7 系统 Trojan 安装及维护](https://www.notion.so/CentOS-7-Trojan-880dbe85f39146289bd45cba8f9c265d) 

nginx.conf

```bash
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http 
{
    include            mime.types;
    default_type       application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    charset            UTF-8;

    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;

    keepalive_timeout  60;

    gzip               on;
    gzip_vary          on;

    gzip_comp_level    6;
    gzip_buffers       16 8k;

    gzip_min_length    1000;
    gzip_proxied       any;
    gzip_disable       "msie6";

    gzip_http_version  1.0;

    gzip_types         text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript image/svg+xml image/x-icon;

    #include            /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

站点配置文件（/etc/nginx/site-available）：注意修改域名

```bash
server
{
    listen 80;
    listen [::]:80;
    server_name rss.sysview.me;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2 fastopen=3 reuseport;
    listen [::]:443 ssl http2 fastopen=3 reuseport;
    server_name rss.sysview.me;
 
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log warn;

    ssl_certificate /usr/local/etc/certfiles/certificate.crt;
    ssl_certificate_key /usr/local/etc/certfiles/private.key;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    ssl_dhparam /usr/local/etc/certfiles/dhparam.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256';
    ssl_prefer_server_ciphers on;

    add_header Strict-Transport-Security max-age=15768000;

    ssl_stapling on;

    ssl_early_data on;

    resolver 114.114.114.114 valid=300s;
    resolver_timeout 10s;

    location / {
        proxy_pass http://127.0.0.1:1200;
    }
}
```

使能配置文件

```bash
sudo ln -s /etc/nginx/sites-available/rss.sysview.me /etc/nginx/sites-enabled/
```

### 启动nginx

```bash
systemctl start nginx
```

注意：vultr的机器还有以下坑：

[vultr机器的安全措施](https://www.notion.so/vultr-fee294351065488782cc23daec2c8e84)