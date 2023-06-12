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

