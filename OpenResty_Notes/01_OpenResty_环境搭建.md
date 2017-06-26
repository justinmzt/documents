# OpenResty_环境搭建

OpenResty版本：1.11.2.2

Linux：Centos 7
## 下载安装
### 1.源码安装 
[参考官网安装方法][1]
#### 1.1 [下载 openresty-1.11.2.2.tar.gz ][2]
#### 1.2 根据代码示例安装

``` bash
tar -xzvf openresty-openresty-1.11.2.2.tar.gz 
cd openresty-1.11.2.2/
./configure
make
sudo make install
```

### ２.linux安装（官方推荐，我是用的源码安装，还未尝试）
[参考官网linux安装方法][3]

【摘自官网】：

　　你可以在你的 CentOS 系统中添加 `openresty` 资源库，这样就可以方便的安装我们的包，以后也可以更新（通过 `yum update` 命令）。添加资源库，你只用创建一个名为 `/etc/yum.repos.d/OpenResty.repo` 的文件，内容如下:

``` ini
 [openresty]
name=Official OpenResty Repository
baseurl=https://copr-be.cloud.fedoraproject.org/results/openresty/openresty/epel-$releasever-$basearch/
skip_if_unavailable=True
gpgcheck=1
gpgkey=https://copr-be.cloud.fedoraproject.org/results/openresty/openresty/pubkey.gpg
enabled=1
enabled_metadata=1
```
你也可以直接运行 `sudo yum-config-manager --add-repo https://openresty.org/yum/centos/OpenResty.repo` 添加该文件。

中国大陆的用户可以把 `baseurl` 改成下面的链接，速度会更快。

``` bash
baseurl=https://openresty.org/yum/openresty/openresty/epel-$releasever-$basearch/
```

或者运行 `sudo yum-config-manager --add-repo https://openresty.org/yum/cn/centos/OpenResty.repo` 添加对应的文件。

列出 `openresty` 资源库里面所有的包:

``` bash
sudo yum --disablerepo="*" --enablerepo="openresty" list available
```

然后你可以安装一个包，比如安装 `openresty`, 像这样:

``` bash
sudo yum install openresty
```

在 [OpenResty RPM][4] 包 页面能看到这些包更多的细节。


## Hello World！
### 1. 添加环境变量

``` bash
vi /etc/profile
```

在`profile`的末尾添加nginx路径：

``` bash
export PATH=/usr/local/openresty/nginx/sbin:$PATH
export PATH=/usr/local/openresty/bin:$PATH
```

可通过`echo $PATH`命令查看是否添加成功。

 
### 2.nginx.conf

推荐使用WebStorm，JetBrains的产品的插件是通用的。

#### 2.1 安装插件
##### 2.2.1 安装方法：

方法一：

需要安装的插件是OpenResty及Lua插件。

``` elixir
File-->Settings-->Plugins-->Browse repositories...-->(搜索所需插件安装)
```

方法二：
[IDEA的nginx插件下载][5]

看到 Change logs中：
该插件中包含有 OpenResty 的 Keywords；还有lua内嵌语法！；

若无Lua插件，还需下载安装[Lua][6]插件。

`File-->Settings-->Plugins-->Install plugin from disk...-->Restart IntelliJ IDEA`


##### 2.2.2 如果文件没有关联，可以在
`File-->Settings-->Editor-->File Types`中找到`Nginx Config` 
在其中添加nginx.conf。

之后就可以在IntelliJ IDEA 或 WebStorm下编辑nginx配置文件啦！



#### 2.2 Hello world!
打开`nginx.conf`，写入以下内容

``` nginx
worker_processes  1;        #nginx worker 数量
error_log logs/error.log;   #指定错误日志文件路径
events {
    worker_connections 1024;
}

http {
    server {
        #监听端口，若你的8080端口已经被占用，则需要修改
        listen 8080;
        location  = /hello {
            default_type text/html;

            content_by_lua_block {
                ngx.say("HelloWorld")
            }
        }
    }
}
```

如果成功的话，访问后会得到返回

```
Hello world!
```

##  OPM - OpenResty Package Manager
[opm documentation][7]

官方开发的OpenResty包管理工具，可以用来发布和安装社区里的OpenResty库。

[opm.openresty.org][8]是官方包分享的发布网站。

``` bash
$ which opm
/usr/local/openresty/bin/
```


``` nginx
# show usage
opm --help

# search package names and abstracts with the user pattern "lock".
opm search lock

# search package names and abstracts with multiple patterns "lru" and "cache".
opm search lru cache

# install a package named lua-resty-foo under the name of some_author
opm get some_author/lua-resty-foo

# get a list of lua-resty-foo packages under all authors.
opm get lua-resty-foo

# show the details of the installed package specified by name.
opm info lua-resty-foo

# show all the installed packages.
opm list

# upgrade package lua-resty-foo to the latest version.
opm upgrade lua-resty-foo

# update all the installed packages to their latest version.
opm update

# uninstall the newly installed package
opm remove lua-resty-foo
```

## 目录说明

目前我上传的文件与OpenResty安装根目录对应。

还未安装的包可以使用`opm get lua-resty-foo`命令来安装。


  [1]: https://openresty.org/cn/installation.html
  [2]: https://openresty.org/download/openresty-1.11.2.2.tar.gz
  [3]: https://openresty.org/cn/linux-packages.html
  [4]: https://openresty.org/cn/rpm-packages.html
  [5]: https://plugins.jetbrains.com/plugin/4415-nginx-support
  [6]: https://plugins.jetbrains.com/plugin/5055-lua
  [7]: https://github.com/openresty/opm
  [8]: https://opm.openresty.org