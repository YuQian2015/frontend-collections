## Nginx

轻量级、效率高，高性能的 HTTP 和反向代理 web 服务器

## 配置文件介绍

### 文件结构

- 全局块：配置影响nginx全局的指令。一般有运行nginx的用户组，nginx进程pid存放路径、日志存放路径、配置文件引入，允许生成worker process数等。
- events块：配置影响nginx服务器或与用户的网络连接。有每个进程的最大连接数，选取哪种事件驱动模型处理连接请求，是否允许同时接受多个网络连接，开启多个网络连接序列化等。
- http块：可以嵌套多个server，配置代理，缓存，日志定义等绝大多数功能和第三方模块的配置。如文件引入，mine-type定义，日志自定义，是否使用sendfile传输文件，连接超时时间，单连接请求数等。
- server块：配置虚拟主机的相关参数，一个http可以有多个server。
- location块：配置请求的路由，以及各种页面的处理情况。

```
...                           #全局块
events {                      #events块
     ...
}
http                          #http块
{
    ...                       #http全局块
    server                    #server块
    {
        ...                   #server全局块
        location [PATTERN]    #location块
        {
            ...
        }
        location [PATTERN]
        {
            ...
        }
    }
    server
    {
        ...
    }
    ...                       #http全局块
}
```

### 部分配置

```
# user administrator administrators;  #配置用户或组，默认为nobody nobody
# worker_process 2;  #允许生成的进程数，默认为1
# pid /nginx/pid/nginx.pid #指定nginx进程运行文件的存放位置
error_log log/error.log debug; #指定日志路径，级别。这个配置可以放入区全局块，http块server块。级别依次为：debug|info|notice|warn|error|crit|alert|emerg

events {
  accept_mutex on; 设置网络连接序列化，防止惊群现象发生，默认为on
  multi_accept on; #配置一个进程是否同时接受多个网络连接，默认为off
  #use epoll; #事件驱动模型，select|poll|kqueue|epoll|resig|/dev/poll|eventport
  worker_connections 1024; #最大连接数，默认为512
}

http {
  include       mime.types;  #文件扩展名与文件类型映射表
  default_type  application/octet-stream;  #默认文件类型，默认为text/plain

  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"'; # 自定义格式

  #access_log off; #取消日志服务
  access_log log/access.log myFormat; #combined为日志格式的默认值
  sendfile on; #允许sendfile方式传输文件，默认为off，可以在http块，server块，location块
  sendfile_max_chunk 100k; #每个进程每次调用传输数量不能大于设置的值，默认为0，即不设上限
  keepalive_timeout 65; #连接超时时间，默认为75s，可以在http块server块location块
  error_page 404 https://www.baidu.com; #错误页
  #gzip on;
  server_tokens off; # 不显示nginx版本

  upstream mysvr {
    server 127.0.0.1:7878;
    server 192.168.10.121:3333 backup; #热备
  }

  server {
    keepalive_requests 120; #单连接请求次数上限
    listen 4545; #监听端口
    server_name 127.0.0.1; #监听地址
    location ~*^.+$ { #请求的URL过滤，正则匹配，~为区分大小写，~*为不区分大小写
    #root path; #根路径
    #index vv.txt; #设置默认页
    proxy_pass http://mysvr; #请求转向 mysvr 定义的服务器列表
    deny 127.0.0.1; #拒绝的ip
    allow 172.18.5.54; #允许的ip
  }
}
```

- 配置项
  - $remote_addr与$http_x_forwarded_for：记录客户端的IP
  - $remote_user：记录客户端用户名称
  - $time_local：记录访问时间和时区
  - $request：记录请求的URL和HTTP协议
  - $status：记录请求状态，成功是200
  - $body_bytes_s ent：记录发送给客户端文件主体内容大小
  - $http_referer：记录从哪个页面链接访问过来的
  - $http_user_agent：记录客户端浏览器的信息
- 惊群现象：一个网络连接过来，多个睡眠的进程同时被唤醒，但只有一个进程获得该连接，这样会影响系统性能
- 每个指令必须由分号结尾

原文地址：https://www.kancloud.cn/surahe/front-end-notebook/1105495

### 虚拟主机配置方式

通过 server 块里面的 listen 配置监听端口，server_name 配置监听地址，地址可以是域名或IP，本机IP访问时，如果没有配置对应IP则会查找已经配好的一个虚拟主机。

通过 root 可以指定虚拟主机的根路径，根路径可以是绝对路径，也可以是相对路径，相对路径是以nginx根目录为参考的，比如root html;表示nginx目录下的html目录。index 可以用来配置默认查找的页面，比如index index.html; 表示查找根路径下的index.html页面。



## 负载均衡

### upstream

这个配置是写一组 被代理的服务器地址，然后配置负载均衡的算法。
写法1：

```
upstream mysvr {
    server 192.168.10.121:3333;
    server 192.168.10.122:3333;
}
server {
    ....
    location ~*^.+$ {
        proxy_pass http://mysvr; #请求转向mysvr定义的服务器列表
    }
}
```

写法2：

```
upstream mysvr {
    server http://192.168.10.121:3333;
    server http://192.168.10.122:3333;
}
server {
    ....
    location ~*^.+$ {
        proxy_pass mysvr; #请求转向mysvr定义的服务器列表
    }
}
```

### 热备

加入你有2台服务器，当一台服务器发生事故时，才启用第二台服务器提供服务。服务器处理请求的顺序：AAAA故障，BBBBB...

```
upstream mysvr {
    server 127.0.0.1:7878;
    server 192.168.10.121:3333 backup; #热备
}
```

### 轮询

nginx默认就是轮询，权重默认是1，服务器处理请求的顺序为：ABABAB...

```
upstream mysvr {
    server 127.0.0.1:7878;
    server 192.168.10.121:3333;
}
```

### 加权轮询

根据配置权重的大小而分发给不同服务器不同的请求。如果不设置，默认为1。服务器处理请求的顺序为：ABBABBABB...

```
upstream mysvr {
    server 127.0.0.1:7878 weight=1;
    server 192.168.10.121:3333 weight=2;
}
```

### ip_hash

nginx会让相同的客户端ip请求相同的服务器

```
upstream mysvr {
    server 127.0.0.1:7878;
    server 192.168.10.121:3333;
    ip_hash;
}
```

### 状态参数

- down，表示当前server暂时不参与负载均衡
- backup，预留的备份机器，当其他所有的非backup机器故障或者忙时，才会请求backup机器，因此这台机器压力最轻
- max_fails，允许请求失败的次数，默认为1。当超过最大次数时，返回proxy_next_upstream 模块定义的错误
- fail_timeout，经历max_fail此失败后，暂停服务的时间，max_fails可以与fail_timeout一起使用

```
upstream mysvr {
    server 127.0.0.1:7878 weight=2 max_fails=2 fail_timeout=2;
    server 192.168.10.121:3333 weight=1 max_fails=2 fail_timeout=1;
}
```

原文地址：https://www.kancloud.cn/surahe/front-end-notebook/1106201

## 获取用户IP

### 配置

```
proxy_set_header Host $host; #只要用户在浏览器访问的域名绑定了VIP，VIP下面有RS；则使用$host；host是访问URL中的域名和端口 www.taobao.com:80
proxy_set_header X-Real-IP $remote_addr; #把源IP【$remote_adr，建立HTTP连接header里的信息】赋值给X-Real-IP，这样在代码 $X-Real-IP获取源IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;#在 nginx 作为代理服务器，设置的IP列表，会把经过的机器IP，代理机器IP都记录下来，用【,】隔开，代码中用 echo $x-forwarded-for | awl -F, '{print $1}' 作为源IP
```

### 背景

通过名字就知道，X-Forwarded-For 是一个 HTTP 扩展头部。HTTP/1.1（RFC 2616）协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件引入，用来表示 HTTP 请求端真实 IP。如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入[RFC 7239](http://tools.ietf.org/html/rfc7239)（Forwarded HTTP Extension）标准之中。

X-Forwarded-For 请求头格式非常简单，就这样：

```
X-Forwarded-For: client, proxy1, proxy2
```

可以看到，XFF 的内容由「英文逗号 + 空格」隔开的多个部分组成，最开始的是离服务端最远的设备 IP，然后是每一级代理设备的 IP。

如果一个 HTTP 请求到达服务器之前，经过了三个代理 Proxy1、Proxy2、Proxy3，IP 分别为 IP1、IP2、IP3，用户真实 IP 为 IP0，那么按照 XFF 标准，服务端最终会收到以下信息：

```
X-Forwarded-For: IP0, IP1, IP2
```

Proxy3 直连服务器，它会给 XFF 追加 IP2，表示它是在帮 Proxy2 转发请求。列表中并没有 IP3，IP3 可以在服务端通过 Remote Address 字段获得。我们知道 HTTP 连接基于 TCP 连接，HTTP 协议中没有 IP 的概念，Remote Address 来自 TCP 连接，表示与服务端建立 TCP 连接的设备 IP，在这个例子里就是 IP3。



Remote Address 无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求。不同语言获取 Remote Address 的方式不一样，例如 php 是`$_SERVER["REMOTE_ADDR"]`，Node.js 是`req.connection.remoteAddress`，但原理都一样。

### 问题

这段代码会监听`9009`端口，并在收到 HTTP 请求后，输出一些信息：

```
var http = require('http');

http.createServer(function (req, res) {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.write('remoteAddress: ' + req.connection.remoteAddress + '\n');
    res.write('x-forwarded-for: ' + req.headers['x-forwarded-for'] + '\n');
    res.write('x-real-ip: ' + req.headers['x-real-ip'] + '\n');
    res.end();
}).listen(9009, '0.0.0.0');
```

这段代码除了前面介绍过的 Remote Address 和`X-Forwarded-For`，还有一个`X-Real-IP`，这又是一个自定义头部字段。`X-Real-IP`通常被 HTTP 代理用来表示与它产生 TCP 连接的设备 IP，这个设备可能是其他代理，也可能是真正的请求端。需要注意的是，`X-Real-IP`目前并不属于任何标准，代理和 Web 应用之间可以约定用任何自定义头来传递这个信息。



现在可以用域名 + 端口号直接访问这个 Node.js 服务，再配一个 Nginx 反向代理：

```
server{
  listen 80;
  server_name 127.0.0.1;
  root /srv/ip-test;
  index index.html;
  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-NginX-Proxy true;

    proxy_pass http://127.0.0.1:9009/;
    proxy_redirect off;
  }
}
```

Nginx 监听`80`端口，所以不带端口就可以访问 Nginx 转发过的服务。



测试直接访问 Node 服务：

```
curl http://192.168.199.217:9009/

remoteAddress: 192.168.199.180
x-forwarded-for: undefined
x-real-ip: undefined
```

由于我的电脑直接连接了 Node.js 服务，Remote Address 就是我的 IP。同时我并未指定额外的自定义头，所以后两个字段都是 undefined。



再来访问 Nginx 转发过的服务：

```
remoteAddress: 127.0.0.1
x-forwarded-for: 192.168.199.180
x-real-ip: 192.168.199.180
```

通过 Nginx 访问 Node.js 服务，得到的 Remote Address 实际上是 Nginx 的本地 IP。而前面 Nginx 配置中的这两行起作用了，为请求额外增加了两个自定义头：

```
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```



HTTP 请求头可以随意构造，我们通过 curl 的`-H`参数构造`X-Forwarded-For`和`X-Real-IP`，再来测试一把。
直接访问 Node.js 服务：

```
curl http://192.168.199.217:9009/  -H 'X-Forwarded-For: 1.1.1.1'  -H 'X-Real-IP: 2.2.2.2'

remoteAddress: 192.168.199.217
x-forwarded-for: 1.1.1.1
x-real-ip: 2.2.2.2
```

对于 Web 应用来说，`X-Forwarded-For`和`X-Real-IP`就是两个普通的请求头，自然就不做任何处理原样输出了。这说明，对于直连部署方式，除了从 TCP 连接中得到的 Remote Address 之外，请求头中携带的 IP 信息都不能信。



访问 Nginx 转发过的服务：

```
curl http://192.168.199.217/  -H 'X-Forwarded-For: 1.1.1.1'  -H 'X-Real-IP: 2.2.2.2'

remoteAddress: 127.0.0.1
x-forwarded-for: 1.1.1.1, 192.168.199.217
x-real-ip: 192.168.199.217
```

这一次，Nginx 会在`X-Forwarded-For`后追加我的 IP；并用我的 IP 覆盖`X-Real-IP`请求头。这说明，有了 Nginx 的加工，`X-Forwarded-For`最后一节以及`X-Real-IP`整个内容无法构造，可以用于获取用户 IP。



用户 IP 往往会被使用与跟 Web 安全有关的场景上，例如检查用户登录地区，基于 IP 做访问频率控制等等。这种场景下，确保 IP 无法构造更重要。经过前面的测试和分析，对于直接面向用户部署的 Web 应用，必须使用从 TCP 连接中得到的 Remote Address；对于部署了 Nginx 这样反向代理的 Web 应用，在正确配置了 Set Header 行为后，可以使用 Nginx 传过来的`X-Real-IP`或`X-Forwarded-For`最后一节（实际上它们一定等价）。



那么，Web 应用自身如何判断请求是直接过来，还是由可控的代理转发来的呢？在代理转发时增加额外的请求头是一个办法，但是不怎么保险，因为请求头太容易构造了。如果一定要这么用，这个自定义头要够长够罕见，还要保管好不能泄露出去。



判断 Remote Address 是不是本地 IP 也是一种办法，不过也不完善，因为在 Nginx 所处服务器上访问，无论直连还是走 Nginx 代理，Remote Address 都是 `127.0.0.1`。这个问题还好通常可以忽略，更麻烦的是，反向代理服务器和实际的 Web 应用不一定部署在同一台服务器上。所以更合理的做法是收集所有代理服务器 IP 列表，Web 应用拿到 Remote Address 后逐一比对来判断是以何种方式访问。



首先，如果用户真的是通过代理访问 Nginx，`X-Forwarded-For`最后一节以及`X-Real-IP`得到的是代理的 IP，安全相关的场景只能用这个，但有些场景如根据 IP 显示所在地天气，就需要尽可能获得用户真实 IP，这时候`X-Forwarded-For`中第一个 IP 就可以排上用场了。这时候需要注意一个问题，还是拿之前的例子做测试：

```
curl http://192.168.199.217/ -H 'X-Forwarded-For: unknown, <>"1.1.1.1'

remoteAddress: 127.0.0.1
x-forwarded-for: unknown, <>"1.1.1.1, 192.168.199.217
x-real-ip: 192.168.199.217
```



`X-Forwarded-For`最后一节是 Nginx 追加上去的，但之前部分都来自于 Nginx 收到的请求头，这部分用户输入内容完全不可信。使用时需要格外小心，符合 IP 格式才能使用，不然容易引发 SQL 注入或 XSS 等安全漏洞。

### 结论

1. 直接对外提供服务的 Web 应用，在进行与安全有关的操作时，只能通过 Remote Address 获取 IP，不能相信任何请求头；
2. 使用 Nginx 等 Web Server 进行反向代理的 Web 应用，在配置正确的前提下，要用`X-Forwarded-For`最后一节 或`X-Real-IP`来获取 IP（因为 Remote Address 得到的是 Nginx 所在服务器的内网 IP）；同时还应该禁止 Web 应用直接对外提供服务；
3. 在与安全无关的场景，例如通过 IP 显示所在地天气，可以从`X-Forwarded-For`靠前的位置获取 IP，但是需要校验 IP 格式合法性；

PS：网上有些文章建议这样配置 Nginx，其实并不合理：

```
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $remote_addr;
```

这样配置之后，安全性确实提高了，但是也导致请求到达 Nginx 之前的所有代理信息都被抹掉，无法为真正使用代理的用户提供更好的服务。

### 参考资料

[HTTP 请求头中的 X-Forwarded-For](https://imququ.com/post/x-forwarded-for-header-in-http.html)

原文地址：https://www.kancloud.cn/surahe/front-end-notebook/1106131

原文地址：[Jerry Qu](https://imququ.com/) - [HTTP 请求头中的 X-Forwarded-For]( https://imququ.com/post/x-forwarded-for-header-in-http.html)