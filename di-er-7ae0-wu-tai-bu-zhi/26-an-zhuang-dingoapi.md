## 什么是 DingoApi

> 这里有个视频教程，可以参考一下：[https://learnku.com/courses/laravel-package/2466/api-development-kit-dingoapi](https://learnku.com/courses/laravel-package/2466/api-development-kit-dingoapi)。

[dingo/api](https://github.com/dingo/api/)是一个 Lumen 和 Laravel 都可用的 RestFul 工具包，帮助我们快速的开始构建 RestFul Api。我们的目的是教会大家如何快速的搭建并使用这个包，更多的功能，还需要你仔细阅读 DingoApi 的[文档](https://github.com/dingo/api/wiki)来深入的学习和理解，[这里](https://github.com/liyu001989/dingo-api-wiki-zh)有一份中英对照的翻译，或许能帮到你。

## 1. 安装

因为是 API 的课程，可以先创建一个 API 分支。

```
$ git checkout -b api
```

安装扩展包

```
$ composer require dingo/api:2.0.0-beta1
```

> dingo/api 已经发布了正式版本，目前最新版本为`2.2.3`，但是存在一个较为严重的 Bug：[https://github.com/dingo/api/issues/1645](https://github.com/dingo/api/issues/1645)，在 Bug 修复之前还是会使用指定的版本。

发现报错了：

[![](https://iocaffcdn.phphub.org/uploads/images/201903/27/3995/vQXZVS6WzH.png!large "安装 DingoAPI")](https://iocaffcdn.phphub.org/uploads/images/201903/27/3995/vQXZVS6WzH.png!large)

dingo 的文档中有说明，现在这个包还处在开发阶段，没有一个稳定的 release 版本，`dingo/api`依赖的`dingo/blueprint`与`phpunit`都依赖了`phpdocumentor/reflection-docblock`但是依赖的版本不同，导致出现了冲突。但是我们发现`dingo/blueprint`的开发版本`dev-master`解决了冲突，可以正常安装，所以我们修改一下`composer.json`：

_composer.json_

```
.
.
.
    "config": {
        "preferred-install": "dist",
        "sort-packages": true,
        "optimize-autoloader": true
    },
    "minimum-stability" : "dev",
    "prefer-stable" : true
}
```

增加了两句：

* `"minimum-stability" : "dev"`
  —— 设定的最低稳定性的版本为
  `dev`
  也就是可以依赖开发版本的扩展包；
* `"prefer-stable" : true`
  —— Composer 优先使用更稳定的包版本。

我们设定项目可以依赖开发版本扩展包，但是当依赖有稳定版本可以安装的时候，优先安装稳定版。

再次执行命令安装：

[![](https://iocaffcdn.phphub.org/uploads/images/201903/27/3995/vUbjT2j8XK.png!large "安装 DingoAPI")](https://iocaffcdn.phphub.org/uploads/images/201903/27/3995/vUbjT2j8XK.png!large)

看到`dingo/api`已经成功安装了。

## 2. 配置

先将 dingo 的配置文件 publish 出来

```
$ php artisan vendor:publish
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/ldSqFfIWco.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/ldSqFfIWco.png)

执行成功后，我们会在`config`目录先看到`api.php`文件，打开文件我们可以看到所有的配置都是可以再 env 中修改的，下面我们主要讲解一下我们需要用到的配置

* API\_STANDARDS\_TREE 和 API\_SUBTYPE

  上一节我们已经讨论了 API 版本的重要性，推荐的做法是使用 Accept 头来指定我们需要访问的 API 版本。`API_STANDARDS_TREE`和`API_SUBTYPE`这两个配置就和版本控制有关

  ```
  Accept: application/<API_STANDARDS_TREE>.<API_SUBTYPE>.v1+json
  ```

  API\_STANDARDS\_TREE 有是三个值可选

  * `x`
    本地开发的或私有环境的
  * `prs`
    未对外发布的，提供给公司 app，单页应用，桌面应用等
  * `vnd`
    对外发布的，开放给所有用户

  对于我们的项目，暂时可以选择`prs`。

  ```
  API_STANDARDS_TREE=prs
  ```

  API\_SUBTYPE 一般情况下是我们项目的简称，我们的项目叫`larabbs`

  ```
  API_SUBTYPE=larabbs
  ```

  所以我们可以通过如下方式来访问不同版本的 API

  ```
  访问 v1 版本
  Accept: application/prs.larabbs.v1+json
  访问 v2 版本
  Accept: application/prs.larabbs.v2+json
  ```

* API\_PREFIX 和 API\_DOMAIN  
  对于一个项目，通过前缀或者子域名的方式来区分开 API 与 Web 等页面访问地址是十分有必要的。假如正式上线的项目地址为`www.larabbs.com`，我们可以为 API 添加一个前缀

  ```
  API_PREFIX=api
  ```

  通过`www.larabbs.com/api`来访问 API。  
  或者有可能单独配置一个子域名`api.larabbs.com`

  ```
  API_DOMAIN=api.larabbs.com
  ```

  通过 api.larabbs.com 来访问 API。

  特别要注意的是：**前缀和子域名，两者有且只有一个**。本教程选择`API_PREFIX`的方式。

* API\_VERSION

  默认的 API 版本，当我们没有传  
  `Accept`  
  头的时候，默认访问该版本的 API。一般情况下配置 v1 即可。

* API\_STRICT

  是否开启严格模式，如果开启，则必须使用  
  `Accept`  
  头才可以访问 API，也就是说直接通过浏览器，访问某个 GET 调用的接口，如  
  `https://api.larabbs.com/users`  
  ，将会报错。必须使用 Postman 之类的调试工具，设置  
  `Accept`  
  后才可访问。可以根据需求开启，默认情况下为 false。

* API\_DEBUG

  测试环境，打开 debug，方便我们看到错误信息，定位错误。

  最后我们的配置如下

  .env

  ```
  .
  .
  .
  API_STANDARDS_TREE=prs
  API_SUBTYPE=larabbs
  API_PREFIX=api
  API_VERSION=v1
  API_DEBUG=true
  ```

注意`.env`文件是不会提交到版本库中的，所以可以将以下代码复制到`.env.example`中，提交到版本库，方便其他环境部署。  
.env.example

```
.
.
.
# dingo config
API_STANDARDS_TREE=
API_SUBTYPE=
API_PREFIX=
API_VERSION=
API_DEBUG=
```

## 3. 版本控制

最后将修改的文件加入到版本控制中。

```
$ git add -A
$ git commit -m "add dingo"
```



