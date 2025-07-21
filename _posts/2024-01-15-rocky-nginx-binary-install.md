---
layout: post
title: "Rocky Linux 二进制安装 Nginx 完整指南"
author: "DevOps Engineer"
categories: [Linux, Nginx, Rocky]
tags: [nginx, rocky, systemd, logrotate, binary]
image: images/nginx-logo.png
---

在生产环境中，有时我们需要通过二进制包的方式安装 Nginx，以获得更好的性能和自定义配置。本文将详细介绍在 Rocky Linux 系统上如何进行 Nginx 的二进制安装，并配置 systemd 服务管理和 logrotate 日志轮转。

## 环境信息

- **操作系统**: Rocky Linux 8/9
- **Nginx 版本**: 1.24.0 (stable)
- **安装方式**: 二进制编译安装
- **服务管理**: systemd
- **日志管理**: logrotate

## 1. 系统准备

### 1.1 更新系统

```bash
# 更新系统包
sudo dnf update -y

# 安装必要的开发工具
sudo dnf groupinstall "Development Tools" -y

# 安装编译依赖
sudo dnf install -y pcre-devel zlib-devel openssl-devel wget gcc gcc-c++ make
```

### 1.2 创建 nginx 用户

```bash
# 创建 nginx 系统用户
sudo useradd -r -s /sbin/nologin -d /var/cache/nginx -c "nginx user" nginx
```

### 1.3 创建必要目录

```bash
# 创建 nginx 安装目录
sudo mkdir -p /data/nginx/{conf,logs,client_temp,proxy_temp,fastcgi_temp,uwsgi_temp,scgi_temp}
sudo mkdir -p /data/nginx/html
```

## 2. 下载和编译 Nginx

### 2.1 下载 Nginx 源码

```bash
# 切换到临时目录
cd /tmp

# 下载 Nginx 源码
wget http://nginx.org/download/nginx-1.28.0.tar.gz
tar -zxvf nginx-1.28.0.tar.gz
cd nginx-1.28.0
```

### 2.2 配置编译选项

```bash
# 配置编译参数（安装到 /data 目录）
./configure \
  --prefix=/data/nginx \
  --sbin-path=/data/nginx/sbin/nginx \
  --modules-path=/data/nginx/modules \
  --conf-path=/data/nginx/conf/nginx.conf \
  --error-log-path=/data/nginx/logs/error.log \
  --http-log-path=/data/nginx/logs/access.log \
  --pid-path=/data/nginx/logs/nginx.pid \
  --lock-path=/data/nginx/logs/nginx.lock \
  --http-client-body-temp-path=/data/nginx/client_temp \
  --http-proxy-temp-path=/data/nginx/proxy_temp \
  --http-fastcgi-temp-path=/data/nginx/fastcgi_temp \
  --http-uwsgi-temp-path=/data/nginx/uwsgi_temp \
  --http-scgi-temp-path=/data/nginx/scgi_temp \
  --user=nginx \
  --group=nginx \
  --with-compat \
  --with-file-aio \
  --with-threads \
  --with-http_addition_module \
  --with-http_auth_request_module \
  --with-http_dav_module \
  --with-http_flv_module \
  --with-http_gunzip_module \
  --with-http_gzip_static_module \
  --with-http_mp4_module \
  --with-http_random_index_module \
  --with-http_realip_module \
  --with-http_secure_link_module \
  --with-http_slice_module \
  --with-http_ssl_module \
  --with-http_stub_status_module \
  --with-http_sub_module \
  --with-http_v2_module \
  --with-mail \
  --with-mail_ssl_module \
  --with-stream \
  --with-stream_realip_module \
  --with-stream_ssl_module \
  --with-stream_ssl_preread_module
```

### 2.3 编译和安装

```bash
# 编译 (使用多核心加速)
make -j$(nproc)

# 安装
sudo make install

# 验证安装
nginx -v
```

## 3. 配置 Nginx

### 3.1 优化默认配置文件

安装完成后，Nginx 会在 `/data/nginx/conf/` 目录下生成默认的配置文件。我们需要对默认的 `nginx.conf` 进行优化，主要是完善日志格式：

```bash
# 备份原配置文件
sudo cp /data/nginx/conf/nginx.conf /data/nginx/conf/nginx.conf.bak

# 修改默认配置文件，添加完善的日志格式
sudo tee /data/nginx/conf/nginx.conf > /dev/null <<'EOF'
#user  nobody;
worker_processes  auto;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
error_log  logs/error.log  info;

#pid        logs/nginx.pid;

events {
    worker_connections  1024;
    use epoll;
    multi_accept on;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    # 完善的日志格式定义
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    
    # 详细的日志格式，包含更多信息
    log_format  detailed  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for" '
                         '$request_time $upstream_response_time '
                         '"$upstream_addr" $upstream_status '
                         '$connection $connection_requests';
    
    # JSON 格式日志，便于日志分析
    log_format json escape=json
    '{
        "timestamp": "$time_iso8601",
        "remote_addr": "$remote_addr",
        "remote_user": "$remote_user",
        "request": "$request",
        "status": $status,
        "body_bytes_sent": $body_bytes_sent,
        "http_referer": "$http_referer",
        "http_user_agent": "$http_user_agent",
        "http_x_forwarded_for": "$http_x_forwarded_for",
        "request_time": $request_time,
        "upstream_response_time": "$upstream_response_time",
        "upstream_addr": "$upstream_addr",
        "upstream_status": "$upstream_status",
        "connection": $connection,
        "connection_requests": $connection_requests
    }';

    # 使用详细格式记录访问日志
    access_log  logs/access.log  detailed;
    
    # 可选：同时记录 JSON 格式日志
    # access_log  logs/access.json  json;

    sendfile        on;
    tcp_nopush      on;
    tcp_nodelay     on;
    
    keepalive_timeout  65;
    
    # Gzip 压缩配置
    gzip  on;
    gzip_vary on;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;

    server {
        listen       80;
        server_name  localhost;

        #charset koi8-r;

        # 为这个虚拟主机单独配置访问日志
        access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # 状态监控页面
        location /nginx_status {
            stub_status on;
            access_log off;
            allow 127.0.0.1;
            deny all;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
}
EOF
```

### 3.2 设置文件权限

```bash
# 设置 nginx 目录权限
sudo chown -R nginx:nginx /data/nginx
sudo chmod 755 /data/nginx
sudo chmod 644 /data/nginx/conf/nginx.conf
sudo chmod 755 /data/nginx/conf
sudo chmod 755 /data/nginx/logs
sudo chmod 755 /data/nginx/html
```

## 4. 配置 systemd 服务

### 4.1 创建 systemd 服务文件

```bash
# 创建 nginx.service 文件
sudo tee /etc/systemd/system/nginx.service > /dev/null <<EOF
[Unit]
Description=The nginx HTTP and reverse proxy server
Documentation=http://nginx.org/en/docs/
After=network-online.target remote-fs.target nss-lookup.target
Wants=network-online.target

[Service]
Type=forking
PIDFile=/data/nginx/logs/nginx.pid
ExecStartPre=/data/nginx/sbin/nginx -t
ExecStart=/data/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP \$MAINPID
ExecStop=/bin/kill -s TERM \$MAINPID
KillSignal=SIGQUIT
TimeoutStopSec=5
KillMode=mixed
PrivateTmp=true

# Security settings
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/data/nginx

[Install]
WantedBy=multi-user.target
EOF
```

### 4.2 启用和启动服务

```bash
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启用 nginx 服务（开机自启）
sudo systemctl enable nginx

# 启动 nginx 服务
sudo systemctl start nginx

# 检查服务状态
sudo systemctl status nginx
```

### 4.3 验证服务

```bash
# 检查 nginx 进程
ps aux | grep nginx

# 检查端口监听
ss -tlnp | grep :80

# 测试配置文件
/data/nginx/sbin/nginx -t

# 测试网页访问
curl -I http://localhost
```

## 5. 配置防火墙

```bash
# 检查防火墙状态
sudo firewall-cmd --state

# 永久开放 HTTP 端口
sudo firewall-cmd --permanent --add-service=http

# 永久开放 HTTPS 端口（如果需要）
sudo firewall-cmd --permanent --add-service=https

# 重新加载防火墙规则
sudo firewall-cmd --reload

# 查看开放的服务
sudo firewall-cmd --list-services
```

## 6. 配置 Logrotate

### 6.1 创建 logrotate 配置文件

```bash
# 创建 nginx logrotate 配置
sudo tee /etc/logrotate.d/nginx > /dev/null <<EOF
/data/nginx/logs/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 644 nginx nginx
    sharedscripts
    postrotate
        if [ -f /data/nginx/logs/nginx.pid ]; then
            /bin/kill -USR1 \$(cat /data/nginx/logs/nginx.pid)
        fi
    endscript
}
EOF
```

### 6.2 logrotate 配置说明

- **daily**: 每天轮转一次日志
- **missingok**: 如果日志文件不存在，不报错
- **rotate 30**: 保留30个轮转的日志文件
- **compress**: 压缩轮转的日志文件
- **delaycompress**: 延迟压缩，下次轮转时才压缩
- **notifempty**: 如果日志文件为空，不进行轮转
- **create 644 nginx nginx**: 创建新日志文件的权限和所有者
- **sharedscripts**: 多个日志文件共享脚本
- **postrotate**: 轮转后执行的脚本

### 6.3 测试 logrotate

```bash
# 测试 logrotate 配置
sudo logrotate -d /etc/logrotate.d/nginx

# 强制执行 logrotate
sudo logrotate -f /etc/logrotate.d/nginx

# 查看 logrotate 状态
sudo cat /var/lib/logrotate/logrotate.status | grep nginx
```

## 7. 性能优化配置

### 7.1 系统级优化

```bash
# 优化文件描述符限制
sudo tee -a /etc/security/limits.conf > /dev/null <<EOF
nginx soft nofile 65535
nginx hard nofile 65535
EOF

# 优化内核参数
sudo tee /etc/sysctl.d/99-nginx.conf > /dev/null <<EOF
# 网络优化
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 5000
net.ipv4.tcp_max_syn_backlog = 65535
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_keepalive_time = 1200
net.ipv4.tcp_max_tw_buckets = 5000

# 文件系统优化
fs.file-max = 2097152
vm.swappiness = 10
EOF

# 应用内核参数
sudo sysctl -p /etc/sysctl.d/99-nginx.conf
```

### 7.2 Nginx 性能配置

```bash
# 创建性能优化配置文件
sudo tee /etc/nginx/conf.d/performance.conf > /dev/null <<EOF
# Worker 进程优化
worker_processes auto;
worker_cpu_affinity auto;
worker_rlimit_nofile 65535;

# 事件模块优化
events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
    accept_mutex off;
}

# HTTP 模块优化
http {
    # 连接优化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # 缓冲区优化
    client_body_buffer_size 128k;
    client_max_body_size 10m;
    client_header_buffer_size 1k;
    large_client_header_buffers 4 4k;
    output_buffers 1 32k;
    postpone_output 1460;
    
    # 超时设置
    client_header_timeout 3m;
    client_body_timeout 3m;
    send_timeout 3m;
    
    # 压缩优化
    gzip on;
    gzip_vary on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 6;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/json
        application/javascript
        application/xml+rss
        application/atom+xml
        image/svg+xml;
}
EOF
```

## 8. 监控和维护

### 8.1 创建监控脚本

```bash
# 创建 nginx 状态检查脚本
sudo tee /usr/local/bin/nginx-monitor.sh > /dev/null <<'EOF'
#!/bin/bash

# Nginx 监控脚本
LOG_FILE="/var/log/nginx/monitor.log"
DATE=$(date '+%Y-%m-%d %H:%M:%S')

# 检查 nginx 进程
if ! pgrep nginx > /dev/null; then
    echo "[$DATE] ERROR: Nginx process not running" >> $LOG_FILE
    systemctl start nginx
    echo "[$DATE] INFO: Attempted to restart nginx" >> $LOG_FILE
else
    echo "[$DATE] INFO: Nginx is running" >> $LOG_FILE
fi

# 检查端口监听
if ! ss -tlnp | grep :80 > /dev/null; then
    echo "[$DATE] ERROR: Port 80 not listening" >> $LOG_FILE
fi

# 检查配置文件
if ! nginx -t > /dev/null 2>&1; then
    echo "[$DATE] ERROR: Nginx configuration test failed" >> $LOG_FILE
fi

# 检查日志文件大小
ACCESS_LOG_SIZE=$(stat -c%s "/var/log/nginx/access.log" 2>/dev/null || echo 0)
ERROR_LOG_SIZE=$(stat -c%s "/var/log/nginx/error.log" 2>/dev/null || echo 0)

if [ $ACCESS_LOG_SIZE -gt 1073741824 ]; then  # 1GB
    echo "[$DATE] WARNING: Access log size is large: $ACCESS_LOG_SIZE bytes" >> $LOG_FILE
fi

if [ $ERROR_LOG_SIZE -gt 104857600 ]; then  # 100MB
    echo "[$DATE] WARNING: Error log size is large: $ERROR_LOG_SIZE bytes" >> $LOG_FILE
fi
EOF

# 设置执行权限
sudo chmod +x /usr/local/bin/nginx-monitor.sh

# 添加到 crontab
(crontab -l 2>/dev/null; echo "*/5 * * * * /usr/local/bin/nginx-monitor.sh") | crontab -
```

### 8.2 创建备份脚本

```bash
# 创建配置备份脚本
sudo tee /usr/local/bin/nginx-backup.sh > /dev/null <<'EOF'
#!/bin/bash

# Nginx 配置备份脚本
BACKUP_DIR="/backup/nginx"
DATE=$(date '+%Y%m%d_%H%M%S')
BACKUP_FILE="nginx_config_$DATE.tar.gz"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 备份配置文件
tar -czf "$BACKUP_DIR/$BACKUP_FILE" /etc/nginx/ /usr/share/nginx/html/

echo "Nginx configuration backed up to: $BACKUP_DIR/$BACKUP_FILE"

# 保留最近7天的备份
find $BACKUP_DIR -name "nginx_config_*.tar.gz" -mtime +7 -delete
EOF

# 设置执行权限
sudo chmod +x /usr/local/bin/nginx-backup.sh

# 添加每日备份任务
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/nginx-backup.sh") | crontab -
```

## 9. 常用管理命令

### 9.1 服务管理

```bash
# 启动服务
sudo systemctl start nginx

# 停止服务
sudo systemctl stop nginx

# 重启服务
sudo systemctl restart nginx

# 重新加载配置
sudo systemctl reload nginx

# 查看服务状态
sudo systemctl status nginx

# 查看服务日志
sudo journalctl -u nginx -f
```

### 9.2 配置管理

```bash
# 测试配置文件
/data/nginx/sbin/nginx -t

# 重新加载配置
/data/nginx/sbin/nginx -s reload

# 查看编译参数
/data/nginx/sbin/nginx -V

# 查看版本信息
/data/nginx/sbin/nginx -v
```

### 9.3 日志管理

```bash
# 查看访问日志
sudo tail -f /data/nginx/logs/access.log

# 查看错误日志
sudo tail -f /data/nginx/logs/error.log

# 分析访问日志（统计IP）
sudo awk '{print $1}' /data/nginx/logs/access.log | sort | uniq -c | sort -nr | head -10

# 分析访问日志（统计状态码）
sudo awk '{print $9}' /data/nginx/logs/access.log | sort | uniq -c | sort -nr
```

## 10. 故障排除

### 10.1 常见问题

1. **服务启动失败**
   ```bash
   # 检查配置文件语法
   /data/nginx/sbin/nginx -t
   
   # 查看详细错误信息
   sudo journalctl -u nginx -n 50
   
   # 检查端口占用
   sudo ss -tlnp | grep :80
   ```

2. **权限问题**
   ```bash
   # 检查文件权限
   ls -la /data/nginx/conf/
   ls -la /data/nginx/logs/
   
   # 重新设置权限
   sudo chown -R nginx:nginx /data/nginx
   ```

3. **性能问题**
   ```bash
   # 查看连接数
   ss -s
   
   # 查看 nginx 状态
   curl http://localhost/nginx_status
   
   # 查看系统资源
   top -p $(pgrep nginx)
   ```

### 10.2 调试技巧

```bash
# 启用调试模式
sudo sed -i 's/error_log.*/error_log logs\/error.log debug;/' /data/nginx/conf/nginx.conf
sudo systemctl reload nginx

# 查看详细错误信息
sudo tail -f /data/nginx/logs/error.log

# 恢复正常日志级别
sudo sed -i 's/error_log.*/error_log logs\/error.log info;/' /data/nginx/conf/nginx.conf
sudo systemctl reload nginx
```

## 总结

通过本文的详细步骤，我们成功实现了：

1. **🔧 二进制编译安装**：从源码编译安装 Nginx，获得最佳性能和自定义功能
2. **⚙️ systemd 服务管理**：配置了完整的 systemd 服务，支持开机自启和服务管理
3. **📋 logrotate 日志轮转**：实现了自动日志轮转，避免日志文件过大
4. **🚀 性能优化**：系统级和应用级的性能优化配置
5. **📊 监控维护**：自动化监控和备份脚本
6. **🛡️ 安全配置**：合理的权限设置和安全加固

这套配置方案具有以下优势：
- ✅ **高性能**：编译时启用了多种性能模块
- ✅ **易管理**：systemd 服务管理，支持标准的服务操作
- ✅ **自动化**：日志轮转、监控、备份全部自动化
- ✅ **可扩展**：模块化配置，便于后续扩展
- ✅ **生产就绪**：包含了生产环境所需的各种配置

建议在生产环境部署前，先在测试环境验证所有配置，并根据实际业务需求调整相关参数。