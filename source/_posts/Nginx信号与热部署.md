---
title: Nginx信号与热部署
date: 2020-03-01 01:19:11
tags: ["nginx"]
---
Nginx的一个重要特性是支持热部署,这使得我们可以在不停止服务的同时升级Nginx。Nginx的另外一个特性是可扩展性好，所以有很多第三方模块。本文将以安装[echo模块](https://github.com/openresty/echo-nginx-module)为例展示Nginx热部署的步骤。
<!-- more -->

## 信号
我们经常使用`nginx -s signal`去控制nginx,实际上我们也可以使用操作系统信号(可以通过`kill -l`查看所有信号)去控制nginx,nginx主进程可接收的信号如下:

| 信号 | 对应nginx -s|功能|
| :----: | :----: | :----: |
|TERM, INT | stop| 快速停止nginx|
|QUIT | quit| 优雅停止nginx|
|HUP | reload| 重新加载配置文件|
|USR1 | reopen| 重新打开日志文件|
|USR2 | | 升级nginx可执行文件|
|WINCH | | 优雅关闭worker进程|

我们也可以通过信号去控制worker进程(除了上述的`HUP`和`USR2`信号),但一般比较少用。
## echo模块简介
echo模块是一个第三方模块,对输入输出、并行/串行子请求、timer和sleep等nginx内部API进行了封装。使用echo模块，我们可以很方便进行调试。在实际应用中,我们可以发现在以下场景中echo模块很有用:  
* 直接从内存中提供静态内容
* 使用自定义的头部和尾部包装上游的返回
* 合并几个子请求的内容到一个主请求中

## 安装nginx和相关模块
先看下已有的nginx版本及配置选项:
```bash
[root@VM_74_12_centos ~]# nginx -V
nginx version: nginx/1.16.1
built by gcc 4.8.5 20150623 (Red Hat 4.8.5-4) (GCC) 
configure arguments: --prefix=/usr/local/nginx
```

从[nginx官网](http://nginx.org/en/download.html)我们知道，截止到目前(2020-03-01)nginx的最新稳定版本是1.16.1,主线版本是	nginx-1.17.8,关于这两个版本的差异可查看[这里](https://www.nginx.com/blog/nginx-1-6-1-7-released/)。实际上官方推荐我们使用Mainline版本，因为加入了一些新特性和bugfixes,除非担心一些新特性可能会产生一些不良影响，例如与一些第三方模块不兼容。

由于[echo模块wiki](https://github.com/openresty/echo-nginx-module#compatibility)里nginx的兼容最高只到1.16.x。所以我们重新编译nginx并将echo模块集成进去。 

1. 下载并解压nginx
```
wget http://nginx.org/download/nginx-1.16.1.tar.gz

tar zxvf nginx-1.16.1.tar.gz
```
2. 下载并解压echo模块
```bash
wget https://github.com/openresty/echo-nginx-module/archive/v0.61.tar.gz

tar zxvf v0.61.tar.gz
```
3. 编译安装
```bash
// 进入nginx目录
cd nginx-1.16.1
// 执行configure
./configure --prefix=/usr/local/nginx  --add-module=/root/echo-nginx-module-0.61

成功之后会有如下提示:
configuring additional modules
adding module in /root/echo-nginx-module-0.61
 + ngx_http_echo_module was configured

 // 安装
 make && make install
```
以上步骤都成功之后我们查看nginx的sbin目录会发现nginx可执行文件已经替换成我们最新编译的,而老的nginx可执行文件已经重命名为nginx.old
```bash
[root@VM_74_12_centos ~/nginx-1.16.1]# ll /usr/local/nginx/sbin/
total 7848
-rwxr-xr-x 1 root root 4207472 Mar  1 12:32 nginx
-rwxr-xr-x 1 root root 3825493 Nov 19 13:49 nginx.old
```

*说明:从nginx 1.9.11版本开始,可以将echo模块编译为一个动态模块，具体可见官网文档*
## 热部署
安装完毕后,我们可以修改下原来的配置:
```nginx
server {
    listen       80;
    server_name  www.a.com;
    return 200 "www.a.com\n";
}
```
修改为:
```nginx
server {
    listen       80;
    server_name  www.a.com;
    location / {
        echo "www.a.com by echo";
    }
}
```
*说明:这里改成location是因为echo指令不支持在server块下配置*

修改完之后由于此时的nginx master进程还是基于老的可执行文件生成的，所以不能使用reload去重新加载配置文件,否则会提示错误:
> [emerg] 28354#0: unknown directive "echo" in /usr/local/nginx/conf/nginx_test.conf:71`

正确做法如下:
### 1. 先给原来的master进程发送一个USR2信号表明要升级可执行文件:
```bash
[root@VM_74_12_centos ~/nginx-1.16.1]# ps -ef | grep nginx
root     32183     1  0 13:58 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_test.conf
nobody   32184 32183  0 13:58 ?        00:00:00 nginx: worker process
root     32342 22840  0 13:59 pts/3    00:00:00 grep --color=auto nginx
[root@VM_74_12_centos ~/nginx-1.16.1]# kill -USR2 32183
[root@VM_74_12_centos ~/nginx-1.16.1]# ps -ef | grep nginx
root       361 22840  0 14:02 pts/3    00:00:00 grep --color=auto nginx
root     32183     1  0 13:58 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_test.conf
nobody   32184 32183  0 13:58 ?        00:00:00 nginx: worker process
root     32673 32183  0 14:01 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_test.conf
nobody   32674 32673  0 14:01 ?        00:00:00 nginx: worker process
```
可以看到,给老的master进程发送完USR2信号后,如果配置文件没有错误的话,将会生成新的master进程和worker进程。
这里需要注意的是:
* 新master进程的父进程是老master,而新worker进程的父进程是新master进程,因此所有这些进程都可以监听同一个端口(或者说socket/句柄),而不会冲突。
* nginx会将老master进程的pid文件加上.oldbin后缀,而新master进程的id还是保存在pid文件中。
```bash
[root@VM_74_12_centos ~/nginx-1.16.1]# ll /usr/local/nginx/logs/
total 956
-rw-r--r-- 1 nobody root 579995 Mar  1 14:07 access.log
-rw-r--r-- 1 root   root   2438 Nov 20 00:04 access1.log
-rw-r--r-- 1 nobody root 379146 Mar  1 14:01 error.log
-rw-r--r-- 1 root   root      6 Mar  1 14:01 nginx.pid
-rw-r--r-- 1 root   root      6 Mar  1 13:58 nginx.pid.oldbin
```
 此时虽然新的master进程已经启动,但是我们会发现，所有的worker进程(老的和新的)都会处理请求:
```bash
[root@VM_74_12_centos ~/nginx-1.16.1]# curl http://www.a.com
www.a.com
[root@VM_74_12_centos ~/nginx-1.16.1]# curl http://www.a.com
www.a.com by echo
```
注意:如果是用相对路径启动的nginx，那么发送USR2信号给老master进程会产生如下错误:
> [alert] 30483#0: execve() failed while executing new binary process "nginx" (2: No such file or directory)

原因是`execve()`函数不会去根据系统环境变量去查找nginx可执行文件,因此需要使用绝对路径启动nginx
### 2. 升级成功，给老master进程发送WINCH信号

前面我们说过,master进程接收到WINCH信号,会优雅地关闭其worker进程。
```bash
[root@VM_74_12_centos ~/nginx-1.16.1]# kill -WINCH 32183
[root@VM_74_12_centos ~/nginx-1.16.1]# ps -ef | grep nginx
root      3122 22840  0 14:19 pts/3    00:00:00 grep --color=auto nginx
root     32183     1  0 13:58 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_test.conf
root     32673 32183  0 14:01 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx_test.conf
nobody   32674 32673  0 14:01 ?        00:00:00 nginx: worker process
[root@VM_74_12_centos ~/nginx-1.16.1]# curl http://www.a.com
www.a.com by echo
[root@VM_74_12_centos ~/nginx-1.16.1]# curl http://www.a.com
www.a.com by echo
```
可以看到，发送WINCH信号给老的master进程后,老的worker进程将会优雅地退出(如果有还在处理的请求会等到处理完毕再退出)，之后就只有新的worker进程会处理请求。

这里需要注意的是老master进程并没有退出,并且会继续监听socke。这主要是为了当新的可执行文件不符合预期时，还可以重新使用老的可执行文件(相当于版本回退)。
### 3. 根据情况关闭老master进程或回滚
根据前面的说明,这里可能会有两种情况:
* 结果符合预期,给老master进程发送QUIT信号,老master进程将会退出，升级完成。
```bash
    [root@VM_74_12_centos ~/nginx-1.16.1]# kill -QUIT 32183
    [root@VM_74_12_centos ~/nginx-1.16.1]# ps -ef | grep nginx
    root      9191 22840  0 14:57 pts/3    00:00:00 grep --color=auto nginx
    root     32673     1  0 14:01 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/ngin
    x_test.conf
    nobody   32674 32673  0 14:01 ?        00:00:00 nginx: worker process
```
    
* 结果不符合预期，这时需要使用原来的可执行文件，有两种选择:
    * 给老master进程发送HUP信号。老master进程将基于原来的配置文件(不用重新读取)重新生成worker进程。之后发送QUIT信号给新master进程让其优雅退出。
    * 发送TERM信号给新master进程，它会发送消息给相应的worker进程要求其退出,之后自己也会退出。新master进程退出后，老master进程会自动生成新的worker进程。

## 总结
本文首先简单介绍了通过信号去控制nginx,然后详细说明了nginx热部署的步骤。nginx热部署特性，也称平滑升级,可以在升级期间继续处理请求，对于服务实现高可用，具有重大意义。
