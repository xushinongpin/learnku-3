## 1. 什么是 PostMan？

PostMan 是一款跨平台，方便我们调试 API 的工具，你可以在[PostMan 官网](https://www.getpostman.com/)下载，或者使用[百度网盘下载](https://pan.baidu.com/s/1dEMMD2L)。

[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/2GQLI18FGu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/2GQLI18FGu.png)  
打开 PostMan 界面如上图所示，大体可以分为四个区域，左侧`接口集合`类似文件夹的功能，我们可以把我们的接口保存在这里，右侧上中下分别是`请求地址`，`请求参数`和`请求结果`。

随便找个接口调用一下，这个是国家气象局提供的[天气预报接口](http://www.weather.com.cn/data/sk/101010100.html)，将其填入『请求地址』处：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/24/1/fw3oUqkvyP.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/24/1/fw3oUqkvyP.png)

可以在左侧的区域，保存接口，我们新建一个`Larabbs`目录。

[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/lTodfiiaqQ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/lTodfiiaqQ.png)  
[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/oWg7CJjtrz.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/oWg7CJjtrz.png)

在左侧我们看到了新建的文件夹 Larabbs，然后保存接口。  
[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/cjYL1IsJNG.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/cjYL1IsJNG.png)  
可以给接口起个名字，填写响应的描述。  
[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/7QL7E75B2u.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/7QL7E75B2u.png)

[![](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/xuw9pIhHaf.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/29/6351/xuw9pIhHaf.png)

我们可以将已经调试好的接口保存下来，方便下次调试，PostMan 也为我们提供了导出导入接口的功能，方便分享接口给他人，当然 PostMan 也为付费用户提供了更多方便的功能，大家有需要的可以取官网了解，目前免费版的功能已经满足我们的需求。

## 2. 编写调试接口

已经安装好了 DingoApi，接下来写两个路由，尝试一下 PostMan。Laravel 5.5 已经为我们准备好了 api 的路由文件，`routes/api.php`。打开这个文件，可以看到 Laravel 为我们写好的例子。

```
<?php

use Illuminate\Http\Request;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/

Route::middleware('auth:api')->get('/user', function (Request $request) {
    return $request->user();
});
```

由于我们使用的是 DingoApi 的路由，所以将文件替换为如下内容

```
<?php

use Illuminate\Http\Request;

/*
|--------------------------------------------------------------------------
| API Routes
|--------------------------------------------------------------------------
|
| Here is where you can register API routes for your application. These
| routes are loaded by the RouteServiceProvider within a group which
| is assigned the "api" middleware group. Enjoy building your API!
|
*/

$api = app('Dingo\Api\Routing\Router');

$api->version('v1', function($api) {
    $api->get('version', function() {
        return response('this is version v1');
    });
});

$api->version('v2', function($api) {
    $api->get('version', function() {
        return response('this is version v2');
    });
});
```

路由需要使用`Dingo\Api\Routing\Router`注册，写法同 Laravel 的路由，大家应该比较熟悉。DingoApi 提供了 version 方法，用来进行版本控制，第一个参数是版本名称，version 中的就是不用版本的路由。我们在 v1 和 v2 两个版本中，都注册了 verison 路由，但是响应不同，现在通过 PostMan 来试试。  
还记得我们配置的是`API_PREFIX=api`，所以需要`http://larabbs.test/api`来访问。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/TDGI068yFV.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/TDGI068yFV.png)

我们得到了 v1 版本的结果，`this is version v1`，因为配置了默认的版本是 v1`API_VERSION=v1`。但是下面却多出来了一些 html，这其实是 Larabbs 安装的`laravel-debugbar`，做接口相关的开发，不涉及网页，所以我们现在来关闭它。  
打开 config/debugbar.php，可以看到一行配置`enabled`。

```
'enabled' => env('APP_DEBUG', false),
```

该配置控制`debugbar`的开启和关闭，但是它现在是随着`APP_DEBUG`的改变而改变的，我们可以把代码修改为

```
'enabled' => env('DEBUGBAR_ENABLE', false),
```

这样我们本地测试环境`APP_DEBUG`依然可以是 true，同时也可以关闭`laravel-debugbar`。默认`laravel-debugbar`是关闭的，我们再次通过 PostMan 访问该接口。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/w8Scu1mA4g.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/w8Scu1mA4g.png)  
一切正常了，记得当我们调试网页，需要用到`laravel-debugbar`的时候，在 .env 中增加`DEBUGBAR_ENABLE=true`即可。

下面我们增加`Accept: application/prs.larabbs.v2+json`来访问 v2 版本的 version。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/UIeJWYMiou.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/UIeJWYMiou.png)

> 提示：由于路由被 DingoApi 接管了，如果将来部署上线后你需要缓存路由，可以使用`php artisan api:cache`代替`php artisan route:cache`，本地测试请**不要执行这个命令**。

## 3. PostMan 的环境变量功能

PostMan 为我们提供了环境变量的功能，通过切换不同的值，可以使用不同的环境，比如 host 我们就可以做成变量，这样当我们某一天切换了本地调试的域名，比如由 larabbs.app 切换为 larabbs.test 时，不用去每个接口中修改，只需要修改变量即可。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/02/6351/SinuoV0uVD.gif "file")](https://iocaffcdn.phphub.org/uploads/images/201801/02/6351/SinuoV0uVD.gif)  
比如上面，我们增加了一个环境变量`host`，然后我们选择对应的环境，将域名替换为`{{host}}`，PostMan url 中，使用双括号表示变量。同样能访问得到正确的结果。

## 4. 代码版本控制

最后，我们将测试的路由代码恢复：

```
$ git checkout routes/api.php
```

然后将`config/debugbar.php`提交：

```
$ git add -A
$ git commit -m "fix laravel-debugbar"
```



