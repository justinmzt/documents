# OpenResty_Notes
## 简介
![logo][1]
**OpenResty** （也称为 *ngx_openresty*）是一个**全功能**的 Web 应用服务器。它打包了标准的 **Nginx 核心**，很多的常用的**第三方模块**，以及它们的大多数依赖项。

![enter description here][2]


通过众多进行良好设计的 Nginx 模块，OpenResty 有效地把 Nginx 服务器*转变为*一个**强大的 Web 应用服务器**，基于它开发人员可以使用**Lua 编程语言**对 Nginx 核心以及现有的各种Nginx C 模块进行***脚本编程***，构建出可以处理一万以上并发请求的极端高性能的 Web 应用。

 OpenResty 致力于将你的服务器端应用**完全**运行于 Nginx 服务器中，充分利用 Nginx 的事件模型来进行**非阻塞 I/O 通信**。不仅仅是和 HTTP 客户端间的网络通信是非阻塞的，与MySQL、PostgreSQL、Memcached、以及 Redis 等众多远方后端之间的网络通信也是非阻塞的。

 因为 OpenResty 软件包的维护者也是其中打包的许多 Nginx 模块的作者，所以 OpenResty 可以确保所包含的所有组件可以**可靠地**协同工作。
<div style="
background-color:#eee;
margin:10px;
border-radius:20px
">
<img src="http://c.hiphotos.baidu.com/baike/w%3D268%3Bg%3D0/sign=9d1a2a548f82b9013dadc4354bb6ce4a/e4dde71190ef76c6f509fa589816fdfaae5167ed.jpg" style="width:100px;border-radius:20px">
<p style="margin:10px;">作者简介：</p>
<p style="margin:10px;"><strong>agentzh</strong>，本名章亦春，现任 CloudFare 系统工程师，主要是 Nginx 和 OpenResty 开发，是一名快乐的程序员，现定居美国旧金山。曾经在北京的时候供职于 Yahoo！中国以及淘宝（阿里巴巴）。</p>
<p style="margin:10px;">教程：<a href="http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html">agentzh 的 Nginx 教程</a></p>

</div>

其他内容可以参看：
[OpenResty简介][3]，摘自【[OpenResty 作者章亦春访谈实录][4]】
## 一、OpenResty综述
<style>
.site-name-container{
background-color:#599059;
border-radius:10px;
max-width:500px
}
.site-name {
  /* text-transform: uppercase; */
  margin-top: 20px;
}
.site-name *{
margin-left:10px
}
.site-name a {
  font-size: 34px;
  font-weight: bold;
  color: #f9fcf9;
  text-decoration: none;
}
.site-name a span {
  color: #effc67;
  font-weight: 300;
}
.site-name small {
  display: block;
  font-size: 13px;
  color: #f9fcf9;
}
.site-name img {
  margin-top: -6px;
  margin-left: -20px;
  margin-right: 10px;
}
</style>
<div class="site-name-container">
<p class="site-name left">
                <a href="">OpenResty<span class="trade">®</span></a>
                <small>通过 Lua 扩展 NGINX 实现的可伸缩的 Web 平台</small></p>
</div>				
先扔三个网站：

首先，[OpenResty官网!][5]

还有Github：[OpenResty][6]

为数不多的书籍：[OpenResty最佳实践][7]（Lua语言入门也可以全靠它）

下图给出的是Lua Nginx Module中各指令的执行顺序
![enter description here][8]

- set_by_lua: 流程分支处理判断变量初始化
- rewrite_by_lua: 转发、重定向、缓存等功能(例如特定请求代理到外网)
- access_by_lua: IP 准入、接口权限等情况集中处理(例如配合 iptable 完成简单防火墙)
- content_by_lua: 内容生成
- header_filter_by_lua: 应答 HTTP 过滤处理(例如添加头部信息)
- body_filter_by_lua: 应答 BODY 过滤处理(例如完成应答内容统一成大写)
- log_by_lua: 会话完成后本地异步完成日志记录(日志可以记录在本地，还可以同步到其他机器)

实际上我们只使用其中一个阶段 content_by_lua，也可以完成所有的处理。但这样做，会让我们的代码比较臃肿，越到后期越发难以维护。把我们的逻辑放在不同阶段，分工明确，代码独立，后期发力可以有很多有意思的玩法。

## 二、指令说明：
### *_by_lua < lua-script-str >

无后缀的指令，后加字符串型的lua程序。

如：

``` nginx
 location / {
     default_type text/html;
     content_by_lua '
         ngx.say("<p>hello, world</p>")
     ';
 }
```

### *_by_lua_block {lua_script}

有block后缀，可以直接跟lua程序段。

如：

``` nginx
 location / {
     content_by_lua_block{
         local test = 'Hello,world'
         ngx.say(test)
     }
 }
```


### *_by_lua_file < path-to-lua-script-file >

file后缀跟lua文件路径

如：

``` nginx
location ~ ^/api/([-_a-zA-Z0-9/]+) {
    # 准入阶段完成参数验证
    access_by_lua_file  lua/access_check.lua;

    #内容生成阶段
    content_by_lua_file lua/$1.lua;

}
```

## 二、Gzip
[Module ngx_http_gzip_module][12]

一个例子：
``` nginx
gzip on; #开启gzip压缩输出数据
gzip_min_length 1000; #压缩临界长度
gzip_proxied expired no-cache no-store private auth; #设置代理
gzip_types text/plain application/xml; #设置要压缩的格式
gzip_http_version 1.0; #设置最低HTTP版本
```

当nginx要对后台数据进行过滤时：

``` elixir
后台发送-->nginx解压缩模块-->过滤数据-->gzip压缩-->发送到客户端
```

关键步骤是解压缩。

网上有相关资料，基本都是使用`lua_zlib`模块对数据进行解压。我下载安装了`lua_zlib`模块，但根据网上的示例程序没能实现。

现在贴一些相关资料：

 [OpenResty 之 nginx lua gzip数据][13]，[在OpenResty中使用lua-zlib的方法][14]

我通过之前的方法导出了res.body，但经过zlib.inflate()函数解压后得到空值，之后就卡在这里了。

### 解决方案

原流程：

``` elixir
后台发送-->nginx解压缩模块-->过滤数据-->gzip压缩-->发送到客户端
```

简化流程：

``` elixir
后台发送未压缩数据-->nginx接收并过滤数据-->gzip压缩-->发送到客户端
```

上述步骤简化了nginx解压缩模块。

上代码：
   

``` nginx
#配合nginx的gzip模块，接收未压缩数据，过滤后在用gzip发送
location ~ ^/api/([-_a-zA-Z0-9/]+) {
    proxy_pass http://192.168.79.128:1018;
    proxy_set_header    Host $host;
    proxy_set_header    X-Real-IP   $remote_addr;
    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
    body_filter_by_lua_block {
        ngx.arg[1] = string.gsub(ngx.arg[1], "", "")
    }
}
```

其中`string.gsub(s, p, r [, n])`

将目标字符串 s 中所有的子串 p 替换成字符串 r。可选参数 n，表示限制替换次数。返回值有两个，第一个是被替换后的字符串，第二个是替换了多少次。

此函数不能为 LuaJIT 所 JIT 编译，而只能被解释执行。官方推荐使用 ngx_lua 模块提供的 ngx.re.gsub 函数。

## 三、登陆验证

在`access_by_lua*`中集中进行一些权限认证，防止恶意ip、非法行为进入到服务器中。

``` nginx
    location / {
	    access_by_lua_block{
            ngx.exit(ngx.HTTP_FORBIDDEN)
        }
	    content_by_lua_block{
            --内容生产阶段
        }
	}
```
### IP防火墙

OpenResty最佳实践提到：[禁止某些终端访问][15]

上面文档下方推荐第三方包：Github，[Lua-resty-iputils][16]


### 操作头信息检测

通过`access_by_lua*`可以对请求进行过滤，比如可以在这里检查有没有带`authorization`头部信息，若没有带可以用`ngx.exit()`直接退出。

### 登陆缓存

Github: [lua-resty-jwt模块进行jwt校验][17]

结合`lua-resty-jwt`和`lua-resty-redis`甚至可以将服务器登陆缓存**移植**到nginx上。

### 过滤参数

还未尝试。

[防止 SQL 注入][18]

## 四、输出过滤

通常是`header_filter_by_lua*`与`body_filter_by_lua*`配合实现。

【文档来源官方】：

注意下列API函数现在还不能在`set_by_lua*`、`header_filter_by_lua*`、`body_filter_by_lua*`、`log_by_lua*`四个环境中使用：

- 输出API 函数 (e.g., ngx.say and ngx.send_headers)
- 控制API 函数 (e.g., ngx.redirect and ngx.exec)
- 子请求API 函数 (e.g., ngx.location.capture and ngx.location.capture_multi)
- Cosocket API 函数 (e.g., ngx.socket.tcp and ngx.req.socket).

### header_filter_by_lua*

使用Lua代码在lua块中定义输出头信息。

example1：

``` nginx
 location / {
     proxy_pass http://mybackend;
     header_filter_by_lua 'ngx.header.Foo = "blah"';
 }
```
example2：

``` nginx
 header_filter_by_lua_block {
     ngx.header["content-length"] = nil
 }
```


### body_filter_by_lua*

参考文献：

[谈谈 OpenResty 中的 body_filter_by_lua*][19]


输入数据块`chunks`要经过ngx.arg[1] (Lua 字符串)的过滤，"eof"标志指示响应体数据流的末端通过ngx.arg[2] (Lua bollean值)显示。


这个情景下，"eof"标志就是主请求的`last_buf`或子请求的 `last_in_chain`。"eof" == "true" 表示此次nginx请求的响应结束了。

由于响应体可能会分多块发送，`body_filter_by_lua*`可能会被多次调用。详细机理参考上面文献有提到。

``` nginx
 location /t {
     echo hello world;
     echo hiya globe;

     body_filter_by_lua '
	     ......
     ';
 }
```

比如上面，`body_filter_by_lua*`首次调用时ngx.arg[1] 的值只是`hello world`，不包括下面的`hiya globe`。

我验证了一下：

``` nginx
location = /test_bf {
    echo hello world;
    echo this;
    echo is;
    echo world;
    header_filter_by_lua_block {
        ngx.header.content_length = nil
    }
    body_filter_by_lua_block {
        if string.match(ngx.arg[1], "this") then
            ngx.arg[2] = true
            ngx.arg[1] = "end"
            return
        end
    }
}
```

上述location发送四块数据，匹配到this便设置"eof"不再发送下去了。


``` elixir
$ curl
```
结果：

``` elixir
$ hello world
$ end
```
要点：

① `ngx.arg[2] = true`设置新的"eof"标志使截断响应，设置"eof"依旧是有效的响应。

② `ngx.arg[1] = "end"`修改`ngx.arg[1]` 的值，即为修改输出响应。而当需要替换一些信息，仅需一行代码即可实现。

``` lua
ngx.arg[1] = ngx.re.gsub(ngx.arg[1], "o", "*")
```

上文也提到。上述即为将所有 ' o ' 字符替换为 ' * ' 字符

若该`location`作为其他`location`的子请求，而作为子请求**不想**被过滤数据。需要通过`ngx.is_subrequest`判断。

``` nginx
body_filter_by_lua_block {
    if ngx.arg[1] and not ngx.is_subrequest then
        ......
    end
}
```

还有一点，如果在`body_filter_by_lua*`中的Lua代码会修改响应体的长度，这就需要去把`Content-Length`的响应头去除掉。我之前测试与前端联调时，就是发现body过滤必须要字符数相同才会显示，很头疼不知道什么原因。其实只要去掉这个头就行了。

``` nginx
location = / {
    proxy_pass ......
    header_filter_by_lua_block {
        ngx.header.content_length = nil
    }
    body_filter_by_lua_block {
        ngx.arg[1] = ngx.re.gsub(ngx.arg[1], "o", "**")
	}
}
```

## 五、Redis

官方github：[lua-resty-redis][20]

OpenResty最佳实践：[访问有授权验证的 Redis][21]


  [1]: https://openresty.org/images/logo.png
  [2]: http://nginx.org/nginx.png
  [3]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/base/intro.html
  [4]: http://www.oschina.net/question/28_60461?sort=default&p=3
  [5]: http://openresty.org/en/
  [6]: https://github.com/openresty
  [7]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/
  [8]: https://cloud.githubusercontent.com/assets/2137369/15272097/77d1c09e-1a37-11e6-97ef-d9767035fc3e.png
  [9]: https://github.com/openresty/lua-nginx-module#http-method-constants
  [10]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/openresty/how_request_http.html
  [11]: https://github.com/pintsized/lua-resty-http
  [12]: http://nginx.org/en/docs/http/ngx_http_gzip_module.html
  [13]: http://blog.csdn.net/mrsunnycream/article/details/44999217
  [14]: http://www.cnblogs.com/littlehb/p/4194943.html
  [15]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/ngx_lua/allow_deny.html
  [16]: https://github.com/hamishforbes/lua-resty-iputils
  [17]: https://github.com/SkyLothar/lua-resty-jwt
  [18]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/openresty/safe_sql.html
  [19]: https://segmentfault.com/a/1190000007483746
  [20]: https://github.com/openresty/lua-resty-redis
  [21]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/redis/auth_connect.html