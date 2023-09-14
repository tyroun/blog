{% raw %}

# Nginx Lua开发实战

## 第1章 Nginx高效服务器

### 1.1 Nginx的特点

1. 速度快
2. 扩展性好         -      核心+模块，  Epoll/Kqueue做分发， ngx_lua开发，lua的协程性能好
3. 高可靠性         -      管理进程master   +  若干个工作进程 worker
4. 低内存占用      -      10000个非活跃的HTTP保活连接仅占用2.5MB内存
5. 高并发
6. 热部署
7. 开源

### 1.2 Nginx的安装

#### 1 依赖库

1. GCC
2. PCRE       - Perl Compatible Regular Expressions, Perl兼容正则表达式
3. Zlib         - 用于对HTTP报文内容做gzip格式压缩，如果在nginx.conf中配置了gzip on，并指定某些Content-Type的HTTP应答包体使用gzip进行压缩，以减小网络传输量，那么就需要在编译Nginx时指定zlib，将其编译进Nginx
4. OpenSSL - 使用HTTPS在SSL上传输HTTP，就需要安装OpenSSL。另外，在ngx_lua中使用MD5、SHA1等散列函数时，也需要安装OpenSSL

#### 2 Linux内核参数优化

修改/etc/sysctl.conf，常用配置如下：[插图]执行sysctl -p命令，使修改生效。

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

执行sysctl -p命令，使修改生效。

各内核参数的作用如下。

1. fs.file-max：表示进程（在Nginx里指一个工作进程）可以同时打开的最大句柄数。本参数影响最大并发连接数。
2. net.ipv4.tcp_syncookies：解决TCP的SYN攻击。
3. net.ipv4.tcp_tw_reuse：参数为1时表示允许将TIME_WAIT状态的套接字重新用于新的TCP连接。服务器上的TCP协议栈在工作时会有大量的TIME_WAIT状态连接，重新使用这些连接对于服务器处理大并发连接非常有用。
4. net.ipv4.tcp_keepalive_time：表示当keepalive启用时，TCP发送keepalive消息的频率。默认为2小时，如果本值变小，可以更快地清理无效的连接。
5. net.ipv4.tcp_fine_timeout：表示服务器主动关闭连接时，套接字的FIN_WAIT-2状态最大时间
6. net.ipv4.tcp_max_tw_buckets：表示操作系统允许的TIME_WAIT套接字数量的最大值。当超过这个值，TIME_WAIT状态的套接字被立即清除并输出警告消息，默认值为180000，过多的TIME_WAIT套接字会使服务器速度变慢。
7. net.ipv4.tcp_max_syn_backlog：表示TCP三次握手阶段SYN请求队列最大值，默认为1024。调置为更大的值可以使Nginx在非常繁忙的情况下，若来不及接收新的连接时，Linux不至于丢失客户端新创建的连接请求。
8. net.ipv4.ip_local_port_range：定义UDP和TCP连接中本地端口范围（不包括连接到远端的端口）。
9. net.ipv4.tcp_rmen：定义TCP接收缓存（TCP接收窗口）的最小值、默认值、最大值。
10. net.ipv4.tcp_wmen：定义TCP发送缓存（TCP发送窗口）的最小值、默认值、最大值。
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

## 第3章 OpenResty

### 3.2 OpenResty的组成

1. 标准Lua 5.1解释器；
2. Drizzle Nginx模块；
3. Postgres Nginx模块；
4. Iconv Nginx模块

上面4个模块默认并未启用，需要分别加入--with-lua51、--with-http_drizzle_module、--with-http_postgres_module和--with-http_iconv_module编译选项开启它们

运行下面命令启动Nginx：

```shell
/usr/local/openresty/nginx/sbin/nginx -p /usr/local/openresty/nginx/
```

在浏览器里输入http://127.0.0.1（或主机IP），看到“Welcome to OpenResty！ ”表示已经启动成功。

可以进一步修改/usr/local/openresty/nginx/conf/nginx.conf，测试Lua是否正常工作，nginx.conf内容如下

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

### 3.4 Nginx多实例

只需要把OpenRestry中的Nginx目录复制一份就可以启动不同的实例：

```shell
cp -r /usr/local/openrestry/nginx  /usr/local/openrestry/nginx_9090
```

然后修改nginx_9090/conf/nginx.conf，把端口从8080修改为9090，把“hello world”修改为“hello world2”，修改完成后启动实例。

```shell
/usr/local/openrestry/nginx_9090/sbin/nginx -p /usr/local/openrestry/nginx_9090/
```

## 第4章 Nginx核心技术

### 4.2 Nginx架构

Nginx使用事件驱动的服务模型。把一个处理过得的过程（如HTTP请求）划分为7个、9个或11个阶段，每一个阶段都异步处理，将请求和处理结果异步化处理。

管理进程和工作进程的机制使Nginx可以充分利用多处理器机制，充分利用了SMP机制的硬件资源

对于高并发下的多工程进程容易引起的“惊群”以及负载均衡问题，Nginx都做了比较好的处理

#### 4.2.1 事件驱动

描述Nginx事件处理的模型，即事件、事件管理器和事件消费者之间的一个概貌

![image-20230905114040251](../image/Nginx OpenResty Keepalived/image-20230905114040251.png)

#### 4.2.2 异步多阶段处理

获取一个静态文件的HTTP请求可以划分为下面的几个阶段

1. 建立TCP连接阶段：收到TCP的SYN包。
2. 开始接收请求：接收到TCP中的ACK包表示连接建立成功。
3. 接收到用户请求并分析请求是否完整：接收到用户的数据包。
4. 接收到完整用户请求后开始处理：接收到用户的数据包。
5. 由静态文件读取部分内容：接收到用户数据包或接收到TCP中的ACK包，TCP窗口向前划动。这个阶段可多次触发，直到把文件完全读完。不一次性读完是为了避免长时间阻塞事件分发器。
6. 发送完成后：收到最后一个包的ACK。对于非keepalive请求，发送完成后主动关闭连接。
7. 用户主动关闭连接：收到TCP中的FIN报文

#### 4.2.3 模块化设计

![image-20230905114437079](../image/Nginx OpenResty Keepalived/image-20230905114437079.png)

#### 4.2.4 管理进程、工作进程设计

![image-20230905114515568](../image/Nginx OpenResty Keepalived/image-20230905114515568.png)

##### 1“惊群”问题

多个进程监听同一个端口，连接请求来的时候，开始抢夺事件   -     惊群

在epoll模式下，内核在收到了TCP的SYN包时，会激活所有休眠的工作进程，最先接收连接请求的工作进程可以成功建立新连接，其他工作进程的接收会失败。这些失败的唤醒是不必要的，引发了不必要的进程上下文切换，增加了系统开销，这就是“惊群”问题

Nginx应用层制定了一个机制解决这个问题：规定同一时刻只能有唯一一个工作进程监听Web端口，这样，新的连接事件只能唤醒唯一一个工作进程。内部的实现实际上是使用了一个进程间的同步锁，工作进程每次唤醒都先尝试这把锁，保证同一时间只有一个工作进程可以进入锁，获得锁的进程设置监听连接的读事件，以处理未来的新连接请求，并处理已连接上的事件；未能进入监听锁的工作进程则不监听新连接事件，只处理已连接上的事件，将唤醒的工作进程分为了两类，一类（只有1个）是可以监听新连接的，另一类是正常处理已有连接请求的。

就是有个队列，只有一个进程监听端口，被换醒后马上释放锁，让另一个进程监听

##### 2 负载均衡

**系统级的负载均衡**

1. 系统级的负载均衡，实现方法是使用一个Nginx服务通过upstream机制将请求分配到上游后端服务器，而这里可以使用模块内置的一些负载均衡机制将请求均衡地分配到服务器组中。
2. 使用一个单独的Nginx服务以自定义负载均衡算法实现代理模式，实现负载均衡集群。

**单Nginx服务内部工作进程间的负载均衡**

不让某个进程“累死”，其他的进程“闲死”，才能发挥系统的最佳性能

#### 4.2.11 keepalive

keepalive是HTTP长连接

请求 request：

POST的Content-Length表示body大小

应答 response:

HTTP 1.0 -  Content-Length 表示body长，如果响应头中没有Content-Length头，则客户端会一直接收数据，直到服务端主动断开连接，才表示body接收完

HTTP 1.1 -  

​		Transfer-encoding: chunked 表示是流式传输，body会被分成多个块，每块的开始会标识出当前块长度

​        Transfer-encoding != chunked，则按照Content-Length接收数据。如果Content-Length也没有，则是长连接

如果客户端的request header 中的connection: close，则表示客户端需要关掉长连接。如果客户端的请求头中的connection: keepalive，则客户端需要打开长连接

服务器response的header里Keep-Alive: timeout=60 表示超时时间, Connection: keep-alive 表示打开长连接

## 第9章 nginx.conf文件配置

### 9.1 默认nginx.conf文件

Nginx提供了一个默认的nginx.conf模板，里面包含了一个HTTP服务配置块、一个HTTPS配置块和主要的全局配置项

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

### 9.2 nginx.conf示例

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

#### 9.3.1 main全局配置

##### 1．worker_processes 

工作进程数

```nginx
    worker_processes number | auto;   #语法
    worker_process 1;				 #示例
```

在配置文件的全局main部分，管理进程接收任务并将请求分配给工作进程处理，工作进程是实际的处理进程。工作进程的个数可以设置为CPU的核数

##### 2．worker_cpu_affinity 

绑定工作进程到指定的CPU内核

```nginx
    worker_cpu_affinity cpumask[cpumask…] #语法

    worker_processes 4;
    worker_cpu_affinity 1000010000100001; #示例
```

如果CPU非常繁忙，不一定会把每一个工作进程分配到一个核心上，通过本指令手工指定会得到真正的并发（仅对Linux有效）

##### 3．worker_rlimit_nofile

工作进程最大打开文件数

```nginx
    worker_rlimit_nofile number;
```

##### 4．worker_directory

工作进程的当前工作路径，主要用于生成coredump文件，工作进程所在用户和组需要对工作目录有写权限

```nginx
    worker_directory directory;
```

##### 5．worker_priority

工作进程优先级，取值范围为-20～+20

```nginx
    worker_priority number;
```

##### 6. worker_rlimit_core

coredump文件最大尺寸

```nginx
    worker_rlimit_core size;
```

##### 7．daemon

是否以守护进程方式运行Nginx

```nginx
    daemon on|off;
```

##### 8．master_process

是否以master_process方式工作, 指明工作进程是否马上启动，主要用于Nginx深度开发使用，不是常规配置功能项

```nginx
    master_process on|off;
```

##### 9. error_log

error日志设置, 设置日志文件路径名，还可以设置要写入的错误级别

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

引用其他配置文件，文件名可以是绝对路径，也可以是相对路径，如果是相对路径，就是nginx.conf所在的路径

```nginx
    include /path/file;
```

##### 12．lock_file

锁定文件，Nginx使用锁定文件实现accept_mutex。在多数系统上，这个锁用原子操作实现，则这个值就被忽略掉了。这个配置在使用lock file机制的系统上使用

```nginx
    lock_file file;
```

##### 13．pid

设置pid文件路径，设置保存管理进程ID的文件，用于使用进程ID操作Nginx的环境

```nginx
    pid path/file;
```

##### 14．user

设置工作进程运行时的用户及用户组

```nginx
    user username[groupname];
```

##### 16．thread_pool

工作进程中多线程读和写的线程池，定义工作进程中对文件读和写时用到的线程池。threads参数定义线程池中线程的数量。max_queue限制允许在队列中等待的任务，默认是65536个任务。队列超出时，任务将返回一个错误信息

```nginx
    thread_pool name threads=number [max_queue=number];

    thread_pool default threads=32 max_queue=65535;
```

#### 9.3.2 events配置块

events模块中包含Nginx中所有处理连接的设置。events是Nginx使用到的I/O事件模型，是最主要的进出交互部分。Nginx强大的部分就是在Linux上完美实现了epoll模型。

常用配置项如下：

```nginx
    events{
        use epoll;
        worker_connections 20000;
    }
```

#### 9.3.3 http服务器配置块

HTTP模块中一般使用HTTP全局配置参数控制整体行为，使用server配置虚拟主机，包含监听地址、文档路径和各种location

反向代理、负载均衡等都是在内部的server等模块实现的。同时在各个子配置块或location等内部划分了许多阶段（phase），这些阶段可以注册Lua代码或Lua文件，干预处理的过程

一个典型的Web服务器会包含全局配置、多个server块和多个location块

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

1. default：将所在的server块作为整个Web服务的默认server块。如果没有设置这个参数，将以找到的第一个server块为默认server块。
2. default_server：同default。
3. backlog=num：表示TCP中backlog队列大小，默认为-1，表示不设置。在TCP三次握手过程中，进程还没有开始处理监听句柄，backlog队列就会放置这些新连接。如果队列已满，新的客户尝试连接，则会失败。
4. rcvbuf=size：设置so_rcvbuf接收缓冲区大小。
5. sndbuf=size：设置so_sndbuf发送缓冲区大小。
6. accept_filter：设置accept过滤器，只对FreeBSD操作系统有用。
7. deferred：设置本参数后，客户端建立连接，并且完成了三次握手，也不会调度工作进程来处理，直到客户端实际请求数据到来才分配工作进程处理，适用于大并发情况下减轻工作进程负担。
8. bind：绑定当前ip/port对，只有在一个端口监听多个地址时才会生效。
9. ssl：在当前监听的端口上建立的连接必须使用SSL协议。

##### 2．server_name

主机名称，server_name后可以跟多个主机名称

```nginx
    server_name name[...];

    server_name www.google.com mail.google.com;
```





{% endraw %}



