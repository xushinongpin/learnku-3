## 本地化

这一节我们来实现接口的本地化。本地化主要的是客户端的工作，切换语言后，客户端显示不同的界面，例如下面就是微信**中文**和**英文**语言下的界面。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/13/3995/5PfbxeCVi0.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/13/3995/5PfbxeCVi0.png)

除了界面显示之外，还有一些报错信息需要做本地化，举个例子，用户登录时，密码错误：

* 英文客户端，提示
  `invalid username or password`
* 中文客户端，提示
  `用户名和密码错误`

报错信息本地化的处理方式，一般有两种：

* 客户端通过服务器端返回的状态码和错误码，自行翻译为错误信息；
* 服务器端返回状态码时，返回已经格式化了的错误消息。

接下来我们会一一讲解。

## 1. 本地化完全交给客户端

因为我们是 RESTFul 风格的接口，返回了标准的状态码，大部分情况下，客户端可以根据状态码，以及语言设置提示给用户不同语言的报错信息。例如上面的例子，客户端调用`登录`接口时，报错信息中的 message 统一为中文，客户端根据状态码等标识进行本地化提示。

```
{
    "message": "用户名和密码错误",
    "status_code": 401
}
```

但是某些情况下，只有状态码是不够的，比如下面这个场景`发布话题`，我们增加了以下限制：

| 错误原因 | 状态码 | 错误描述 |
| :--- | :--- | :--- |
| 被加入黑名单的用户不能发帖 | 403 | 您已被加入黑名单 |
| 会员用户才能发帖 | 403 | 您还不是会员 |
| 实名认证的用户才能发帖 | 403 | 您还没有通过认证 |

状态码都是 403 但是报错信息却不同，这时就需要定义不同的错误码，以便让客户端进行判断：

| 错误原因 | 状态码 | 错误描述 | 自定义错误码 |
| :--- | :--- | :--- | :--- |
| 被加入黑名单的用户不能发帖 | 403 | 您已被加入黑名单 | 1001 |
| 会员用户才能发帖 | 403 | 您还不是会员 | 1002 |
| 实名认证的用户才能发帖 | 403 | 您还没有通过认证 | 1003 |

接口响应类似下面这样：

```
{
    "message": "您还没有通过认证",
    "status_code": 403,
    "code": 1003
}
```

### 错误响应中增加 code 错误码字段

我们需要提前定义好『错误码』，并且在响应中增加 code 字段，这样客户端才能错误码，提示出正确的报错信息。DingoApi 会自动将异常中的异常码 （code）添加到响应中，所以我们只需要抛出异常的时候增加自定义的异常码即可，例如：

```
throw new \Symfony\Component\HttpKernel\Exception\HttpException(
                                401, 
                                '您还没有通过认证', 
                                null, 
                                [], 
                                1003);
```

### 进行封装

我们就会得到上面的响应结果。当然我们应该对上述代码进行封装：

_app/Http/Controllers/Api/Controller.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Dingo\Api\Routing\Helpers;
use App\Http\Controllers\Controller as BaseController;
use Symfony\Component\HttpKernel\Exception\HttpException;

class Controller extends BaseController
{
    use Helpers;

    public function errorResponse($statusCode, $message=null, $code=0)
    {
        throw new HttpException($statusCode, $message, null, [], $code);
    }
}
```

我们在`Api/Controller`中加了`errorResponse`方法，所以我们在任意 API 控制器中直接使用`$this->errorResponse`即可。

### 添加测试代码

比如我们在`发布话题`接口的代码中增加下面的测试代码：

_app/Http/Controllers/Api/TopicsController.php_

```
.
.
.
public function store(TopicRequest $request, Topic $topic)
{
    return $this->errorResponse(403, '您还没有通过认证', 1003);
.
.
.
```

由于是我们假设的业务，所以直接抛出异常，观察响应。

### 使用 PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/OvVJeBJyaH.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/OvVJeBJyaH.png)

可以看到响应中增加了 code 字段。记得还原测试代码：

```
$ git checkout app/Http/Controllers/Api/TopicsController.php
```

## 2. 接口根据客户端语言切换错误信息

如果接口根据客户端语言设置，返回对应语言的报错信息，客户端就可以直接将服务器报错提示给用户。由于每个接口都是无状态的，客户端需要在每次请求接口的时候增加参数，告诉接口支持的语言。我们可以利用 HTTP 的`Accept-Language`头信息。

* Accept-Language zh-CN —— 简体中文
* Accept-Language en —— 英文

### 增加 middleware

通过以下命令创建一个中间件。

```
$ php artisan make:middleware ChangeLocale
```

打开文件，填入如下代码：

_app/Http/Middleware/ChangeLocale.php_

```
<?php

namespace App\Http\Middleware;

use Closure;

class ChangeLocale
{
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        $language = $request->header('accept-language');
        if ($language) {
            \App::setLocale($language);
        }

        return $next($request);
    }
}
```

逻辑很简单，获取请求头中的`accept-language`，然后设置语言。

### 注册中间件

_app/Http/Kernel.php_

```
protected $routeMiddleware = [
    .
    .
    .
    // 访问节流，类似于 『1 分钟只能请求 10 次』的需求，一般在 API 中使用
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,

    // 接口语言设置
    'change-locale' => \App\Http\Middleware\ChangeLocale::class,
    .
    .
    .
];
```

_routes/api.php_

```
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => ['serializer:array', 'bindings', 'change-locale']
], function ($api) {
```

我们注册了`change-locale`这个中间件，并且在 v1 版本的接口中增加该中间件。

### PostMan 调试

接下来我们以`登录`接口为例。

由于我们已经在 config/app.php 中设置了默认的语言为`'locale' => 'zh-CN',`简体中文，所以我们直接访问登录接口，密码只填写 3 位。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/pEGBbSGwV0.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/pEGBbSGwV0.png)

可以看到中文的报错信息：

增加`accept-language`头，值为`en`切换为英文。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/aBDe7eItUX.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/aBDe7eItUX.png)

可以切换语言，并且得到了正确的报错信息，看一下登录的代码：

_app/Http/Controllers/Api/AuthorizationsController.php_

```
.
.
.
if (!$token = \Auth::guard('api')->attempt($credentials)) {
    return $this->response->errorUnauthorized('用户名或密码错误');
}
.
.
.
```

发现用户名密码错误的时候，报错并没有进行本地化处理，修改代码如下：

```
.
.
.
if (!$token = \Auth::guard('api')->attempt($credentials)) {
    return $this->response->errorUnauthorized(trans('auth.failed'));
}
.
.
.
```

输入错误的密码，访问登录接口来测试。

使用默认的中文：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/Uz4ydTlTcs.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/Uz4ydTlTcs.png)

设置为英文：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/JkNJdJRuTR.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/JkNJdJRuTR.png)

## 方案总结

第一种方案比较专业，大部分的第三方 API 平台接口提供方都会提供状态码，如[新浪微博](http://open.weibo.com/wiki/Error_code)错误码，但是你需要一个个地将错误码写出。在某些开发需求中，错误码可能不仅影响本地化，有时客户端需要服务器返回的特定状态码做不同的业务处理，比如跳转到不同的页面。

第二种方案比较便捷，客户端可以直接将服务器的报错消息显示给用户，从快速实现的角度来看，省去了错误码的定义，并且后期可通过服务器代码来控制错误消息内容，也带来一定的灵活性。

从接口的可扩展性、适用性和专业性上考虑，最合理的也是最推荐的做法是将两种方案合并 —— API 既提供错误码，又提供错误消息。

另外，错误消息**默认**返回英文，也会是比较合理的最佳实践，当然最终视 API 业务逻辑而定。

## 代码版本控制

```
$ git add -A
$ git commit -m 'locale'
```



