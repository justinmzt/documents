# Lua_Notes

![enter description here][3]

## 语法笔记

 1、`ngx.null`用于表示不同于`nil`的“空值”
 
 2、Lua中只有`nil`和`false`为”假“
 
 3、应尽量使用“局部函数”，local
 
 4、函数内定义变量需要用`local`
 
 5、删除数组元素，应该使用`remove`
 
 6、不等号： `~=`
 
 7、`elseif`是连在一起的。
 
 8、只有`break`来跳出循环，没有`continue`
 
 9、repeat循环：

``` lua
    repeat
    print(x)
    until true
```
直到until后条件为true为止。
 
 10、Table（表）是一种“关联数组”。默认为数字索引，索引可以定义为字符串。
 
 定义：`local corp = { }`
``` lua
 local corp = {
    web = "www.google.com",   
	--索引为字符串，
	--key = "web", value = "www.google.com"
    
	staff = {"Jack", "Scott", "Gary"}, 
	--索引为字符串，值也是一个表
    
	100876,
	--相当于 [1] = 100876，此时索引为数字
    --key = 1, value = 100876
	
    100191,
	--相当于 [2] = 100191，此时索引为数字
	
    [10] = 360,
	
	--直接把数字索引给出
}
```
``` lua
print(corp.web)               -->output:www.google.com
print(corp[2])                -->output:100191
print(corp.staff[1])          -->output:Jack
print(corp[10])               -->output:360
```

想了解更多关于 table 的操作，请查看 [Table 库][6] 章节。

 11、 函数的定义

``` lua
 function function_name (arc)  -- arc 表示参数列表，函数的参数列表可以为空
   -- body
end
```
等价于

``` lua
function_name = function (arc)
  -- body
end
```
局部函数：

``` lua
local function function_name (arc)
  -- body
end
```

  [3]: http://www.lua.org/images/lua.gif
  [6]: https://moonbingbing.gitbooks.io/openresty-best-practices/content/lua/table_library.html
