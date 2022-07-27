# my-DNS
This is a test about self built authoritative DNS.

这是一个关于自建权威DNS服务器的测试。

## Requirement - 必要条件
* Go
* bind
* nginx

## Process - 实验过程
### 1、准备一个简单的后端
这里是用最简单的Gin框架下的后端，本地访问8080端口后返回“hello world”。
```
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "hello world",
		})
	})
	r.Run() // 监听并在 0.0.0.0:8080 上启动服务
}
```
### 2、在bind的工作目录下创建zone文件example.com.zone
```
$TTL 10M
@   IN    SOA    ns1.example.com    admin.example.com. (
        0       ; serial
        1D      ; refresh
        1H      ; retry
        1W      ; expire
        3H )    ; minimum

@    IN    NS   ns1.example.com.

ns1        A    192.168.44.134

www        A    192.168.44.134
```
### 3、修改配置文件named.conf（以下为参考配置）
```
logging {
 channel default_log {
        #这里注意提前创建log目录
        file "/var/log/named/named.log" versions 10 size 200m;
        severity dynamic;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    channel query_log {
        file "/var/log/named/query.log" versions 10 size 200m;
        severity dynamic;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    channel resolver_log {
        file "/var/log/named/resolver.log" versions 10 size 200m;
        severity dynamic;
        print-category yes;
        print-severity yes;
        print-time yes;
    };
    category default {default_log;};
    category queries {query_log;};
    category query-errors {query_log;};
    category resolver {resolver_log;};
};

options {
    #这里注意换成本机地址
    listen-on port 53 { 192.168.44.134; };
    #这里注意换成自己的文件位置
    directory "/etc/bind";
    dnssec-validation no;
    #支持递归查询
    recursion yes;
    #转发到公共DNS优先，而不是自己去迭代查询，节省网络IO资源消耗
    forward first;
    forwarders {
        223.5.5.5;
        223.6.6.6;
    };
    allow-query { any; };
};

zone "example.com" {
    type master;
    file "example.com.zone";
};
```
### 4、终端运行
```
dig @192.168.44.134 www.example.com
```
可获得输出数据
另，如终端运行
```
dig @192.168.44.134 www.baidu.com
```
由于未命中会自动查询当地的Local DNS，依然能获得数据。
### 5、nginx添加四层负责均衡
这里只是用作个人学习，因此不做复杂的负载均衡算法，只实现流量转发功能：
（1）将到达本机53端口的UDP报文转发给DNS服务器；

（2）将到达本机80端口的TCP报文转发到自定后端。

首先将DNS服务器专用的53端口腾出来
```
options {
    #这里的ip地址可以换成本机地址
    listen-on port 8053 { 192.168.44.134; };
    ......
};
```
修改nginx配置文件nginx.conf
```
stream {
    log_format proxy '$remote_addr [$time_local] '
                 '$protocol $status $bytes_sent $bytes_received '
                 '$session_time "$upstream_addr" '
                 '"$upstream_bytes_sent" "$upstream_bytes_received" "$upstream_connect_time"';

    access_log  /var/log/nginx/access.log  proxy;
    open_log_file_cache off;

    #注意修改为本机ip
    upstream dns_proxy {
        server 192.168.44.134:8053;
    }
    upstream hello_proxy {
        server 192.168.44.134:8080;
    }
    server {
        listen 53 udp reuseport;
        proxy_pass dns_proxy;
    }
    server {
        listen 80;
        proxy_connect_timeout 1s;
        proxy_timeout 300s;
        proxy_pass hello_proxy;
    }
}
```
执行dig命令，会在日志中看待UDP相关；

执行curl命令，会在日志中看到TCP相关。
### 6、七层负载均衡
七层负载均衡添加在四层和后端之间，监控880端口
```
upstream hello_proxy {
        server 10.227.89.58:880;
    }
```
修改配置文件
```
http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;
    upstream backend {
       server 10.227.89.58:8080;
    }
    server {
        listen       880;
        server_name  www.example.com;
        location / {
            proxy_pass http://backend;
             proxy_set_header HOST $host;
             proxy_connect_timeout 60;
             proxy_send_timeout 60;
             proxy_read_timeout 60;
        }
    }
}
```
### 到此完成权威DNS自建以及四层负载、七层负载的测试。