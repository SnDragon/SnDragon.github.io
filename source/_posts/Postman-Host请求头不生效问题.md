---
title: Postman Host请求头不生效问题
date: 2020-02-29 23:59:39
tags: ["http", "postman"]
---
个人开发或查看API接口经常会使用Postman来调试,特别是搭配[Postman Intercepter](https://chrome.google.com/webstore/detail/postman-interceptor/aicmkgpgakddgnaphhhpliifpcfhicfo),可以直接在postman中使用Chrome浏览器的cookie,非常方便。近日,在做[nginx实验](http://sndragon.com/2020/02/17/Nginx%E7%9A%84server_name%E5%92%8Clocation%E9%85%8D%E7%BD%AE/)时遇到一个问题:在Postman中设置了`Host`请求头没生效,这里记录下排查过程和原因。
<!-- more -->
## 问题现象
设置了Postman的Host请求头,如下图所示,怀疑该请求头不起作用。
![Postman设置](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/Postman_Host.png)

## 问题排查
### 用curl工具测试
用`curl`工具测试,则返回正常:
```bash
curl http://www.a.com -H "Host: www.b.com"

返回:default_server
```
`curl`返回结果正常,说明服务器配置正常,问题应该出在`Postman`上。

### 抓包查看请求报文
既然怀疑Postman Host请求头不生效,我们可以抓包看下原始的http请求报文,这里可以在客户端使用fiddler或tcpdump(类Unix系统)等工具抓包,当然也可以直接在服务端抓包。我们使用以下命令在服务端进行抓包:
```bash
tcpdump tcp and -i any host 服务端IP  and port 80 -A
```
使用上面的curl请求时,我们能获取到如下报文:
![curl tcpdump报文](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/curl_host_tcpdump.png)
可以看到,Host请求头正常生效

接着,再看下postman的报文:
![postman tcpdump报文](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/postman_tcpdump.png)
可见,Host请求头实际上并没有生效。

### 查阅资料
在网上查找了一下，比较相关的是这个[issue](https://github.com/postmanlabs/postman-app-support/issues/781),但是里面说使用Postman Interceptor扩展程序可以解决此问题,而我使用了还是有这个问题。

## 结论
某些版本的`Postman`设置Host请求头不会生效,可能是由于`Host`请求头比较特殊，可以用来实现虚拟主机,Http1.1规范中规定所有的请求都必须带上该请求头。其实,当我们使用`curl`不手动指定`Host`请求头时,会默认加上,值为请求URL中的域名,这个大家可以自己动手抓包看下。