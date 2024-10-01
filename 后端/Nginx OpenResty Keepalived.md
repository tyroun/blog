{% raw %}

# Nginx Lua 开发实战

## 第 1 章 Nginx 高效服务器

### 1.1 Nginx 的特点

1. 速度快
2. 扩展性好 - 核心+模块， Epoll/Kqueue 做分发， ngx_lua 开发，lua 的协程性能好
3. 高可靠性 - 管理进程 master + 若干个工作进程 worker
4. 低内存占用 - 10000 个非活跃的 HTTP 保活连接仅占用 2.5MB 内存
5. 高并发
6. 热部署
7. 开源

### 1.2 Nginx 的安装

#### 1 依赖库

1. GCC
2. PCRE - Perl Compatible Regular Expressions, Perl 兼容正则表达式
3. Zlib - 用于对 HTTP 报文内容做 gzip 格式压缩，如果在 nginx.conf 中配置了 gzip on，并指定某些 Content-Type 的 HTTP 应答包体使用 gzip
   进行压缩，以减小网络传输量，那么就需要在编译 Nginx 时指定 zlib，将其编译进 Nginx
4. OpenSSL - 使用 HTTPS 在 SSL 上传输 HTTP，就需要安装 OpenSSL。另外，在 ngx_lua 中使用 MD5、SHA1 等散列函数时，也需要安装 OpenSSL

#### 2 Linux 内核参数优化

修改/etc/sysctl.conf，常用配置如下：[插图]执行 sysctl -p 命令，使修改生效。

```shell
    fs.file-max = 999999
    net.core.netdev_max_backlog = 8096
    net.core.rmem_default = 262144
    net.core.wmem_default = 262144
    net.core.rmem_max = 2097152
    net.core.wmem_max = 2097152
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_keepalive_time = 600
    net.ipv4.tcp_fin_timeout = 30
    net.ipv4.tcp_max_tw_buckets = 5000
    net.ipv4.ip_local_port_range = 102461000
    net.ipv4.tcp_rmem = 409632768262142
    net.ipv4.tcp_wmen = 409632768262142
    net.ipv4.tcp_syncookies = 1
    net.ipv4.tcp_max_syn_backlog = 1024
```

执行 sysctl -p 命令，使修改生效。

各内核参数的作用如下。

1. fs.file-max：表示进程（在 Nginx 里指一个工作进程）可以同时打开的最大句柄数。本参数影响最大并发连接数。
2. net.ipv4.tcp_syncookies：解决 TCP 的 SYN 攻击。
3. net.ipv4.tcp_tw_reuse：参数为 1 时表示允许将 TIME_WAIT 状态的套接字重新用于新的 TCP 连接。服务器上的 TCP 协议栈在工作时会有大量的 TIME_WAIT
   状态连接，重新使用这些连接对于服务器处理大并发连接非常有用。
4. net.ipv4.tcp_keepalive_time：表示当 keepalive 启用时，TCP 发送 keepalive 消息的频率。默认为 2 小时，如果本值变小，可以更快地清理无效的连接。
5. net.ipv4.tcp_fine_timeout：表示服务器主动关闭连接时，套接字的 FIN_WAIT-2 状态最大时间
6. net.ipv4.tcp_max_tw_buckets：表示操作系统允许的 TIME_WAIT 套接字数量的最大值。当超过这个值，TIME_WAIT 状态的套接字被立即清除并输出警告消息，默认值为 180000，过多的
   TIME_WAIT 套接字会使服务器速度变慢。
7. net.ipv4.tcp_max_syn_backlog：表示 TCP 三次握手阶段 SYN 请求队列最大值，默认为 1024。调置为更大的值可以使 Nginx 在非常繁忙的情况下，若来不及接收新的连接时，Linux
   不至于丢失客户端新创建的连接请求。
8. net.ipv4.ip_local_port_range：定义 UDP 和 TCP 连接中本地端口范围（不包括连接到远端的端口）。
9. net.ipv4.tcp_rmen：定义 TCP 接收缓存（TCP 接收窗口）的最小值、默认值、最大值。
10. net.ipv4.tcp_wmen：定义 TCP 发送缓存（TCP 发送窗口）的最小值、默认值、最大值。
11. net.core.netdev_max_backlog：当网卡接收报文速度大于内核处理速度时，本参数设置这个缓冲队列最大值
12. net.core.rmem_default：表示内核套接字接收缓冲区默认值。
13. net.core.wmem_default：表示内核套接字发送缓冲区默认值。
14. net.core.rmem_max：表示内核套接字接收缓冲区最大值。
15. net.core.wmem_max：表示内核套接字发送缓冲区最大值。

#### 3 启动、停止、重启

启动命令如下：

```shell
/opt/nginx/sbin/nginx -p /opt/nginx/
/opt/nginx/sbin/nginx -p /opt/nginx -s stop
/opt/nginx/sbin/nginx -p /opt/nginx -s reload
```

## 第 3 章 OpenResty

### 3.2 OpenResty 的组成

1. 标准 Lua 5.1 解释器；
2. Drizzle Nginx 模块；
3. Postgres Nginx 模块；
4. Iconv Nginx 模块

上面 4 个模块默认并未启用，需要分别加入--with-lua51、--with-http_drizzle_module、--with-http_postgres_module 和--with-http_iconv_module
编译选项开启它们

运行下面命令启动 Nginx：

```shell
/usr/local/openresty/nginx/sbin/nginx -p /usr/local/openresty/nginx/
```

在浏览器里输入http://127.0.0.1（或主机IP），看到“Welcome to OpenResty！ ”表示已经启动成功。

可以进一步修改/usr/local/openresty/nginx/conf/nginx.conf，测试 Lua 是否正常工作，nginx.conf 内容如下

```nginx
    worker_processes  1;
    error_log logs/error.log;
    events {
        worker_connections 1024;
    }
    http {
        server {
            listen 8080;
            location / {
                default_type text/html;
                content_by_lua '
                    ngx.say("<p>hello, world</p>")
                ';
            }
        }
    }
```

重载配置文件

```shell
/usr/local/openresty/nginx/sbin/nginx -p /usr/local/openresty/nginx/ -s reload
```

测试一下配置文件的正确性

```shell
    /usr/local/openresty/nginx/sbin/nginx -p /usr/local/openresty/nginx/ -t
```

### 3.4 Nginx 多实例

只需要把 OpenRestry 中的 Nginx 目录复制一份就可以启动不同的实例：

```shell
cp -r /usr/local/openrestry/nginx  /usr/local/openrestry/nginx_9090
```

然后修改 nginx_9090/conf/nginx.conf，把端口从 8080 修改为 9090，把“hello world”修改为“hello world2”，修改完成后启动实例。

```shell
/usr/local/openrestry/nginx_9090/sbin/nginx -p /usr/local/openrestry/nginx_9090/
```

## 第 4 章 Nginx 核心技术

### 4.2 Nginx 架构

Nginx 使用事件驱动的服务模型。把一个处理过得的过程（如 HTTP 请求）划分为 7 个、9 个或 11 个阶段，每一个阶段都异步处理，将请求和处理结果异步化处理。

管理进程和工作进程的机制使 Nginx 可以充分利用多处理器机制，充分利用了 SMP 机制的硬件资源

对于高并发下的多工程进程容易引起的“惊群”以及负载均衡问题，Nginx 都做了比较好的处理

#### 4.2.1 事件驱动

描述 Nginx 事件处理的模型，即事件、事件管理器和事件消费者之间的一个概貌

![image-20230905114040251](../image/Nginx OpenResty Keepalived/image-20230905114040251.png)

#### 4.2.2 异步多阶段处理

获取一个静态文件的 HTTP 请求可以划分为下面的几个阶段

1. 建立 TCP 连接阶段：收到 TCP 的 SYN 包。
2. 开始接收请求：接收到 TCP 中的 ACK 包表示连接建立成功。
3. 接收到用户请求并分析请求是否完整：接收到用户的数据包。
4. 接收到完整用户请求后开始处理：接收到用户的数据包。
5. 由静态文件读取部分内容：接收到用户数据包或接收到 TCP 中的 ACK 包，TCP 窗口向前划动。这个阶段可多次触发，直到把文件完全读完。不一次性读完是为了避免长时间阻塞事件分发器。
6. 发送完成后：收到最后一个包的 ACK。对于非 keepalive 请求，发送完成后主动关闭连接。
7. 用户主动关闭连接：收到 TCP 中的 FIN 报文

#### 4.2.3 模块化设计

![image-20230905114437079](../image/Nginx OpenResty Keepalived/image-20230905114437079.png)

#### 4.2.4 管理进程、工作进程设计

![image-20230905114515568](../image/Nginx OpenResty Keepalived/image-20230905114515568.png)

##### 1“惊群”问题

多个进程监听同一个端口，连接请求来的时候，开始抢夺事件 - 惊群

在 epoll 模式下，内核在收到了 TCP 的 SYN
包时，会激活所有休眠的工作进程，最先接收连接请求的工作进程可以成功建立新连接，其他工作进程的接收会失败。这些失败的唤醒是不必要的，引发了不必要的进程上下文切换，增加了系统开销，这就是“惊群”问题

Nginx 应用层制定了一个机制解决这个问题：规定同一时刻只能有唯一一个工作进程监听 Web
端口，这样，新的连接事件只能唤醒唯一一个工作进程。内部的实现实际上是使用了一个进程间的同步锁，工作进程每次唤醒都先尝试这把锁，保证同一时间只有一个工作进程可以进入锁，获得锁的进程设置监听连接的读事件，以处理未来的新连接请求，并处理已连接上的事件；未能进入监听锁的工作进程则不监听新连接事件，只处理已连接上的事件，将唤醒的工作进程分为了两类，一类（只有
1 个）是可以监听新连接的，另一类是正常处理已有连接请求的。

就是有个队列，只有一个进程监听端口，被换醒后马上释放锁，让另一个进程监听

##### 2 负载均衡

**系统级的负载均衡**

1. 系统级的负载均衡，实现方法是使用一个 Nginx 服务通过 upstream 机制将请求分配到上游后端服务器，而这里可以使用模块内置的一些负载均衡机制将请求均衡地分配到服务器组中。
2. 使用一个单独的 Nginx 服务以自定义负载均衡算法实现代理模式，实现负载均衡集群。

**单 Nginx 服务内部工作进程间的负载均衡**

不让某个进程“累死”，其他的进程“闲死”，才能发挥系统的最佳性能

#### 4.2.11 keepalive

keepalive 是 HTTP 长连接

请求 request：

POST 的 Content-Length 表示 body 大小

应答 response:

HTTP 1.0 - Content-Length 表示 body 长，如果响应头中没有 Content-Length 头，则客户端会一直接收数据，直到服务端主动断开连接，才表示 body 接收完

HTTP 1.1 -

​ Transfer-encoding: chunked 表示是流式传输，body 会被分成多个块，每块的开始会标识出当前块长度

​ Transfer-encoding != chunked，则按照 Content-Length 接收数据。如果 Content-Length 也没有，则是长连接

如果客户端的 request header 中的 connection: close，则表示客户端需要关掉长连接。如果客户端的请求头中的 connection: keepalive，则客户端需要打开长连接

服务器 response 的 header 里 Keep-Alive: timeout=60 表示超时时间, Connection: keep-alive 表示打开长连接

## 第 9 章 nginx.conf 文件配置

### 9.1 默认 nginx.conf 文件

Nginx 提供了一个默认的 nginx.conf 模板，里面包含了一个 HTTP 服务配置块、一个 HTTPS 配置块和主要的全局配置项

```nginx
    #user  nobody;
    worker_processes  1;


    #error_log  logs/error.log;
    #error_log  logs/error.log  notice;
    #error_log  logs/error.log  info;


    #pid logs/nginx.pid;


    events {
        worker_connections  1024;
    }


    http {
        include        mime.types;
        default_type  application/octet-stream;


        #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
        #                     '$status $body_bytes_sent "$http_referer" '
        #                     '"$http_user_agent""$http_x_forwarded_for"';


        #access_log  logs/access.log  main;


        sendfile          on;
        #tcp_nopush      on;


        #keepalive_timeout  0;
        keepalive_timeout  65;


        #gzip  on;


        server {
            listen     80;
            server_name  localhost;


            #charset koi8-r;


            #access_log  logs/host.access.log  main;


            location / {
                root    html;
                index  index.html index.htm;
            }


            #error_page  404                 /404.html;


            # redirect server error pages to the static page /50x.html
            #
            error_page    500502 503504  /50x.html;
            location = /50x.html {
                root    html;
            }


            # proxy the PHP scripts to Apache listening on 127.0.0.1:80
            #
            #location ～ \.php$ {
            #     proxy_pass    http://127.0.0.1;
            #}


            # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
            #
            #location ～ \.php$ {
            #     root            html;
            #     fastcgi_pass  127.0.0.1:9000;
            #     fastcgi_index index.php;
            #     fastcgi_param SCRIPT_FILENAME  /scripts$fastcgi_script_name;
            #     include        fastcgi_params;
            #}


            # deny access to .htaccess files, if Apache's document root
            # concurs with nginx's one
            #
            #location ～ /\.ht {
            #     deny  all;
            #}
        }


        # another virtual host using mix of IP-, name-, and port-based configuration
        #
        #server {
        #     listen     8000;
        #     listen     somename:8080;
        #     server_name  somename  alias  another.alias;


        #     location / {
        #          root    html;
        #          index  index.html index.htm;
        #     }
        #}


        # HTTPS server
        #
        #server {
        #     listen     443 ssl;
        #     server_name  localhost;


        #     ssl_certificate     cert.pem;
        #     ssl_certificate_key  cert.key;


        #     ssl_session_cache     shared:SSL:1m;
        #     ssl_session_timeout  5m;


        #     ssl_ciphers  HIGH:！ aNULL:！ MD5;
        #     ssl_prefer_server_ciphers  on;


        #     location / {
        #          root    html;
        #          index  index.html index.htm;
        #     }
        #}
}
```

### 9.2 nginx.conf 示例

```nginx
    user  root;
    worker_processes  4;
    worker_rlimit_nofile 1000000;


    error_log  logs/error.log;
    #error_log  logs/error.log  notice;
    #error_log  logs/error.log  info;


    #pid          logs/nginx.pid;


    events {
        use  epoll;
        worker_connections  300000;
    }


    http {
        include        mime.types;
        default_type  text/html;


        #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
        #                     '$status $body_bytes_sent "$http_referer" '
        #                     '"$http_user_agent""$http_x_forwarded_for"';


        access_log    off;
        server_tokens off;


        sendfile      on;
        tcp_nopush    on;
        tcp_nodelay  on;
        open_file_cache max=10240 inactive=60s;
        open_file_cache_valid 80s;
        open_file_cache_min_uses 1;
        lua_shared_dict gkey 50m;
        lua_shared_dict gpost 10m;
        lua_shared_dict gvar 80m;
        lua_shared_dict msg_queue 300m;
        lua_shared_dict gsqs 100m;
        lua_shared_dict gex_session 50m;


        keepalive_timeout  0;
        #keepalive_timeout  600s;
        #keepalive_requests 10000;
        chunked_transfer_encoding off;
        #gzip  on;
        lua_package_path "/usr/local/lib/lua/5.1/?.lua; ; ";


        upstream bk_mysql {
            drizzle_server 10.185.220.120:3306 protocol=mysql dbname=test user=he
            password=33Er3～#;
            drizzle_keepalive max=300 overflow=reject mode=single;
        }
        upstream bk_master_db {
                drizzle_server  127.0.0.1:3306  protocol=mysql  dbname=test  user=he password=33Er3～#;
            drizzle_keepalive max=100 overflow=reject mode=single;
        }
        upstream bk_redis {
            server 10.185.220.120:6009;
            # a pool with at most 1024 connections
            # and do not distinguish the servers:
            keepalive 1000;
        }


        upstream bk_svr_conf {
            server 10.195.194.47:9001;
            keepalive 1000;
        }


        server {
            listen        9500 default so_keepalive=on;
            server_name  10.185.194.47;
            set $pub_ip "120.26.57.240:9510";
            set $idm "121.40.249.246:8300";


            #charset koi8-r;
            charset utf-8;
            #chunked_transfer_encoding off;


            #access_log  logs/host.access.log  main;


            location /redis_set_ex {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                set $expire $arg_expire;
                redis2_query set $key $request_body;
                redis2_query expire $key $expire;
                redis2_pass bk_redis;
            }


            location /redis_expire {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                set $expire $arg_expire;
                redis2_query expire $key $expire;
                redis2_pass bk_redis;
            }


            location /redis_persist {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                redis2_query persist $key;
                redis2_pass bk_redis;
            }


            location /redis_set1 {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                redis2_query set $key $request_body;
                redis2_query expire $key 86400;
                redis2_pass bk_redis;
            }


            location /redis_get {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                redis2_query get $key;
                #redis2_query expire $key 86400;
                redis2_pass bk_redis;
            }


            location /redis_del {
                include /usr/local/ip_limit.conf;
                set $key $arg_key;
                redis2_query del $key;
                redis2_pass bk_redis;
            }


            location /mt_redis_set_ex {
                include /usr/local/ip_limit.conf;
                #access_by_lua_file access.lua;
                content_by_lua_block {'
                    local val=ngx.unescape_uri(ngx.var.arg_val)
                    local resp = ngx.location.capture("/redis_set_ex?key=" .. ngx.var.arg_key .. "&expire=" .. ngx.var.arg_expire, {
                    	method = ngx.HTTP_POST, body = val
                    })
                    ngx.exit(resp.status)
                    }
            }


            location /user_status {
                include /usr/local/ip_limit.conf;
                default_type 'text/plain';
                lua_need_request_body on;
                client_max_body_size 50k;
                client_body_buffer_size 50k;
                content_by_lua "local srcid
                if ngx.var.arg_uid == nil then
                ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
            else
                srcid=ngx.var.arg_uid
            end
            local gvar = ngx.shared.gvar
            local user_status = gvar:get('user_status_' .. srcid)
            if user_status == nil then
                ngx.print('0')
            else
                ngx.print(user_status)
            end
    ";
            }


            location /user_offline {
                include /usr/local/ip_limit.conf;
                default_type 'text/plain';
                lua_need_request_body on;
                client_max_body_size 50k;
                client_body_buffer_size 50k;
                content_by_lua "local srcid
                if ngx.var.arg_uid == nil then
                    local gvar = ngx.shared.gvar
                    local gex_session = ngx.shared.gex_session
                    local keys = gex_session:get_keys(0)
                    local k, v
                        for k, v in ipairs(keys) do
                        gvar:set('user_session_' .. v, nil)
                        gvar:set('user_status_' .. v, nil)
                        gex_session:set(v, nil)
                        local resp = ngx.location.capture('/redis_del?key=S_' .. v)
                        ngx.say('cleanup session:' .. v)
                    end
                    ngx.exit(ngx.HTTP_OK)
                else
                    srcid=ngx.var.arg_uid
                end
                local gvar = ngx.shared.gvar
                local session_key = gvar:get('user_session_' .. srcid)
                if session_key ～= nil then
                gvar:set('user_session_' .. srcid, nil)
                local resp = ngx.location.capture('/redis_del?key=S_' .. srcid)
                end
                local user_status = gvar:get('user_status_' .. srcid)
                if user_status ～= nil then
                gvar:set('user_status_' .. srcid, nil)
                end
                ";
            }


            location /msg_count {
                include /usr/local/ip_limit.conf;
                default_type 'text/plain';
                lua_need_request_body on;
                client_max_body_size 50k;
                client_body_buffer_size 50k;
                content_by_lua "local srcid
                if ngx.var.arg_uid == nil then
                ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
                else
                srcid=ngx.var.arg_uid
                end
                local msg_queue = ngx.shared.msg_queue
                ngx.print('unread:' .. msg_queue:llen('user_msg_' .. srcid))
                ";
            }
    }
```

### 9.3 全局配置与顶层配置块

#### 9.3.1 main 全局配置

##### 1．worker_processes

工作进程数

```nginx
    worker_processes number | auto;   #语法
    worker_process 1;				 #示例
```

在配置文件的全局 main 部分，管理进程接收任务并将请求分配给工作进程处理，工作进程是实际的处理进程。工作进程的个数可以设置为 CPU 的核数

##### 2．worker_cpu_affinity

绑定工作进程到指定的 CPU 内核

```nginx
    worker_cpu_affinity cpumask[cpumask…] #语法

    worker_processes 4;
    worker_cpu_affinity 1000010000100001; #示例
```

如果 CPU 非常繁忙，不一定会把每一个工作进程分配到一个核心上，通过本指令手工指定会得到真正的并发（仅对 Linux 有效）

##### 3．worker_rlimit_nofile

工作进程最大打开文件数

```nginx
    worker_rlimit_nofile number;
```

##### 4．worker_directory

工作进程的当前工作路径，主要用于生成 coredump 文件，工作进程所在用户和组需要对工作目录有写权限

```nginx
    worker_directory directory;
```

##### 5．worker_priority

工作进程优先级，取值范围为-20 ～+20

```nginx
    worker_priority number;
```

##### 6. worker_rlimit_core

coredump 文件最大尺寸

```nginx
    worker_rlimit_core size;
```

##### 7．daemon

是否以守护进程方式运行 Nginx

```nginx
    daemon on|off;
```

##### 8．master_process

是否以 master_process 方式工作, 指明工作进程是否马上启动，主要用于 Nginx 深度开发使用，不是常规配置功能项

```nginx
    master_process on|off;
```

##### 9. error_log

error 日志设置, 设置日志文件路径名，还可以设置要写入的错误级别

```nginx
    error_log /path/file level;
    error_log logs/error.log err;
```

##### 10．env

定义环境变量

```nginx
    env VAR|VAR=Value;

    env MACCOC_OPTIONS;
    env PERL5LIB=/data/site/modules;
    env OPENSSL_ALLOW_PROXY_CERTS=1;
```

##### 11．include

引用其他配置文件，文件名可以是绝对路径，也可以是相对路径，如果是相对路径，就是 nginx.conf 所在的路径

```nginx
    include /path/file;
```

##### 12．lock_file

锁定文件，Nginx 使用锁定文件实现 accept_mutex。在多数系统上，这个锁用原子操作实现，则这个值就被忽略掉了。这个配置在使用 lock file 机制的系统上使用

```nginx
    lock_file file;
```

##### 13．pid

设置 pid 文件路径，设置保存管理进程 ID 的文件，用于使用进程 ID 操作 Nginx 的环境

```nginx
    pid path/file;
```

##### 14．user

设置工作进程运行时的用户及用户组

```nginx
    user username[groupname];
```

##### 16．thread_pool

工作进程中多线程读和写的线程池，定义工作进程中对文件读和写时用到的线程池。threads 参数定义线程池中线程的数量。max_queue 限制允许在队列中等待的任务，默认是 65536 个任务。队列超出时，任务将返回一个错误信息

```nginx
    thread_pool name threads=number [max_queue=number];

    thread_pool default threads=32 max_queue=65535;
```

#### 9.3.2 events 配置块

events 模块中包含 Nginx 中所有处理连接的设置。events 是 Nginx 使用到的 I/O 事件模型，是最主要的进出交互部分。Nginx 强大的部分就是在 Linux 上完美实现了 epoll 模型。

常用配置项如下：

```nginx
    events{
        use epoll;
        worker_connections 20000;
    }
```

#### 9.3.3 http 服务器配置块

HTTP 模块中一般使用 HTTP 全局配置参数控制整体行为，使用 server 配置虚拟主机，包含监听地址、文档路径和各种 location

反向代理、负载均衡等都是在内部的 server 等模块实现的。同时在各个子配置块或 location 等内部划分了许多阶段（phase），这些阶段可以注册 Lua 代码或 Lua 文件，干预处理的过程

一个典型的 Web 服务器会包含全局配置、多个 server 块和多个 location 块

```nginx
    http{
        gzip on;


        upstream{
                …
        }
        …
        server{
            listen localhost:80;
            …
            location /webstatic {
                if … {
                    …
                }
                root /opt/webresource;
                …
            }
            location ～＊ .(jpg|jpeg|png|jpe|gif)${
                …
            }
        }
        server{
            …
        }
    }
```

##### 1．listen

监听端口

```nginx
    listen  address[:port]  [default_server]  [ssl]  [http2  |  spdy]  [proxy_protocol] [setfib=number]  [fastopen=number]  [backlog=number]  [rcvbuf=size]  [sndbuf=size] [accept_filter=filter] [deferred] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];
    listen port [default_server] [ssl] [http2 | spdy] [proxy_protocol] [setfib=number] [fastopen=number] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [ipv6only=on|off] [reuseport] [so_keepalive=on|off|[keepidle]:[kee pintvl]:[keepcnt]];
    listen  unix:path  [default_server]  [ssl]  [http2  |  spdy]  [proxy_protocol] [backlog=number] [rcvbuf=size] [sndbuf=size] [accept_filter=filter] [deferred] [bind] [so_keepalive=on|off|[keepidle]:[keepintvl]:[keepcnt]];

    listen *:80 | *:8000;
    listen [::]:8000;   #IPv6
    listen [::1];		#IPv6
    listen 127.0.0.1 default_server accept_filter=dataready backlog=1024;
```

主要参数的意义如下：

1. default：将所在的 server 块作为整个 Web 服务的默认 server 块。如果没有设置这个参数，将以找到的第一个 server 块为默认 server 块。
2. default_server：同 default。
3. backlog=num：表示 TCP 中 backlog 队列大小，默认为-1，表示不设置。在 TCP 三次握手过程中，进程还没有开始处理监听句柄，backlog 队列就会放置这些新连接。如果队列已满，新的客户尝试连接，则会失败。
4. rcvbuf=size：设置 so_rcvbuf 接收缓冲区大小。
5. sndbuf=size：设置 so_sndbuf 发送缓冲区大小。
6. accept_filter：设置 accept 过滤器，只对 FreeBSD 操作系统有用。
7. deferred：设置本参数后，客户端建立连接，并且完成了三次握手，也不会调度工作进程来处理，直到客户端实际请求数据到来才分配工作进程处理，适用于大并发情况下减轻工作进程负担。
8. bind：绑定当前 ip/port 对，只有在一个端口监听多个地址时才会生效。
9. ssl：在当前监听的端口上建立的连接必须使用 SSL 协议。

##### 2．server_name

主机名称，server_name 后可以跟多个主机名称

```nginx
    server_name name[...];

    server_name www.google.com mail.google.com;
```

##### 3. location

```nginx
    location[=|～|～＊|^～|@]/uri/{...}
```

location 尝试根据用户请求中的 URI 匹配上面的/uri 表达式，如果匹配，就选择 location 中的配置处理用户请求

| 模式                 | 含义                                                                             |
| :------------------- | :------------------------------------------------------------------------------- |
| location = /uri      | = 表示精确匹配，只有完全匹配上才能生效                                           |
| location ^~ /uri     | ^~ 开头对 URL 路径进行前缀匹配，并且在正则之前。                                 |
| location ~ pattern   | 开头表示区分大小写的正则匹配                                                     |
| location ~\* pattern | 开头表示不区分大小写的正则匹配                                                   |
| location /uri        | 不带任何修饰符，也表示前缀匹配，但是在正则匹配之后                               |
| location /           | 通用匹配，任何未匹配到其它 location 的请求都会匹配到，相当于 switch 中的 default |

多个 location 配置的情况下匹配顺序为:

- 首先精确匹配 `=`
- 其次前缀匹配 `^~`
- 其次是按文件中顺序的正则匹配
- 然后匹配不带任何修饰的前缀匹配。
- 最后是交给 `/` 通用匹配
- 当有匹配成功时候，停止匹配，按当前匹配规则处理请求

前缀匹配，如果有包含关系时，按最大匹配原则进行匹配。比如在前缀匹配：`location /dir01` 与 `location /dir01/dir02`，如有请求 `http://localhost/dir01/dir02/file `将最终匹配到 `location /dir01/dir02`

##### 4．root

```nginx
    root path
    location /download/{
        root /opt/web/html/;
    }
```

定义资源文件相对于 HTTP 请求的根目录

如果有一个 URI 是/download/index/test.html，那么 Web 服务器会返回服务器上/opt/web/html/download/index/test.html 文件的内容

##### 5．alias

以别名方式设置资源路径

alias 是用来设置文件资源路径的，与 root 的不同点在于如何解读 location 后面的 uri 参数，alias 和 root 会以不同的方式将用户请求映射到真正的磁盘文件上。例如，有一个请求的 URI
是/conf/nging.conf，而实际文件在/usr/local/nginx/conf/nginx.conf，那么可以使用下面的方式设置

```nginx
	# alias 方式
    location /conf{
        alias /usr/local/nginx/conf/;
    }
	# root 方式
    location /conf{
        root /usr/local/nginx
    }
	# alias添加正则
    location ～ ^/test/(\w+)\.(\w+)${
        alias /usr/local/nginx/$2/$1.$2;
    }
```

##### 6．index

首页，如果访问站点的 URI 是/，一般返回网站首页。index 后面可以跟多个参数，Nginx 按照顺序访问这些文件

```nginx
    index file ...;
    index index.html;
    location /{
        root path;
        index /index.html /html/index.php /index.asp;
    }
```

##### 7. error_page

根据 HTTP 返回码重定向页面

```nginx
    error_page code[code...][=|=answer-code]uri|@named_location

    error_page 404              /404.html;
    error_page 502503 504     /50x.html;
    error_page 403              http://example.com/forbidden.html;
    error_page 404              = @fetch;

    #虽然重定向了URI，但返回的HTTP错误码还是原来的值，可以使用=更改返回的错误码
    error_page 404 =200 /empty.gif;
    error_page 404 =403 /forbidden.gif;

	#如果不想修改URI，只想重定向到另外一个location中处理
    location / {
        error_page 404 @fallback;
    }


    location @fallback{
        proxy_pass http://backend;
    }
```

##### 9 try_files

```nginx
    try_files path1 [path2] uri;

    try_files /system/maintenance.html $uri $uri/index.html $uri.html @other;
    location @other{
        proxy_pass http://backend;
    }

    #与error_page配合使用
    location / {
        try_files $uri $uri/ /error.php?c=404 =404;
    }
```

说明：try_fies 后要跟若干路径，最后必须要有 uri 参数，表示尝试按照顺序访问每一个 path，如果可以有效地读取，就直接向用户返回这个 path 对应的文件并结束请求，否则继续向下访问。如果都找不到，就定向到最后的 uri
上，所以这个 uri 必须存在，而且应该是可以有效重定向的

##### 25 keepalive 超时时间

```nginx
    keepalive_timeout time;
```

一个 keepalive 连接在闲置超过一定时间后，服务器和浏览器都会关闭这个连接。这个值是用于限制 Nginx 服务器的，Nginx 会把这个值传给浏览器，但每个浏览器对待 keepalive 的策略可能是不同的

# Keepalived 主从环境部署

## Bare metal 环境

### 1 安装 keepavlived

```shell
sudo apt-get install keepalived
```

### 2 修改配置文件

```shell
# 在/home/test下
git clone git@cn-git.netint.ca:ojo-system/gateway.git
cp conf/keepalived.conf /etc/keepalived
```

修改 keepalived.conf 的本机网卡和虚拟 IP

```json
vrrp_instance redis {
  state
  BACKUP
  interface
  enp5s0
  //这个改成本机网卡
  priority
  100
    advert_int 1
    virtual_ipaddress {
       10.20.130.94           //这行改成相应的虚拟地址
    }
}
```

### 3 修改 mysql 的设置

/etc/mysql/mysql.conf.d/mysqld.cnf

```
bind-address = 0.0.0.0
server-id = xxx  # 这个ID确保主从服务器要不一样
validate_password=off         #关闭密码安全策略
default_password_lifetime=0     #设置密码不过期
```

配置 mysql 免密登录

```shell
mysql_config_editor set --login-path=local --user=root --port=3306 --password
mysql_config_editor print --all
用root登录mysql
GRANT SYSTEM_USER ON *.* TO 'root'@'%';
```

### 4 配置主从机免密 ssh 访问

如果是裸机环境，以下 ssh 需要拷贝到/root/.ssh/下

```shell
# 在MachineA上
ssh-keygen -t rsa
# 在MachineB上
ssh-keygen -t rsa

# 在MachineA上
scp ~/.ssh/id_rsa.pub userB@MachineB:~/MachineA.pub
# 在MachineB上
scp ~/.ssh/id_rsa.pub userA@MachineA:~/MachineB.pub

# 在MachineA上
cat ~/MachineB.pub >> ~/.ssh/authorized_keys
# 在MachineB上
cat ~/MachineA.pub >> ~/.ssh/authorized_keys

# 在两台机器上执行
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 5 修改运行脚本

scripts/.mysqlenv

```shell
REMOTE_IP=10.20.130.17  #这个改成另一台服务器IP
```

启动 keepalived

```shell
service keepalived restart
```

## Mysql 主从机制详解

### 1 实现原理

异步模式是一种基于偏移量的主从复制，**实现原理是:**

1. 主库开启 binlog 功能并授权从库连接主库
2. 从库通过 change master 得到主库的相关同步信息然后连接主库进行验证，
3. 主库 IO 线程根据从库 slave 线程的请求，从 master.info 开始记录的位置点向下开始取信息，同时把取到的位置点和最新的位置与 binlog 信息一同发给从库 IO 线程,
4. 从库将相关的 sql 语句存放在 relay-log 里面，最终从库的 sql 线程将 relay-log 里的 sql 语句应用到从库上

所谓异步模式指的是 MySQL 主服务器上 I/O thread 线程将二进制日志写入 binlog 文件之后就返回客户端结果，不会考虑二进制日志是否完整传输到从服务器以及是否完整存放到从服务器上的 relay 日志中，这种模式一旦主服务(
器)宕机，数据就可能会发生丢失。

### 2 三种复制模式

**从 MySQL5.5 开始，MySQL 以插件的形式支持半同步复制**。先来区别下 mysql 几个同步模式概念：

**异步复制（Asynchronous replication）**
MySQL 默认的复制即是异步的，主库在执行完客户端提交的事务后会立即将结果返给给客户端，并不关心从库是否已经接收并处理，这样就会有一个问题，主如果 crash
掉了，此时主上已经提交的事务可能并没有传到从上，如果此时，强行将从提升为主，可能导致新主上的数据不完整。

**全同步复制（Fully synchronous replication）**
指当主库执行完一个事务，所有的从库都执行了该事务才返回给客户端。因为需要等待所有从库执行完该事务才能返回，所以全同步复制的性能必然会收到严重的影响。

**半同步复制（Semisynchronous replication）**
介于异步复制和全同步复制之间，主库在执行完客户端提交的事务后不是立刻返回给客户端，而是等待至少一个从库接收到并写到 relay log
中才返回给客户端。相对于异步复制，半同步复制提高了数据的安全性，同时它也造成了一定程度的延迟，这个延迟最少是一个 TCP/IP 往返的时间。所以，**半同步复制最好在低延时的网络中使用**。

### 3 半同步机制具体特性

\- 从库会在连接到主库时告诉主库，它是不是配置了半同步。 \-
如果半同步复制在主库端是开启了的，并且至少有一个半同步复制的从库节点，那么此时主库的事务线程在提交时会被阻塞并等待，结果有两种可能，要么至少一个从库节点通知它已经收到了所有这个事务的 Binlog
事件，要么一直等待直到超过配置的某一个时间点为止，而此时，半同步复制将自动关闭，转换为异步复制。 \- 从库节点只有在接收到某一个事务的所有 Binlog，将其写入并 Flush 到 Relay Log
文件之后，才会通知对应主库上面的等待线程。 \- 如果在等待过程中，等待时间已经超过了配置的超时时间，没有任何一个从节点通知当前事务，那么此时主库会自动转换为异步复制，当至少一个半同步从节点赶上来时，主库便会自动转换为半同步方式的复制。
\- 半同步复制必须是在主库和从库两端都开启时才行，如果在主库上没打开，或者在主库上开启了而在从库上没有开启，主库都会使用异步方式复制。

### 4 半同步复制的潜在问题

1. 事务还没发送到从库上
   此时，客户端会收到事务提交失败的信息，客户端会重新提交该事务到新的主上，当宕机的主库重新启动后，以从库的身份重新加入到该主从结构中，会发现，该事务在从库中被提交了两次，一次是之前作为主的时候，一次是被新主同步过来的。
2. 事务已经发送到从库上
   此时，从库已经收到并应用了该事务，但是客户端仍然会收到事务提交失败的信息，重新提交该事务到新的主上。

**无数据丢失的半同步复制**
针对上述潜在问题，MySQL 5.7 引入了一种新的半同步方案：Loss-Less 半同步复制。针对上面这个图，"Waiting Slave dump"被调整到"Storage Commit"之前。当然，之前的半同步方案同样支持，MySQL
5.7.2 引入了一个新的参数进行控制: rpl_semi_sync_master_wait_point, 这个参数有两种取值：1) **AFTER_SYNC** , 这个是新的半同步方案，Waiting Slave dump 在
Storage Commit 之前。2) AFTER_COMMIT, 这个是老的半同步方案。

### 5 半同步复制部署

要想使用半同步复制，必须满足以下几个条件： 1）MySQL 5.5 及以上版本 2）变量 have_dynamic_loading 为 YES （查看命令：show variables like "have_dynamic_loading"
;） 3）主从复制已经存在 (即提前部署 mysql 主从复制环境,主从同步要配置基于整个数据库的，不要配置基于某个库的同步，即同步时不要过滤库)

#### 5.1 加载插件

因用户需执行 INSTALL PLUGIN, SET GLOBAL, STOP SLAVE 和 START SLAVE 操作，所以用户需有 SUPER 权限。

半同步复制是一个功能模块，库要能支持动态加载才能实现半同步复制! (安装的模块存放路径为/usr/local/mysql/lib/plugin） 主数据库执行:

```mysql
mysql> INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';

[要保证/usr/local/mysql/lib/plugin/目录下有semisync_master.so文件 (默认编译安装后就有)]
---------------------------------------------------------------------------------------
如果要卸载(前提是要关闭半同步复制功能)，就执行
mysql> UNINSTALL PLUGIN rpl_semi_sync_master;
```

从数据库执行:

```mysql
mysql> INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
[要保证/usr/local/mysql/lib/plugin/目录下有semisync_slave.so文件 (默认编译安装后就有)]
---------------------------------------------------------------------------------------
如果要卸载(前提是要关闭半同步复制功能)，就执行
mysql> UNINSTALL PLUGIN rpl_semi_sync_slave;
```

#### 5.2 查看插件是否加载成功

```mysql
mysql> show plugins;
........
| rpl_semi_sync_master    | ACTIVE  | REPLICATION    | semisync_master.so | GPL   |
```

#### 5.3 启动半同步复制

主数据库执行:

```mysql
mysql> SET GLOBAL rpl_semi_sync_master_enabled = 1;
```

从数据库执行:

```mysql
mysql> SET GLOBAL rpl_semi_sync_slave_enabled = 1;
```

主数据库的 my.cnf 配置文件中添加:

```mysql
plugin-load=rpl_semi_sync_master=semisync_master.so
rpl_semi_sync_master_enabled=1
```

从数据库的 my.cnf 配置文件中添加:

```mysql
plugin-load=rpl_semi_sync_slave=semisync_slave.so
rpl_semi_sync_slave_enabled=1
```

在个别高可用架构下，master 和 slave 需同时启动，以便在切换后能继续使用半同步复制！即在主从数据库的 my.cnf 配置文件中都要添加:

```mysql
plugin-load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
rpl-semi-sync-master-enabled = 1
rpl-semi-sync-slave-enabled = 1
```

#### 5.4 重启从数据库上的 IO 线程

```mysql
mysql> STOP SLAVE IO_THREAD;
mysql> START SLAVE IO_THREAD;
```

**特别注意:** 如果没有重启，则默认的还是异步复制模式！

#### 5.5 查看半同步是否在运行

主数据库:

```mysql
mysql> show status like 'Rpl_semi_sync_master_status';
+-----------------------------+-------+
| Variable_name               | Value |
+-----------------------------+-------+
| Rpl_semi_sync_master_status | ON    |
+-----------------------------+-------+
1 row in set (0.00 sec)
```

从数据库:

```mysql
mysql> show status like 'Rpl_semi_sync_slave_status';
+----------------------------+-------+
| Variable_name              | Value |
+----------------------------+-------+
| Rpl_semi_sync_slave_status | ON    |
+----------------------------+-------+
1 row in set (0.20 sec)
```

#### 5.6 其他变量说明

**环境变量（show variables like '%Rpl%';）**

`rpl_semi_sync_master_wait_for_slave_count` -MySQL 5.7.3 引入的，该变量设置主需要等待多少个 slave 应答，才能返回给客户端，默认为 1。

`rpl_semi_sync_master_wait_no_slave`
ON - 默认值，当状态变量 Rpl_semi_sync_master_clients 中的值小于 rpl_semi_sync_master_wait_for_slave_count
时，Rpl_semi_sync_master_status 依旧显示为 ON。

OFF - 当状态变量 Rpl_semi_sync_master_clients 中的值于 rpl_semi_sync_master_wait_for_slave_count 时，Rpl_semi_sync_master_status
立即显示为 OFF，即异步复制。

**状态变量（show status like '%Rpl_semi%';）**

`Rpl_semi_sync_master_clients` - 当前半同步复制从的个数，如果是一主多从的架构，并不包含异步复制从的个数。

`Rpl_semi_sync_master_no_tx` - The number of commits that were not acknowledged successfully by a slave.

`Rpl_semi_sync_master_yes_tx` - The number of commits that were acknowledged successfully by a slave.

#### 5.7 简单总结

1. 在一主多从的架构中，如果要开启半同步复制，并不要求所有的从都是半同步复制。
2. MySQL 5.7 极大的提升了半同步复制的性能。 5.6 版本的半同步复制，dump thread 承担了两份不同且又十分频繁的任务：传送 binlog 给 slave ，还需要等待 slave
   反馈信息，而且这两个任务是串行的，dump thread 必须等待 slave 返回之后才会传送下一个 events 事务。dump thread 已然成为整个半同步提高性能的瓶颈。在高并发业务场景下，这样的机制会影响数据库整体的
   TPS 。 5.7 版本的半同步复制中，独立出一个 ack collector thread ，专门用于接收 slave 的反馈信息。这样 master 上有两个线程独立工作，可以同时发送 binlog 到 slave ，和接收
   slave 的反馈

#### 5.8 mysql.cnf 其他配置

```shell
主数据库172.16.60.205配置：
[root@mysql-master ~]# cat /usr/local/mysql/my.cnf
.......
server-``id``=1
log-bin=mysql-bin
sync_binlog = 1
binlog_checksum = none
binlog_format = mixed
plugin-load=rpl_semi_sync_master=semisync_master.so
rpl_semi_sync_master_enabled=ON         #或者设置为"1"，即开启半同步复制功能
rpl-semi-sync-master-timeout=1000       #超时时间为1000ms，即1s

```

```shell
从数据库172.16.60.206配置：
[root@mysql-slave ~]# cat /usr/local/mysql/my.cnf
.........
server-``id``=2
log-bin=mysql-bin
slave-skip-errors = all
plugin-load=rpl_semi_sync_slave=semisync_slave.so
rpl_semi_sync_slave_enabled=ON
```

然后从库同步操作，不需要跟 master_log_file 和 master_log_pos=120

如果从库不切换的确可以不设，这个时候默认从上次从库同步的位置开始

{% endraw %}
