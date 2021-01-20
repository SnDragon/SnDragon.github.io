---
title: OAuth2.0介绍与实践
date: 2020-04-07 22:47:36
tags: ["Laravel", "OAuth2.0"]
---
# 基于Laravel5.5+OAuth2.0实现Github/微博/QQ/微信登录
<p align="center"><img src="https://laravel.com/assets/img/components/logo-laravel.svg"><img src="https://oauth.net/images/oauth-logo-square.png"></p>

[OAuth2.0](https://en.wikipedia.org/wiki/OAuth)是目前业界关于授权的标准协议,使用OAuth,用户可以授权第三方应用获取一些特定的权限而无需输入相应的账号密码，并且可随时收回权限。例如我们常见的QQ登录/微信登录等,背后都采用了OAuth2.0协议。

有关OAuth的更多介绍,可以参考[这篇博客](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

本项目将基于Laravel框架和OAuth2.0实现Github/微博/QQ/微信登录。

<!-- more -->
## 最终效果
[查看视频](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/OAuth_results.mp4)

## 运行项目
1. 下载项目
```bash
git clone https://github.com/SnDragon/laravel-oauth.git
```
2. 安装依赖
```bash
cd laravel-oauth
composer install
```
3. 配置nginx
```
server {
        listen 80;
        # server_name可以自定义,与配置回调的域名一样即可
        server_name longerwu.oauth.com www.longerwu.com;
        # 替换为对应的项目路径
        root /Users/wuxilong/code/laravel-oauth/public;

        # 网站默认首页
        index index.php index.html index.htm;

        # 修改为 Laravel 转发规则，否则PHP无法获取$_GET信息，提示404错误
        location / {
                try_files $uri $uri/ /index.php?$query_string;
        }

        location ~ \.php$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }
}
```
4. 往hosts增加规则(非本地调试可忽略)
```
127.0.0.1 longerwu.oauth.com www.longerwu.com
```
5. 运行nginx
6. 复制一份`.env.example`到`.env`,并对OAuth的相关配置进行修改
7. 效果验证

本地访问`http://www.longerwu.com/`(替换为上面配置的域名),若出现如下效果,则运行成功:
![运行效果](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_start.png)

## Github接入
Github接入相对来说比较简单，无需审核,可直接创建。

[开发文档](https://developer.github.com/apps/building-oauth-apps/authorizing-oauth-apps/)
1. 申请应用&配置回调

打开[这个地址](https://github.com/settings/developers),点击右上角的`New OAuth App`按钮,填入相应信息即可获得`client_id`和`client_secret`

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_github1.png)

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_github2.png)

2. 将获取到的client_id和client_secret以及回调配置到.env中
```
GITHUB_CLIENT_ID=xxx
GITHUB_CLIENT_SECRET=xxx
GITHUB_CALLBACK_URL=xxx
```
3. 效果验证
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_github3.png)

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_github4.png)

## 微博登录
微博接入稍微麻烦一点,需要填写开发者资料,认证之后才能申请应用。

[开发文档](https://open.weibo.com/wiki/%E6%8E%88%E6%9D%83%E6%9C%BA%E5%88%B6)
1. [填写开发者资料](https://open.weibo.com/developers/basicinfo)
2. 申请应用&配置回调

填写应用基本信息:
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_weibo1.png)

填写应用高级信息(回调):
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_weibo2.png)

添加测试账号:
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_weibo3.png)

3. 将获取到的client_id和client_secret以及回调配置到.env中
```
WEIBO_CLIENT_ID=xxx
WEIBO_CLIENT_SECRET=xxx
WEIBO_CALLBACK_URL=xxx
```
4. 效果验证
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_weibo4.png)

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_weibo5.png)

## QQ互联接入
QQ登录应该说是我们最常用的登录方式之一,以前只需要在QQ互联申请应用,就算没通过审核也可以进行本地开发调试,可惜最近QQ互联开放平台貌似改了策略，只有备过案的网站才能接入OAuth2.0登录了,不知道后续还会不会放开,目前未备案通过的应用会报100008错误:
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_qq1.png)

[开发文档](https://wiki.connect.qq.com/%E5%BC%80%E5%8F%91%E6%94%BB%E7%95%A5_server-side)
1. [完善开发者资料](https://connect.qq.com/devuser.html#/)
2. [创建应用](https://connect.qq.com/manage.html#/)

注意,这一步需要填写网站备案号,所以需要先去备案,备案周期由于需要管局审核,可能会比较长(5个工作日以上),个人开发者主办单位名称填自己的名字,填写完相关信息之后等待审核通过即可(一般是一个工作日)

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_qq2.png)

3. 将获取到的client_id和client_secret以及回调配置到.env中
```
QQ_CLIENT_ID=
QQ_CLIENT_SECRET=
QQ_CALLBACK_URL=
```
4. 效果验证
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_qq3.png)

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_qq4.png)

## 微信接入

[开发文档](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/Wechat_webpage_authorization.html)


1. 应用申请

微信授权登录分两种:公众平台和开放平台,其中开放平台一般是给企业申请的，需要上传营业执照啥的，不适合个人开发者开发学习,而公众平台又分两种,订阅号和服务号,服务号一般也是给企业申请的,但订阅号又无法开通网页授权获取用户信息功能。

好在微信考虑到申请流程可能比较麻烦,为方便开发者,提供了一个沙盒环境,开发者可以在[这里](http://mp.weixin.qq.com/debug/cgi-bin/sandbox?t=sandbox/login)申请微信公众平台测试账号并进行开发

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_wechat1.png)


2. 回调域名配置
要想使用微信的OAuth2.0网页授权,我们还需要配置回调域名(注意这里只需填写域名即可,无需填写完整回调地址),找到`网页服务-网页账号-网页授权获取用户基本信息`,点击右边的修改按钮,填入域名即可
![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_wechat2.png)

3. 修改配置
```
WECHAT_CLIENT_ID=
WECHAT_CLIENT_SECRET=
WECHAT_CALLBACK_URL=
```
4. 效果验证

这里需要注意的是,微信公众平台的授权登录需要在微信客户端打开,开放平台则可以直接在PC网页打开扫描二维码登录。由于我们使用的是公众平台测试账号,所以也需要在手机打开,如果有线上域名和服务器,则直接部署到线上即可。

若没有线上环境或者想在本地调试的话,这里提供两种思路:
* 电脑安装手机模拟器进行调试
* 手机配置代理连接到本地电脑(需要和电脑在同一个局域网内)

例如Mac可以使用[charles](https://www.charlesproxy.com/)来配置代理,还可以用来抓包。

charlse配置:

打开`Proxy-Proxy Settings`,默认使用8888端口:

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/charlse_proxy.png )

手机配置(以苹果手机为例):
无线局域网-点击连着的wifi-最底下配置代理改为手动,服务器填电脑的IP,端口填上面charlse配置的端口:

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/charlse2.png )

配置完成后手机应该就可以正常访问我们的虚拟站点了。

用微信打开我们的网站,点微信登录:

![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_wechat3.png )


![image](https://raw.githubusercontent.com/SnDragon/laravel-oauth/master/images/oauth_wechat4.png )

## TODO
* 代码优化
* 制作composer包
