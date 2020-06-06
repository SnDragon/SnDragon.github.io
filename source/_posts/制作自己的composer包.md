---
title: 制作自己的composer包
date: 2020-06-06 11:18:00
tags: ["php", "composer"]
---
[Composer](https://getcomposer.org/)是现代PHP的依赖管理器,地位相当于js的npm,java的maven/gradle,python的pip等，可以让我们快速利用已有的组件库,而无需重复造轮子。想使用其他人开发的包时,可以使用`composer require 包名`的方式引入。这里记录下如何制作自己的composer包,为开源社区献出自己的一份力。
<!--more-->
## 开发自己的composer包
### 1. 项目初始化&引入依赖

假设我们要开发一个能从给定URL数组中检测无效URL的功能，并共享给其他人用，首先我们需要将自己的项目声明为一个composer包,新建项目文件夹，然后执行`composer init`
填好必要信息后,会在项目根目录下生成`composer.json`文件,我们可以通过`composer require`引入我们需要的其他依赖,例如如果我们需要用到[guzzle](http://docs.guzzlephp.org/en/stable/)(一个优秀的PHP HTTP客户端),可以使用`composer require guzzlehttp/guzzle`来引入。
安装成功后，会发现多了一个vendor文件夹，里面是我们所安装的依赖,对应着`composer.lock`文件也会更新，记录着我们本次所安装依赖的具体版本号，校验和等。

### 2. 开发代码
根据自己的喜好新建目录并开发代码,例如,我们在src/Url下面新建一个Scanner类:
```php
<?php

namespace Longerwu\Util\Url;

use GuzzleHttp\Client;

class Scanner
{
    /**
     * @var array URL数组
     */
    protected $urls;
    /**
     * @var Client
     */
    protected $httpClient;

    /**
     * Scanner constructor.
     * @param array $urls 要扫描的url数组
     */
    public function __construct(array $urls)
    {
        $this->urls       = $urls;
        $this->httpClient = new Client();
    }

    /**
     * 返回死链
     * @return array
     */
    public function getInvalidUrls()
    {
        $invalidUrls = [];
        foreach ($this->urls as $url) {
            try {
                $statusCode = $this->getStatusCodeForUrl($url);
            } catch (\Exception $e) {
                $statusCode = 500;
            }
            if ($statusCode > 400) {
                $invalidUrls[] = [
                    'url'    => $url,
                    'status' => $statusCode
                ];
            }
        }
        return $invalidUrls;
    }

    /**
     * 获取指定URL的HTTP状态码
     * @param $url
     * @return int
     */
    protected function getStatusCodeForUrl($url)
    {
        $response = $this->httpClient->head($url);
        return $response->getStatusCode();
    }
}
```

### 3. 修改composer.json的PSR4设置
注意上面类的命名空间我们可以自己定义，为了能让我们的类能被自动加载，需要修改composer.json的自动加载配置:
```
"autoload": {
    "psr-4": {
        "Longerwu\\Util\\": "src/"
    }
}
```
这样，后续我们可以通过`\Longerwu\Util\Url\Scanner`来访问到我们的类
### 4. 打tag并上传到github
在github新建一个仓库后，我们把本地代码提交到远程,并打一个tag:
```
git init
git remote add origin https://github.com/SnDragon/scanner.git
git pull origin master
git add .
git commit -m "init" 
git push
git tag v1.0.0
git push --tag
```
注意，一般来说，为了保持项目的简洁,我们不会上传本地的vendor目录,由于composer.lock已经记录了相关依赖的版本号,所以其他人安装依赖时会跟我们保持一致，不会出现版本不同导致出现兼容问题的情况。要忽略指定文件或文件夹，我们可以在根目录下建一个`.gitignore`文件:
```
.idea/
vendor/
```
这样git add的时候就会忽略这些文件/目录。
最后的项目目录:
```
tree -I 'vendor'
.
├── README.md
├── composer.json
├── composer.lock
└── src
    └── Url
        └── Scanner.php
```

### 5. 提交我们的包
打开[packagist官网](https://packagist.org/),若之前没注册过,则需注册一下,之后在打开submit页面输入包的github地址并提交:
![packagist注册包](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/packagist.png)
提交成功后可以看到:
![packagist注册包成功](https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/packagist_scanner.png)

## 使用我们的包

新建一个项目,使用composer init后通过composer requrie引入我们的包:
```
composer require longerwu/scanner
```
新建一个main.php来测试:
```php
<?php
require_once "vendor/autoload.php";

$urls = [
    "http://www.baidu.com",
    "http://www.qq.com",
    "http://www.longerwu.com",
    "http://www.test.com"
];

$scanner = new \Longerwu\Util\Url\Scanner($urls);
var_dump($scanner->getInvalidUrls());
```
执行`php main.php`可以看到非法的URL被打印出来了。