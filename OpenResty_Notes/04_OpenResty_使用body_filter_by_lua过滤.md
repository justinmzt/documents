
## OpenResty_使用body_filter_by_lua*过滤
[toc]
### 建立过滤框架
首先，在子请求中`body_filter_by_lua*`也将会被调用。

据此，搭建起过滤用的`location`，其中包含proxy子请求。有时候会有头部信息Content-length，同时过滤时改变响应长度，与length不符。可以使用 `header_filter_by_lua*`过滤头部信息。


``` nginx
 location = /test {
     proxy_pass http://192.168.79.128:8080;
	 header_filter_by_lua_block {
	     ngx.header.content_length = nil
	 }
	 body_filter_by_lua_block {
	     ngx.arg[1] = ngx.re.gsub(ngx.arg[1], "o", "*")
	 }
 }
```


返回值为所有的字符"o"被字符"*"所替代。

``` 
Hell* W*rld.
```

### 过滤大型数据
往往我们不仅仅需要过滤几个单词，而是成篇的文章，整个页面的内容等。需要过滤庞大的数据。现在我们模拟庞大的数据量过滤，内容为Hello World的重复。

过滤结果:

    Hell* W*rld. Hell* W*rld. Hell* W*rld. Hell* W*rld. Hell* W*rld. Hell* W*rld. Hell* W*rld. .....

接着，改成过滤单词"Hello"，即修改"o"为"Hello"

结果：

    ......* World.* World.* World.Hello World.* World. * World.* World.* World.* World.* World. * World.* World.* 
	World.* World.* World. * World.* World.* World.* World. * World.* World.* World.* World. * World.* World.* ......
	
在过滤单词时，会出现其中的某一个"Hello"无法被过滤掉的问题。下面提供了一个解决方法。
### 分块与多次调用
>  response body may be delivered in chunks

官方引述了response body可能会分多块打包传送。接着我们就上面的代码对分块进行实验。

``` nginx
 location = /test {
     proxy_pass http://192.168.79.128:8080;
	 header_filter_by_lua_block {
	     ngx.header.content_length = nil
	 }
	 body_filter_by_lua_block {
	     ngx.arg[1] = string.len(ngx.arg[1]).." "
	 }
 }
```
修改`body_filter_by_lua_block`中的内容`ngx.arg[1] = string.len(ngx.arg[1]).." "`如果分块打包的话，我们就会把每个包内字符长度打印出来。

我的测试结果：

``` nginx
"8070 8192 8192 8192 4096 4096 8192 8192 8192 8192 4096 8192 8192 8192 8192 4096 13536 0 "
```
如果response body比较庞大，Nginx会自动分包，一般最后一个包（如果没有人为设置）长度为0。

来看看官方原理：
>The input data chunk is passed via `ngx.arg[1]` (as a Lua string value) and the "eof" flag indicating the end of the response body data stream is passed via `ngx.arg[2]` (as a Lua boolean value).

>Behind the scene, the "eof" flag is just the `last_buf` (for main requests) or `last_in_chain` (for subrequests) flag of the Nginx chain link buffers.

>输入数据块chunks要经过`ngx.arg[1]` (Lua 字符串)的过滤，"eof"标志指示response body数据流的末端通过`ngx.arg[2]` (Lua bollean值)表示。

>这里的"eof"标志就是主请求的`last_buf`或子请求的 `last_in_chain`。`"eof" == "true"` 表示此次nginx响应结束了。


以下原理阐述摘自[segmentfault, 《谈谈 OpenResty 中的 body_filter_by_lua*》——spacewander 2016年11月15日发布][111]

官方API文档中举了个例子：

``` nginx
 location /t {
     echo hello world;
     echo hiya globe;

     body_filter_by_lua {
         local chunk = ngx.arg[1]
         if string.match(chunk, "hello") then
             ngx.arg[2] = true  -- new eof
             return
         end

         -- just throw away any remaining chunk data
         ngx.arg[1] = nil
     }
 }
```
访问 /t 将只返回

``` nginx
hello world
```

`body_filter_by_lua*` 首次调用时，`ngx.arg[1]` 的值只是 `hello world`，不包括下面的 `hiya globe`。

不过初看下来，很难将`echo hello world`; 和 `delivered in chunks` 联系起来。这个 `chunk` 的大小是怎么确定的？看例子，应该跟`echo/ngx.say` 这一类输出方式有关。但是会不会跟输出的大小也有关？如果我一次性 `ngx.say` 了很多内容，是否会分成多个 `chunks` 发送？如果响应来自上游服务器，`chunks` 的数目又怎么定？

要回答这个问题，需要看看 `Nginx` 内响应内容的组织方式。`Nginx` 上游产生的内容，存储为 `ngx_chain_t` 类型的数据。这其实是一条 `ngx_buf_t` 链表。很容易可以想像到，这个链表就代表着**数据流**。上游产生的内容，像流水线上的包裹一样，不停地向下游传递。`output filter` 阶段像流水线上的机器，处理这些“包裹”。跟流水线上的机器不同的是，`Nginx` 中的 `output filter` 并非逐个处理这些“包裹”，而是一批一批地处理。上游成批成批地生产出这些包裹，每批包裹构成 `ngx_chain_t` 的子串，而 `output fiter` 则遍历这一子串，把其中的每个包裹打开处理。

想到 `body_filter_by_lua*` 其实属于 `output filter` 的一种，我们就回到了一开始讨论的问题。既然 `body_filter_by_lua*` 是一批一批处理上游的响应，那么它的调用次数就取决于上游的响应次数。上游的一次响应，如一次 `ngx.say`，会产生一个 `ngx_chain_t` 的子串（就ngx.say 而言，这个子串仅包含单个 `ngx_buf_t`）。至于响应的大小，最多只会影响到子串的长短，具体情况则取决于具体实现。

以上几个字符串会通过栈从 lua 域传递给 C 域。接着 OpenResty 计算它们的总长度，从 buffer chain 中找出一个空闲的大小合适的 `ngx_buf_t`，把它们拷贝进来。 之后就走 `http_output_filter` 把这个 `ngx_buf_t` （准确来说，是它所在的链表）发送出去。

那么，上游什么时候会把数据发完了？

Nginx 采用了一个 `last_buf` 的标志位，如果某个`ngx_buf_t` 是链表中的最后一个，跟上游交互的模块会设置这一个标志位为1. 映射回 OpenResty 的 lua 域，则是 `body_filter_by_lua*` 中的 `ngx.arg[2]`。你可能会注意到，`last_buf` 是一个设置在 `ngx_buf_t` 上的标志位，而传递给 `output filter` 的是 `ngx_chain_t`。OpenResty 把这一差别隐藏在实现之下——它会遍历当前输入的子串，如果某个 `ngx_buf_t` 存在 `last_buf`，那么就返回 `true`。

总结:
1. 往往一次Nginx请求会调用多次`output filter`，在这里即放到了`body_filter_by_lua*`，至少为两次，最后一次为传递结束位。
2. `body_filter_by_lua*`可以处理子请求。
3. 分块：在多次响应；response body比较大的情况下，Nginx会以数据流的形式将数据分块。

### 利用ngx.ctx解决过滤问题
``` lua
 location = /test {
     proxy_pass http://192.168.79.128:8080;
	 header_filter_by_lua_block {
	     ngx.header.content_length = nil
	 }
	 body_filter_by_lua_block {
	     if(ngx.ctx.body_filter ~= nil) then
		     ngx.arg[1] = ngx.ctx.body_filter..ngx.arg[1]
		 end
         ngx.arg[1] = ngx.re.gsub(ngx.arg[1], "Hello", "*")
		 if(ngx.arg[2] ~= true) then
		     local len = string.len(ngx.arg[1])
			 local max_len = string.len("Hello")
			 ngx.ctx.body_filter = string.sub(ngx.arg[1], len - (max_len - 1), len)
		     ngx.arg[1] = string.sub(ngx.arg[1], 1, len - max_len)
		 end
	 }
 }
```
设置一个变量`ngx.ctx.body_filter`用于存放每次打包的结尾字符，字符长度为所过滤字段（"Hello"）长度减一。
  [111]: https://segmentfault.com/a/1190000007483746
#### 流程图

``` flow
st=>start: 开始处理过滤请求
e=>end: 结束
filter=>operation: 准备处理一次包过滤
concat=>operation: 连接ngx.ctx.*
与当前响应包
do=>operation: 过滤
set=>operation: 设置末尾字符为
全局变量ngx.ctx.*，
长度为过滤字段
最长长度减一，
（作用域在本次请求内）
sub=>operation: 当前包内的字符串分割，去掉移到下个包的末尾字符
ngx_ctx=>condition: ngx.ctx.*
是否存在？
end=>condition: 是否继续循环？

st->filter->ngx_ctx
concat->do
do->end
ngx_ctx(yes)->concat
ngx_ctx(no)->do
end(no)->e
end(yes)->set
set->sub->filter
```
整体思路：每次包内容**在过滤后**截取末尾内容，将这部分内容连接到下一次请求进行过滤。

### 针对多个字段过滤
对于现有的Lua里String包中的接口，可以对单个词进行高效的过滤。也仅仅是对于单个词，放到for循环扫描的话，整体过滤的时间也会随词数增加正比增加。需要用一些数据结构（如字典树）来使遍历扫描更为快捷。

### Trie树
Trie树也叫字典树，就是将单词拆开成字符，生成一棵树，树的每一个节点代表一个字符，每一条边为一个单词。从而达到字典的效果。
直接搬一个实例。
以下是需要过滤的字段：
["inter", "instance", "string", "strong", "strang", "development"]

将上述字段转换成一个前缀Trie树，使用js对象就是如下结果：
``` json
{
    "i": {
        "n": {
            "t": {
                "e": {"r": {"en": 1}}
            }, 
            "s": {"t": {"a": {"n": {"c": {"e": {"en": 1}}}}}}
        }
    },
    "s": {
        "t": {"r": {
            "i": {"n": {"g": {"en": 1}}},
            "o": {"n": {"g": {"en": 1}}},
            "a": {"n": {"g": {"en": 1}}}}}
    },
    "d": {
        "e": {"v": {"e": {"l": {"o": {"p": {"m": {"e": {"n": {"t": {"en": 1}}}}}}}}}}
    },
}
```
通过递归Trie树以辨别是否需要被过滤。

### 生成Trie树
以下我使用Lua实现的Trie树的生成：

``` lua
local function generateTree(branch, str, i)
    local len = string.len
    local sub = string.sub -- 用于取字符
    if not branch[sub(str,i)] then -- 如果不存在下一个节点，则创建
        branch[sub(str, i)] = {}
    end
    if(i == len(str)) then -- 递归结束条件，达到最后一位
        branch[sub(str, i)]["en"] = 1
    else
        generateTree(branch[sub(str,i)], str, i + 1)
    end
end

function generate(table) -- 循环使每个字段都加入字典树中
    local tree = {}
    for k,v in ipairs(table) do
        generateTree(tree, v, 1)
    end
    return tree
end
--  调用
local tree = {}
local table = {"inter", "instance", "string", "strong", "strang", "development"}
local body_filter_tree = generate(table) --生成树
```
以上为通过递归生成Trie树过程。

### 字符串分割成数组

``` lua
function gsplit(str)
    local str_tb = {}
    if string.len(str) ~= 0 then
        for i=1,string.len(str) do
            str_tb[i] = string.sub(str,i,i) -- 插入数组
        end
        return str_tb
    else
        return nil
    end
end
```
上述方法，插入数组的两种方法：
一种是上面的`str_tb[i] = string.sub(str,i,i)`
另一种是`table.insert(str_tb, string.sub(str,i,i))`
前一种性能更好。分配了键后存储，比未分配键存储快。

### 数组连接成字符串

使用Lua的Table库里的函数`table.concat`

### 递归遍历Trie树

``` lua
function treeFilter(testTable, tree)
    local len = table.getn
    local rstStr = {}
    local i = 1    -- 输入数组循环变量
    local j = 1    -- 输出数组循环变量
    while i <= len(testTable) do -- 循环每个字符，利用字典树匹配
        local x = checkBranch(tree, testTable, i) -- 得到匹配到单词时字符所在数组内的位置
        if not x then --如果未匹配到复制当前单个字符到输出数组
            rstStr[j] = testTable[i]
            j = j + 1
            i = i + 1
        else -- 如果匹配到，输出数组插入"*"，输入数组循环变量加到x的值
            rstStr[j] = "*"
            j = j + 1
            i = x + 1
        end
    end
    return rstStr
end

function checkBranch(branch, tab, i) -- 根据树枝查询匹配，若无符合返回false，若匹配到了返回字符在数组里的位置
    if not branch[tab[i]] then -- （贪婪匹配，尽量匹配最长单词）如果没有下一个节点返回false
        return false
    else
        local x = checkBranch(branch[tab[i]], tab, i + 1) -- 首先递归下一个节点是否存在
        if not x then
            if branch[tab[i]]["en"] then -- 如果下一个节点不存在，返回上一个字符判断单词是否结束
                return i -- 结束返回字符位置
            else
                return false -- 否则返回false
            end
        else
            return x -- x如果存在就是树枝后面匹配到单词时的该字符在数组里的位置
        end
    end
end

--  调用
local tb2 = treeFilter(tb, tree)
```
### 树的初始化
在Nginx中，可以在`init_by_lua*`中进行全局变量的设置。我们可以把树的初始化放在其中。
我现在将方法封装成了一个方法：
####  generate
使用：generate(*lua-table*)

作用：根据table生成树

用法示例：

``` lua
local tree = require "tools.tree"
body_filter_tree = tree.generate(tree.table)
```
#### 运用到`init_by_lua*`中初始化树

``` nginx
init_by_lua_block {
    local tree = require "tools.tree"
    body_filter_tree = tree.generate(tree.table)
}
server {
    location = /test {
	    proxy_pass http://192.168.79.128:8080;
        header_filter_by_lua_block {
            ngx.header.content_length = nil
		}
        body_filter_by_lua_block {
            -- 使用body_filter_tree
		}
	}
 }
```
