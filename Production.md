# Nginx部署

```shell
yum安装nginx
编译安装nginx
```

# 使用limit_rate限制客户端传输数据的速度

编辑/etc/nginx/nginx.conf

```shell
location / {
            root   /var/www/nginx/;
            index  index.html index.htm;
            limit_rate  2k;  #对每个连接的限速为2k/s
}
```

修改完配置文件，需要重启服务（重新加载服务）

注意：配置文件中的每个语句要以；结尾

# Nginx虚拟主机的配置

```shell
nginx可以实现虚拟主机的配置，nginx支持3种类型的虚拟主机配置。
1、基于域名的虚拟主机 （server_name来区分虚拟主机——应用：web网站）
2、基于ip的虚拟主机， （一个主机绑定多个ip地址）
3、基于端口的虚拟主机 （端口来区分虚拟主机——应用：公司内部网站，外部网站的管理后台）
```

## 基于域名的虚拟主机

```shell
:set paste
```

```shell
user nginx;
worker_processes  4; #指定CPU的进程数，建议是CPU的核心数

#error_log  logs/error.log;
worker_rlimit_nofile 102400;

events {
    worker_connections  1024; #worker进程最大的连接数，建议设为1024
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    server {
        listen       80;
        server_name  web.testpm.com;
        location / {
            root   /var/www/nginx;
            index  index.html index.htm;
            limit_rate  2k;
                }
        }

    server {
        listen       80;
        server_name  web.asuka.com;
        location / {
            root   /asuka/html;
            index  index.html index.htm;
                }
    }
}
```

```shell
[root@localhost nginx]# cat /var/www/nginx/index.html 
hello word

[root@localhost nginx]# cat /asuka/html/index.html 
hello asuka
```

```shell
重新加载配置文件
# 如果编译安装的执行
[root@nginx]# /usr/local/nginx/sbin/nginx -s reload
# 如果 yum 安装的执行
[root@nginx]# nginx -s reload
```

```shell
客户端配置解析
在 C:\Windows\System32\drivers\etc\hosts 文件中添加两行(linux:/etc/hosts)
192.168.91.129 web.testpm.com
192.168.91.129  web.asuka.com
```

## 基于ip的虚拟主机

```shell
[root@localhost ~]# ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 00:0c:29:17:f1:af brd ff:ff:ff:ff:ff:ff
    inet 10.0.105.199/24 brd 10.0.105.255 scope global dynamic ens33
       valid_lft 81438sec preferred_lft 81438sec
    inet6 fe80::9d26:f3f0:db9c:c9be/64 scope link 
       valid_lft forever preferred_lft forever
[root@localhost ~]# yum -y install net-tools
[root@localhost ~]# ifconfig ens33:1 10.0.105.201/24
[root@localhost ~]# ifconfig
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.105.199  netmask 255.255.255.0  broadcast 10.0.105.255
        inet6 fe80::9d26:f3f0:db9c:c9be  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:17:f1:af  txqueuelen 1000  (Ethernet)
        RX packets 9844  bytes 1052722 (1.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 5567  bytes 886269 (865.4 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens33:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.0.105.201  netmask 255.255.255.0  broadcast 10.0.105.255
        ether 00:0c:29:17:f1:af  txqueuelen 1000  (Ethernet)
```

```shell
user nginx;
worker_processes  4; #指定CPU的进程数，建议是CPU的核心数

#error_log  logs/error.log;
worker_rlimit_nofile 102400;

events {
    worker_connections  1024; #worker进程最大的连接数，建议设为1024
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    server {
        listen       80;
        server_name  192.168.174.129;
        location / {
            root   /var/www/nginx;
            index  index.html index.htm;
            limit_rate  2k;
                }
        }

    server {
        listen       80;
        server_name  192.168.174.130;
        location / {
            root   /asuka/html;
            index  index.html index.htm;
                }
    }
}
```

```shell
重新加载配置文件
[root@localhost ~]# /usr/local/nginx/sbin/nginx -s reload
```

```shell
访问测试
浏览器输入：http://192.168.174.129
浏览器输入：http://192.168.174.130
```

```shell
删除绑定的临时ip
[root@localhost ~]# ifconfig ens33:1 10.0.105.201/24 down
```

## 基于端口的虚拟主机

```shell
user nginx;
worker_processes  4; #指定CPU的进程数，建议是CPU的核心数

#error_log  logs/error.log;
worker_rlimit_nofile 102400;

events {
    worker_connections  1024; #worker进程最大的连接数，建议设为1024
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    sendfile        on;

    keepalive_timeout 65;

    server {
        listen       80;
        server_name  web.testpm.com;
        location / {
            root   /var/www/nginx;
            index  index.html index.htm;
            limit_rate  2k;
                }
        }

    server {
        listen       8080;
        server_name  web.testpm.com;
        location / {
            root   /asuka/html;
            index  index.html index.htm;
                }
    }
}

```

```shell
重新加载配置文件
[root@localhost ~]# /usr/local/nginx/sbin/nginx -s reload
测试访问
浏览器输入：http://web.testpm.com/
浏览器输入：http://web.testpm.com:8080
```

# Nginx显示文件夹结构

```shell
user nginx;
worker_processes  4; #指定CPU的进程数，建议是CPU的核心数

#error_log  logs/error.log;
worker_rlimit_nofile 102400;

events {
    worker_connections  1024; #worker进程最大的连接数，建议设为1024
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    sendfile        on;

    keepalive_timeout 65;

    server {
        listen       80;
        server_name  web.testpm.com;
        location / {
                    root /data/www/file;                     #指定实际目录绝对路径；   
                    autoindex on;                            #开启目录浏览功能；   
                    autoindex_exact_size off;            #关闭详细文件大小统计，让文件大小显示MB，GB单位，默认为b；   
                    autoindex_localtime on;              #开启以服务器本地时区显示文件修改日期！  
            #root   /var/www/nginx;
            #index  index.html index.htm;
            #limit_rate  2k;
                }
        }

    server {
        listen       8080;
        server_name  web.testpm.com;
        location / {
            root   /asuka/html;
            index  index.html index.htm;
                }
    }
}
```

```shell
# mkdir -p /data/www/file/dir1/
# mkdir -p /data/www/file/dir2
# vim /data/www/file/dir1/a.txt
123456
# 启动nginx
```

# 启动Nginx proxy代理

```shell
nginx-1的ip：192.168.174.129
已经编译安装好，检查nginx是否启动是否可以访问
```

```shell
nginx-2的ip：192.168.174.130
配置nginx的yum源直接yum安装
启动
编辑nginx的配置文件(编辑之前，删除/注释掉之前的配置):
[root@nginx-server ~]# vim /etc/nginx/conf.d/default.conf
server {
    listen       80;
    server_name  localhost;

    location / {
    proxy_pass http://192.168.174.129:80;
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr; 
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_connect_timeout 30;
    proxy_send_timeout 60;
    proxy_read_timeout 60;
    }
}
重新加载nginx配置文件
[root@nginx-server ~]# systemctl restart nginx
```

```shell
nginx proxy具体配置详解
proxy_pass ：真实服务器的地址，可以是ip也可以是域名和url地址
proxy_set_header：重新定义或者添加发往后端服务器的请求头
proxy_set_header X-Real-IP ：启用客户端真实地址（否则日志中显示的是代理在访问网站）
proxy_set_header X-Forwarded-For：记录代理地址

proxy_connect_timeout：后端服务器连接的超时时间发起三次握手等候响应超时时间
proxy_send_timeout：后端服务器数据回传时间，就是在规定时间之内后端服务器必须传完所有的数据
proxy_read_timeout ：nginx接收upstream（上游/真实） server数据超时, 默认60s, 如果连续的60s内没有收到1个字节, 连接关闭。像长连接
```

使用PC客户端访问nginx-2服务器地址，浏览器中输入http://192.168.174.130/ (也可以是nginx-2服务器的域名)

成功访问nginx-1服务器页面，观察nginx-1（192.168.174.129）服务器的日志

```shell
cat /var/log/nginx/access.log
```

# Nginx-2

```
upstream配置
```

```
负载均衡算法(分发策略)
轮询、ip_hash、url_hash、fair
```

```
配置实例
热备、轮询、加权轮询、ip_hash
```

```
nginx配置7层协议及4层协议方法
```

```
nginx会话保持
ip_hash、sticky_cookie_insert、jvm_route
```

```
nginx实现动静分离
```

```
nginx防盗链
```

```
nginx rewrite地址重写
if、rewrite、set、return、last，break
```

```
apache的https(rewrite)
location区段、location前缀、location配置
```

# Nginx-3

```
nginx日志配置
1、作用域
2、log_format指令
3、自定义日志格式的使用
4、error_log指令
5、rewrite_log指令
6、nginx日志轮转
```

```
nginx平滑升级
```

```
nginx错误页面配置
```

```
nginx流量限制
```

```
nginx访问控制
1、基于ip的访问控制
2、基于用户的信任登陆
```

# Nginx-4

```
自定义变量
nginx安装echo模块
内置预定义变量
```

```
nginx监控
nginx基础监控
监控的主要指标
(基本活跃指标、每秒请求数、服务器错误率、指标的收集)
nginx stub status监控模块安装
nginx状态查看
stub status参数说明
reqstat模块监控
nginx access log分析
```

# Nginx-5

```
Https
```

