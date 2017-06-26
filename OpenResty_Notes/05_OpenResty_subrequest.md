
## OpenResty_subrequest
参考于《OpenResty最佳实践》一书
### 使用场景
我们的使用场景主要就是用nginx发送代理请求。

nginx 世界的 location 是异常强大的，毕竟 nginx 的主要应用场景是在负载均衡、API server，在不同 server、location 之间跳转更是家常便饭。

### 内部调用
例如对数据库、内部公共函数的统一接口，可以把它们放到统一的 location 中。通常情况下，为了保护这些内部接口，都会把这些接口设置为 internal 。这么做的最主要好处就是可以让这个内部接口相对独立，不受外界干扰。

``` nginx
location = /sum {
    # 只允许内部调用
    internal;

    # 这里做了一个求和运算只是一个例子，可以在这里完成一些数据库、
    # 缓存服务器的操作，达到基础模块和业务逻辑分离目的
    content_by_lua_block {
        local args = ngx.req.get_uri_args()
        ngx.say(tonumber(args.a) + tonumber(args.b))
    }
}

location = /app/test {
    content_by_lua_block {
        local res = ngx.location.capture(
                        "/sum", {args={a=3, b=8}}
                        )
        ngx.say("status:", res.status, " response:", res.body)
    }
}
```

### subrequest（子请求）
Nginx **子请求**是一种非常强有力的方式，它可以发起非阻塞的**内部请求**访问目标 location。

目标 location 可以是配置文件中其他文件目录，或 任何 其他 nginx C 模块，包括 ngx_proxy、ngx_fastcgi、ngx_memc、ngx_postgres、ngx_drizzle，甚至 ngx_lua 自身等等 。

需要注意的是，子请求只是模拟 HTTP 接口的形式， **没有** 额外的 HTTP/TCP 流量，也 没有 IPC (进程间通信) 调用。所有工作在内部高效地在 C 语言级别完成。

**语法：ngx.location.capture(uri, option?)** 


发送一个 POST 子请求，可以这样做：

``` nginx
 res = ngx.location.capture(
     '/foo/bar',
     { method = ngx.HTTP_POST, body = 'hello, world' }
 )
```

除了 POST 的其他 HTTP 请求方法请参考 [HTTP method constants][1]

`method` 选项默认值是 `ngx.HTTP_GET`。

`args` 选项可以设置附加的 URI 参数

如：

``` lua
 ngx.location.capture('/foo?a=1',
     { args = { b = 3, c = ':' } }
 )
```

 
 等于：
 

``` lua
 ngx.location.capture('/foo?a=1&b=3&c=%3a')
```


下面是一个简单例子：
``` nginx
 res = ngx.location.capture(uri)
```
返回一个包含四个元素的 Lua 表 (`res.status`, `res.header,` `res.body`, 和 `res.truncated`)。

`res.status` 保存子请求的响应状态码。

`res.header`用一个标准 Lua 表储子请求响应的所有头信息。如果是“多值”响应头，这些值将使用 Lua (数组) 表顺序存储。例如，如果子请求响应头包含下面的行：

``` http
 Set-Cookie: a=3
 Set-Cookie: foo=bar
 Set-Cookie: baz=blah
```


则 `res.header["Set-Cookie"]` 将存储 Lua Table `{"a=3", "foo=bar", "baz=blah"}`。

`res.body` 保存子请求的响应体数据，它可能被截断。用户需要检测 

`res.truncated` 布尔值标记来判断 res.body 是否包含截断的数据。这种数据截断的原因只可能是因为子请求发生了不可恢复的错误，例如远端在发送响应体时过早中断了连接，或子请求在接收远端响应体时超时。

以上是大致概念，我这里做了一个`ngx.location.capture` + `proxy_pass`

### 使用 ngx.location.capture + proxy_pass
假设这样一个情景，现需要将Nginx转发到站点获取信息，之后再对获取到的信息进行处理后输出。

对于一个外部接口，包含`/api/`字段，请求将该url转发，可以这么写：

``` nginx
location ~ /api/([-_a-zA-Z0-9/]+) {
    content_by_lua_file lua/capture.lua;
	# ........next step
}
```
capture.lua: 

``` lua
-- 因为 ngx.req.get_method()得到的是"GET"等字符串
-- 所以 使用 method_map[ngx.req.get_method()] 读出Method常数，称为HTTP method constants
local method_map = {
    ["GET"] = ngx.HTTP_GET,
    ["POST"] = ngx.HTTP_POST,
    ["HEAD"] = ngx.HTTP_HEAD,
    ["PUT"] = ngx.HTTP_PUT,
    ["DELETE"] = ngx.HTTP_POST,
	--其他Method省略
}

-- 读取URL参数
local args = ngx.req.get_uri_args()

-- 读取body
ngx.req.read_body()
local body = ngx.req.get_body_data()

-- 设置subrequest
local res = ngx.location.capture('/proxy' + ngx.var.uri, {
    -- ngx.var.uri = '/api/.....'
	-- 这里把获得的url直接扔给内部调用口/proxy
    method = method_map[ngx.req.get_method()],
    args = args,
    body = body
})
ngx.say(res.body)
```
内部调用口`/proxy`：

``` nginx
location ~ /proxy([-_a-zA-Z0-9/]+) {
# 这里的正则中将url中proxy后的部分放到了变量$1中
    rewrite_by_lua_block {
        ngx.req.set_uri(ngx.var[1], false)
		-- 重写url
		-- 在lua模块，取变量$1可以这么取：ngx.var[1]
    }
    proxy_pass http://192.168.79.128:1018;
	# 这样就可以访问：http://192.168.79.128:1018/api/.....了（url为/proxy后面的部分）
    proxy_set_header    Host $host;
    proxy_set_header    X-Real-IP   $remote_addr;
    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
}
```
### 利用cosocket
运用`resty.http`库资源，https://github.com/pintsized/lua-resty-http
结合《最佳实践》一书及`resty.http`官方说明，我对`capture.lua`改成如下：

``` lua
-- 读取URL参数
local args = ngx.req.get_uri_args()

-- 读取body
ngx.req.read_body()
local body = ngx.req.get_body_data()

local http = require "resty.http"
local httpc = http.new()
local res,err = httpc:request_uri(
    'http://192.168.79.128:1018/api/test', {
	-- 注意这里的method为method字符串，故不用method_map了。
    method = ngx.req.get_method(),
    args = args,
    body = body
})
if 200 ~= res.status then
    ngx.exit(res.status)
end
ngx.say(res.body)
```
### 说明
《最佳实践》一书中提到在对多次请求时的情境下，`resty.http`要较`ngx.location.capture` + `proxy_pass`更好。

[1]: https://github.com/openresty/lua-nginx-module#http-method-constants