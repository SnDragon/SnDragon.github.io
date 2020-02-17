---
title: Nginx的server_name和location配置
date: 2020-02-17 22:07:53
tags: ["nginx", "server"]
---
[Nginx](http://nginx.org/)是目前最流行的Web服务器,由于具备高性能、高可靠以及支持热部署等特性被人们所青睐。Nginx用途广泛,其可作为静态资源服务器，也可充当代理服务器(HTTP/TCP/UDP/MAIL等)，还可以用来实现一些简单的API服务。Nginx主要是通过其配置文件(一般名为`nginx.conf`)来控制它的行为，本文主要介绍其http模块下的 `server_name`和`location`这两条指令的配置。
<!-- more -->
## server指令块与虚拟主机
虚拟主机是一种在单一主机或主机群上运行多个网站或服务的技术，可以用来解决IP地址资源有限而网站数目日益增多的问题。实现方式主要有以下三种:
* 基于域名(Name-based)
* 基于IP地址(IP-based)
* 基于Port端口(Port-based)

其中使用最广泛无疑是基于域名的方式,不同的域名通过DNS最终可以解析到相同的IP地址,在对应的机器上我们可以使用Nginx等Web服务器软件对不同的域名请求进行相应的处理。这里再提及一点,我们平时访问一个网站，是通过DNS将其解析到某一个IP上,我们的客户端(通常是浏览器)最终是和这个IP对应的机器建立连接，从而发送请求的。那么Nginx等服务器是如何知道一个请求对应的是哪个域名的呢？

答案在于HTTP协议中的Host请求头,其值为我们要访问的域名。这里需要注意的是,在HTTP/1.0中是不支持Host请求头字段的,所以HTTP/1.0是不支持虚拟主机技术的，而根据[rfc2616规范](https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html)HTTP/1.1协议中客户端发送的请求必须带上Host这个请求头,否则服务器必须返回`400 Bad Request`响应。

而nginx正是通过http模块下的server指令块来配置虚拟主机。

## server_name指令
### 配置语法:
```
Syntax:	server_name name ...;
Default:	
server_name "";
Context: server
```
### server_name形式
sever_name指令后面的参数值可以是以下几种:
* 精确的域名,例如`www.example.com`
* 通配符名称,可用\*表示任意多字符(类似Linux  Shell中的\*),但是通配符必须在域名的最前面或者最后面,例如`*.example.com`、`www.example.*`
* 正则表达式,最前面是一个波浪号~,例如`~^www\d+\.example\.com$`表示可以匹配以www开头，后跟一个到多个数字，然后以.example.com结尾的域名

除了以上几种形式，还有下面几种表示特殊含义的域名:
* `.example.com`,相当于`*.example.com`+`example.com`
* "",可以匹配没有带Host头的请求
* 国际化域名(用得不多,了解即可),用ASCII码表示，例如`xn--e1afmkfd.xn--80akhbyknj4f`可表示`пример.испытание`
* `_`、`__`或者`!@#`等无效的域名，可以理解为其可以匹配任意域名，但是优先级最低，最常见的用法是用来设置默认的server,即当一个请求的Host没有命中其他规则时，会采用默认server的配置。配置如下:
```
server {
    listen       80  default_server;
    server_name  _;
    return       444;
}
```
### server_name匹配顺序
当需要决定采用哪个server块的配置处理请求时,会根据以下的顺序查找:
1. 精确匹配
2. 以*开头的最长通配符名称
3. 以*结尾的最长通配符名称
4. 根据在配置文件出现的顺序第一个匹配上的正则表示式名称
5. 默认配置，在listen指令中指明了default_server的server块，若无，为配置文件中第一个声明的server块

示例,假设nginx只有以下server配置:
```
    # 这里主要是方便下面输出结果可以直接在浏览器显示
    default_type  text/plain;
    # 这里使用geo指令主要是为了输出$,直接在return输出$会报错
    # 参见https://stackoverflow.com/questions/57466554/how-do-i-escape-in-nginx-variables
    geo $dollar {
        default "$";
    }

    server {
        listen 80;
        server_name ~^www\.a\..*$;
        return 200 "~^www\.a\..*$dollar";
    }

    server {
        listen 80;
        server_name ~^.*a\..*$;
        return 200 "~^.*a\..*$dollar";
    }

    server {
        listen 80;
        server_name www.code.a.*;
        return 200 "www.code.a.*";
    }

    server {
        listen 80;
        server_name *.a.com;
        return 200 "*.a.com";
    }

    server {
        listen       80;
        server_name  www.a.com;
        return 200 "www.a.com";
    }
```
在hosts文件上加上以下配置:
```
127.0.0.1 www.a.com www.code.a.com www.code.a.cn www.a.oa.com dev.a.cn www.b.com
```
我们可以直接用浏览器访问或者借助curl工具来进行测试,测试结果如下,可对照上面的查找顺序进行分析:


| input | output|匹配类型|
| :----: | :----: | :----: |
| http://www.a.com | www.a.com| 精确匹配 |
| http://www.code.a.com | *.a.com| 前导*匹配 |
| http://www.code.a.cn  | www.code.a.*| 后导*匹配 |
| http://www.a.oa.com | `~^www\.a\..*$`| 正则匹配 |
| http://dev.a.cn| `~^.*a\..*$` | 正则匹配 |
| http://www.b.com| `~^www\.a\..*$` | 默认匹配 |


值得说明的是，由于上面的配置没有显示指定默认server,所以会默认匹配到第一个配置,假如我们在配置最后再添加如下配置:
```
    server {
        listen 80 default_server;
        server_name _;
        return 200 "default_server";
    }
```
重启后，再访问http://www.b.com,会输出default_server,其他访问结果不变。注意这里的`default_server`是配置在`listen`指令下的。

关于listen指令,有几点需要注意的地方:
1. 如果server指令块里没有指定listen指令,则根据运行nginx的用户不同，默认监听的端口也不同,root用户启动默认监听80端口，否则默认监听8000端口
2. 如果配置了listen且只指定了IP,则监听端口为80,此时操作系统可能会不允许非root用户启动nginx，提示
```
nginx: [emerg] bind() to 127.0.0.1:80 failed (13: Permission denied)
```
3. 以上说的配置查找规则前提是请求需要跟listen指令配置的IP跟端口相匹配
关于以上注意事项，这里举两个例子:

* 假设运行nginx的是非root用户,且上面最后我们加的配置里listen指令没有指定80端口,即：
```
    server {
        listen default_server;
        server_name _;
        return 200 "default_server";
    }
```
这时访问http://www.b.com,由于上面这个server监听的是8000端口，跟请求的80端口不匹配,结果将会变回`~^www\.a\..*`  
* 假设最后的默认server配置改成如下配置(注意端口前有IP):
```
    server {
        listen 公网IP:80 default_server;
        server_name _;
        return 200 "default_server";
    }
```
这时如果是在公网访问的话，不管访问上面的哪个域名都会返回"default_server"，理由是不设置IP的话nginx默认会监听该机器的所有IP的特定端口,设置了的话只会监听该IP的特定端口。
本地访问同理，不能匹配到listen了公网IP的server。

## location 配置
了解完server_name和listen的配置规则，我们知道了一个请求过来会对应哪个server。接下来我们要讨论的是某个server下不同请求URI对应的location配置查找规则。
### 配置语法
```
Syntax:	location [ = | ~ | ~* | ^~ ] uri { ... }
location @name { ... }
Default: —
Context: server, location
```
根据配置语法我们知道location可以有以下几种形式:
* =,精确匹配
* ～,正则匹配,大小写敏感
* ～*,正则匹配, 大小写不敏感
* ^~，忽略正则表达式的前缀匹配
* 没有修饰符,前缀匹配
* @，命名location,可用来做内部重定向

> 其中=和^~修饰符都可以认为是特殊形式的前缀匹配

### 匹配过程
根据请求的URI和location的配置,查找请求对应的location过程如下:
1. 将请求URI标准化,包括将"%xx"形式编码的文本进行解码，解析相对路径"."和"..",以及合并两个或多个相邻的"/"成单个"/"
2. 根据请求URI找到并记录匹配上的最长前缀匹配，这里有两个特殊的场景:
    1. 找到了=修饰的精确匹配,结束查找,采用它的配置
    2. 如果该步骤最终记录下的前缀以^~修饰，则采用它的配置，不会进行后续的查找步骤
3. 根据在配置文件出现的顺序，检查相应的正则匹配，若有一个匹配上，则应用该配置，且不会继续检查后续的正则配置
4. 若第3步没有找到匹配上的正则匹配，则采用第2步中找到的最长前缀匹配对应的配置

根据上面的查找过程，可以得到一些配置优化点：
* 对于经常要访问的路径，可以使用精确匹配或^=修饰的匹配,可以避免进行正则匹配检查
* 如果一定要用到正则表达式，可以把最经常被访问的location规则配置在最前面，因为正则匹配命中一个就不会继续验证后续的匹配规则

### 示例
假设有如下配置:
```
    server {
        listen 80 default_server;
        server_name _;
        # A
        location = / {
                return 200 "A";
        }

        # B
        location / {
                return 200 "B";
        }

        # C
        location /docs {
                return 200 "C";
        }

        # D
        location ^~ /imgs {
                return 200 "D";
        }
        
        # E
        location ~* \.(gif|jpg|png)$ {
                return 200 "E";
        }

        # F
        location ~ /a/.*$ {
                return 200 "F";
        }
    }
```
测试结果如下:

| input | output | 说明 |
|:---:|:---:|:---:|
|http://127.0.0.1| A | 匹配到A跟B,精确匹配优先级较高|
|http://127.0.0.1/test| B | 只匹配到B|
|http://127.0.0.1/docs/1| C | 匹配到B跟C,C前缀比B长|
|http://127.0.0.1/docs/2.jpg| E |  匹配到B、C、E，正则匹配比普通前缀匹配优先级高|
|http://127.0.0.1/imgs/1| D | 只匹配到B、D,D前缀比B长|
|http://127.0.0.1/imgs/1.jpg| D | 匹配到B、D、E，由于D是最长匹配且有^~修饰符，所以不会再检查正则匹配|
|http://127.0.0.1/docs/a/1| F | 匹配到B、C、F|

关于最后一条测试结果,需要注意的是，`/a/.*$`这个正则表达式,并不要求请求URI以`/a`开头，这也是很容易疏漏的地方,若想匹配以`/a`开头的请求，应改为`^/a/.*$`,此时最后一条测试结果会变为C

### location @name的用法
`@`前缀可以用来定义一个命名的location,该location不处理正常的外部请求,一般用来供内部重定向使用。它们不能嵌套,也不能包含嵌套的location。
例如:
```
location /try {
    try_files $uri $uri/ @name;
}

location /error {
    error_page 404 = @name;
    return 404;
}

location @name {
    return 200 "@name";
}
```
这时访问`/try`或者`/error`都会返回"@name"


## 总结
本文主要介绍了nginx关于`server_name`和`location`的配置以及匹配规则,并举例说明。`server_name`和`location`指令是nginx中非常重要的两条指令，掌握这两条指令对于我们配置nginx以及排查问题都是非常重要的，希望本文能帮到大家。

## 参考文档
* [How nginx processes a request](http://nginx.org/en/docs/http/request_processing.html)
* [Server names](http://nginx.org/en/docs/http/server_names.html)
* [Module ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html#location)
* [How do I escape $ in nginx variables](https://stackoverflow.com/questions/57466554/how-do-i-escape-in-nginx-variables)
