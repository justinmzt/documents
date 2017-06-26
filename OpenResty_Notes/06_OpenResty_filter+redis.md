# OpenResty_filter+redis
## 使用location.capture方法
（由于占内存过多，废弃了。）

过滤数据模块计算成本较高，需要对过滤完成的数据进行缓存。使用`restry.redis`模块操作`redis`对数据进行缓存处理。

`ngx_lua`中的`body_filter_by_lua*`模块中会将response拆包，缓存起来较为困难。故现先以`location.capture` + `proxy` 来实现。

以下代码在 `location = /api/test`中
``` lua
local redis = require "resty.redis"
local red = redis:new()

red:set_timeout(1000) -- 1 sec
local ok, err = red:connect("192.168.79.128", 6379) --连接redis
if not ok then
    ngx.say("failed to connect: ", err)
    return
end
--get，若有直接输出，若无访问过滤模块
--下面的字符串"b"，应是改成独一无二的key值，这里做测试用。
local bodyFromRedis, err = red:get("b")
if bodyFromRedis == ngx.null then
    -- 设置subrequest
    local res = ngx.location.capture('/proxy/api/test',{
        method = method_map[ngx.req.get_method()],
        args = args,
        body = body
    })
	--set，过期时间10秒
    ok, err = red:set("b", res.body, "EX", 10)
    if not ok then
        ngx.say("failed to set: ", err)
        return
    end
    local ok, err = red:set_keepalive(10000, 100)
    if not ok then
        ngx.say("failed to set keepalive: ", err)
        return
    end
    ngx.say(res.body)
else
    local ok, err = red:set_keepalive(10000, 100)
    if not ok then
        ngx.say("failed to set keepalive: ", err)
        return
    end
    ngx.say(bodyFromRedis)
end
```
以下是子请求`/proxy...`（做成过滤模块，代码与之前的类似）：

``` lua
location ~ /proxy([-_a-zA-Z0-9/]+) {
    # 只允许内部调用
    internal;
	
	rewrite_by_lua_block{
        ngx.req.set_uri(ngx.var[1], false)
    }
    proxy_pass http://192.168.79.128:1018;
    header_filter_by_lua_block {
        ngx.header.content_length = nil
    }
    body_filter_by_lua_block {
        local filter = require "tools.filter3"
        if(ngx.ctx.body_filter ~= nil) then
            ngx.arg[1] = ngx.ctx.body_filter..ngx.arg[1]
        end
        if(ngx.arg[2] == true) then
            ngx.arg[1] = ngx.ctx.body_filter
        end
        local tb = filter.gsplit(ngx.arg[1])
        local tb2 = filter.treeFilter(tb, body_filter_tree)
        ngx.arg[1] = table.concat(tb2)
        local len = string.len(ngx.arg[1])
        ngx.ctx.body_filter = string.sub(ngx.arg[1], len - 11, len)
        ngx.arg[1] = string.sub(ngx.arg[1], 1, len - 12)
    }
}
```

上方程序，外部接口先判断所GET信息是否缓存：如果缓存，直接输出；未缓存，请求内部接口`/proxy([-_a-zA-Z0-9/]+)`，代理到后台程序上获取数据库信息。

下图截自浏览器，反应一时间内的多个请求。


<img src="data:image/png;base64,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">

### 问题
1、是否需要用到`resty.http`模块。

后面准备尝试，通过`content`阶段设置共享变量`ngx.ctx.*`，来使`body_filter`阶段是否执行过滤。

2、还没考虑因为`redis`而内存是否会不够。



## 不经过body_filter_by_lua模块
先上代码（2017-05-17）：
`nginx.conf`中的一部分部分：
``` nginx
location ~ /api/([-_a-zA-Z0-9/]+) {
    content_by_lua_file lua/controllers/filter.lua;
}
```

`lua/controllers/filter.lua`文件：

``` lua
local http = require "resty.http"
local redis = require "resty.redis"

-- 读取URL参数
local args = ngx.req.get_uri_args()

-- 生成querystring
local querystring = ngx.encode_args(args)

local url = ngx.var.uri .. "?" .. querystring

--发送http请求
local httpc = http.new()
local res, err = httpc:request_uri(
    'http://192.168.79.128:1018'..ngx.var.uri, {
    method = ngx.req.get_method(),
    args = args,
    -- body = body
})
if 200 ~= res.status then
    ngx.exit(res.status)
end


--redis初始化
local red = redis:new()
local date = os.time()

--扫描每node个字符
local node = 20000

red:set_timeout(1000) -- 1 sec
local ok, err = red:connect("192.168.79.128", 6379) --连接redis
if not ok then
    ngx.say("failed to connect: ", err)
    return
end

--取过滤前响应值的md5值
local md5_before_filter = ngx.md5(res.body)

--获取响应值的md5值
local get_rst, err = red:hget(url, md5_before_filter)
if get_rst ~= ngx.null then
    ngx.say(res.body)
else
    --过滤模块
    local filter = require "tools.filter"
    local tb = filter.gsplit(res.body)
    local tb2 = filter.treeFilter(tb, body_filter_tree)
    local res_body_after_filter = table.concat(tb2)

    --取过滤后的响应md5值
    local md5_after_filter = ngx.md5(res_body_after_filter)

    if md5_before_filter == md5_after_filter then
        local hash_map = {}

        --存整体返回值的md5值
        hash_map[md5_after_filter] = date

        --存分段返回值的md5值，还未使用
        for i = 1,  math.ceil((#res.body) / node) do --(md5, date)键值对redis存储
            hash_map[ngx.md5(i..string.sub(res.body, (i - 1) * node + 1,i * node))] = i
        end

        red:hmset(url, hash_map)
        red:expire(url, 100)
        -- red:set(url, md5_before_filter, "EX", 100)
    end
    ngx.say(res_body_after_filter)
end
```

进入api之后，利用`resty.http`发送子请求，获取`res.body`。对`res.body`进行过滤。

如果过滤后无改变，取`res.body`的整体md5值进行存储。以便下次访问，跳过过滤阶段。

这里的过滤全部在`content_by_lua`模块下进行，不存在分块的问题，相对地，应该会占更多内存。