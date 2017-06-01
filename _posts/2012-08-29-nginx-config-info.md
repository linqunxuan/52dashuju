---
title: Nginx1.0.15常用配置文件说明
date: 2012-08-29
categories: nginx
---

> Nginx is a web server, which can also be used as a reverse proxy, load balancer and HTTP cache.
>
> Wiki介绍Nginx是一个Web服务器，也可以用作反向代理，负载平衡器和HTTP缓存。
>
> 现在很多互联网公司都使用了Nginx大规模的去做Web服务器，下面介绍一下常用的配置文件信息，取材于nginx.1.0.15版本。

配置文件位于 NGINX_HOME/conf/目录下

### nginx.conf

``` nginx
#用户 用户组  
user       www www;  
#工作进程，根据硬件调整，有人说几核cpu，就配几个，我觉得可以多一点  
worker_processes  5；  
#错误日志  
error_log  logs/error.log;  
#pid文件位置  
pid        logs/nginx.pid;  
worker_rlimit_nofile 8192;  
  
events {  
#工作进程的最大连接数量，根据硬件调整，和前面工作进程配合起来用，尽量大，但是别把cpu跑到100%就行  
    worker_connections  4096;  
}  
  
http {  
    include    conf/mime.types;  
    #反向代理配置，可以打开proxy.conf看看  
    include    /etc/nginx/proxy.conf;  
    #fastcgi配置，可以打开fastcgi.conf看看  
    include    /etc/nginx/fastcgi.conf;  
  
    default_type application/octet-stream;  
    #日志的格式  
    log_format   main '$remote_addr - $remote_user [$time_local] $status '  
                      '"$request" $body_bytes_sent "$http_referer" '  
                      '"$http_user_agent" "$http_x_forwarded_for"';  
    #访问日志  
    access_log   logs/access.log  main;  
    sendfile     on;  
    tcp_nopush   on;  
    #根据实际情况调整，如果server很多，就调大一点  
    server_names_hash_bucket_size 128; # this seems to be required for some vhosts  
  
    #这个例子是fastcgi的例子，如果用fastcgi就要仔细看  
    server { # php/fastcgi  
        listen       80;  
        #域名，可以有多个  
        server_name  domain1.com www.domain1.com;  
        #访问日志，和上面的级别不一样，应该是下级的覆盖上级的  
        access_log   logs/domain1.access.log  main;  
        root         html;  
  
        location / {  
            index    index.html index.htm index.php;  
        }  
  
        #所有php后缀的，都通过fastcgi发送到1025端口上  
         #上面include的fastcgi.conf在此应该是有作用，如果你不include，那么就把fastcgi.conf的配置项放在这个下面。  
        location ~ /.php$ {  
            fastcgi_pass   127.0.0.1:1025;  
        }  
    }  
  
    #这个是反向代理的例子  
    server { # simple reverse-proxy  
        listen       80;  
        server_name  domain2.com www.domain2.com;  
        access_log   logs/domain2.access.log  main;  
  
        #静态文件，nginx自己处理  
        location ~ ^/(images|JavaScript|js|css|flash|media|static)/  {  
                root    /var/www/virtual/big.server.com/htdocs;  
                #过期30天，静态文件不怎么更新，过期可以设大一点，如果频繁更新，则可以设置得小一点。  
                expires 30d;  
        }  
  
        #把请求转发给后台web服务器，反向代理和fastcgi的区别是，反向代理后面是web服务器，fastcgi后台是fasstcgi监听进程，当然，协议也不一样。  
        location / {  
            proxy_pass      http://127.0.0.1:8080;  
        }  
    }  
  
    #upstream的负载均衡，weight是权重，可以根据机器配置定义权重。据说nginx可以根据后台响应时间调整。后台需要多个web服务器。  
    upstream big_server_com {  
        server 127.0.0.3:8000 weight=5;  
        server 127.0.0.3:8001 weight=5;  
        server 192.168.0.1:8000;  
        server 192.168.0.1:8001;  
    }  
  
    server {  
        listen          80;  
        server_name     big.server.com;  
        access_log      logs/big.server.access.log main;  
  
        location / {  
                proxy_pass      http://big_server_com;  
        }  
    }  
}  
```

### fastcgi.conf

没有必要改动它，用的时候include就行。

``` nginx
# fastcgi.conf  
fastcgi_param  SCRIPT_FILENAME    $document_root$fastcgi_script_name;  
fastcgi_param  QUERY_STRING       $query_string;  
fastcgi_param  REQUEST_METHOD     $request_method;  
fastcgi_param  CONTENT_TYPE       $content_type;  
fastcgi_param  CONTENT_LENGTH     $content_length;  
fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;  
fastcgi_param  REQUEST_URI        $request_uri;  
fastcgi_param  DOCUMENT_URI       $document_uri;  
fastcgi_param  DOCUMENT_ROOT      $document_root;  
fastcgi_param  SERVER_PROTOCOL    $server_protocol;  
fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;  
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;  
fastcgi_param  REMOTE_ADDR        $remote_addr;  
fastcgi_param  REMOTE_PORT        $remote_port;  
fastcgi_param  SERVER_ADDR        $server_addr;  
fastcgi_param  SERVER_PORT        $server_port;  
fastcgi_param  SERVER_NAME        $server_name;  
  
fastcgi_index  index.php;  
 
# PHP only, required if PHP was built with --enable-force-cgi-redirect  
fastcgi_param  REDIRECT_STATUS    200;  
```

### proxy.conf

``` nginx
# proxy.conf  
proxy_redirect          off;  
proxy_set_header        Host            $host;  
proxy_set_header        X-Real-IP       $remote_addr;  
proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;  
client_max_body_size    10m;  
client_body_buffer_size 128k;  
proxy_connect_timeout   90;  
proxy_send_timeout      90;  
proxy_read_timeout      90;  
proxy_buffers           32 4k;  
```

### mine.types

支持的mine类型

``` nginx
# mime.types  
types {  
    text/html                             html htm shtml;  
    text/css                              css;  
    text/xml                              xml rss;  
    image/gif                             gif;  
    image/jpeg                            jpeg jpg;  
    application/x-javascript              js;  
    text/plain                            txt;  
    text/x-component                      htc;  
    text/mathml                           mml;  
    image/png                             png;  
    image/x-icon                          ico;  
    image/x-jng                           jng;  
    image/vnd.wap.wbmp                    wbmp;  
    application/Java-archive              jar war ear;  
    application/mac-binhex40              hqx;  
    application/pdf                       pdf;  
    application/x-cocoa                   cco;  
    application/x-java-archive-diff       jardiff;  
    application/x-java-jnlp-file          jnlp;  
    application/x-makeself                run;  
    application/x-perl                    pl pm;  
    application/x-pilot                   prc pdb;  
    application/x-rar-compressed          rar;  
    application/x-redhat-package-manager  rpm;  
    application/x-sea                     sea;  
    application/x-shockwave-flash         swf;  
    application/x-stuffit                 sit;  
    application/x-tcl                     tcl tk;  
    application/x-x509-ca-cert            der pem crt;  
    application/x-xpinstall               xpi;  
    application/zip                       zip;  
    application/octet-stream              deb;  
    application/octet-stream              bin exe dll;  
    application/octet-stream              dmg;  
    application/octet-stream              eot;  
    application/octet-stream              iso img;  
    application/octet-stream              msi msp msm;  
    audio/mpeg                            mp3;  
    audio/x-realaudio                     ra;  
    video/mpeg                            mpeg mpg;  
    video/quicktime                       mov;  
    video/x-flv                           flv;  
    video/x-msvideo                       avi;  
    video/x-ms-wmv                        wmv;  
    video/x-ms-asf                        asx asf;  
    video/x-mng                           mng;  
}  
```

更多的，每个项的具体含义和配置选择，可以参考如下：

[NginxWiki](https://en.wikipedia.org/wiki/Nginx)

### © 著作权归作者所有，转载需联系作者

### END