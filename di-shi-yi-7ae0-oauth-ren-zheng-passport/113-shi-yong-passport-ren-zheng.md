## 使用 Passport 认证

这一节我们将现有的接口实现，由之前 JWT 授权方式， 更替为 Passport 的 Oauth2 授权。

## 登录接口

### 调整接口

Passport 提供的默认路由为[http://larabbs.test/oauth/token](http://larabbs.test/oauth/token)，而我们的现在接口统一都有`/api`的前缀，所以我们不使用 Passport 默认的路由，依然使用`/api/authorizations`。先来修改登录接口：

_app/Http/Controllers/Api/AuthorizationsController.php_

```
.
.
.
use Zend\Diactoros\Response as Psr7Response;
use Psr\Http\Message\ServerRequestInterface;
use League\OAuth2\Server\Exception\OAuthServerException;
use League\OAuth2\Server\AuthorizationServer;
.
.
.
public function store(AuthorizationRequest $originRequest, AuthorizationServer $server, ServerRequestInterface $serverRequest)
{
    try {
       return $server->respondToAccessTokenRequest($serverRequest, new Psr7Response)->withStatus(201);
    } catch(OAuthServerException $e) {
        return $this->response->errorUnauthorized($e->getMessage());
    }
}
.
.
.
```

逻辑很简单，我们注入了`AuthorizationServer`和`ServerRequestInterface`，调用`AuthorizationServer`的`respondToAccessTokenRequest`方法并直接返回。

`respondToAccessTokenRequest`会依次处理：

* 检测 client 参数是否正确；
* 检测 scope 参数是否正确；
* 通过用户名查找用户；
* 验证用户密码是否正确；
* 生成 Response 并返回；

最终返回的 Response 是 Zend\Diactoros\Respnose 的实例，代码位置在`vendor/zendframework/zend-diactoros/src/Response.php`，查看代码我们可以使用`withStatus`方法设置该 Response 的状态码，最后直接返回 Response 即可。

### 使用 PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/OHu3pLU7UR.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/OHu3pLU7UR.png)

得到了正确的令牌信息，注意现在使用的用户是邮箱，我们是支持手机和邮箱两种登录方式的，现在尝试一下使用手机登录：

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/A0DvWlpQ0s.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/A0DvWlpQ0s.png)

最终结果报错了，因为默认情况下，Passport 会通过用户的邮箱查找用户，要支持手机登录，我们可以在用户模型定义了`findForPassport`方法，Passport 会先检测用户模型是否存在`findForPassport`方法，如果存在就通过`findForPassport`查找用户，而不是使用默认的邮箱。

### 支持手机登录

_app/Models/User.php_

```
.
.
.
public function findForPassport($username)
{
    filter_var($username, FILTER_VALIDATE_EMAIL) ?
      $credentials['email'] = $username :
      $credentials['phone'] = $username;

    return self::where($credentials)->first();
}
.
.
.
```

定义好`findForPassport`后，再次尝试使用手机登录：

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/PF7ujRVII6.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/PF7ujRVII6.png)

登录成功。

## 刷新 Token

_app/Http/Controllers/Api/AuthorizationsController.php_

```
public function update(AuthorizationServer $server, ServerRequestInterface $serverRequest)
{
    try {
       return $server->respondToAccessTokenRequest($serverRequest, new Psr7Response);
    } catch(OAuthServerException $e) {
        return $this->response->errorUnauthorized($e->getMessage());
    }
}
```

刷新接口的代码同登录接口一致，只是最终返回的状态码是 200。

使用 PostMan 调试：

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/2MmEnM1FNh.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/2MmEnM1FNh.png)

注意 PUT 提交参数需要使用`x-www-form-urlencode`，刷新 Token 成功。

## 获取登录用户信息

得到了访问令牌，我们就能通过令牌获取个人用户信息了，不过在这之前我们需要修改一些配置。

将`Laravel\Passport\HasApiTokens`Trait 添加到`App\Models\User`模型中，这个 Trait 会给你的模型提供一些辅助函数，用于检查已认证用户的令牌和使用范围。

_app/Models/User.php_

```
.
.
.
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable implements JWTSubject
{
    use HasApiTokens;
.
.
.
```

修改`auth`配置，将 api guard 的`driver`由`jwt`修改为`passport`。

_config/auth.php_

```
.
.
.
'guards' => [
.
.
.
    'api' => [
        'driver' => 'passport',
        'provider' => 'users',
    ],
],
.
.
.
```

增加了一个 PassportDingoProvider，因为 DingoApi 没有做 Passport 的适配，所以需要手动处理一下。

```
$ php artisan make:provider PassportDingoProvider
```

_app/Providers/PassportDingoProvider.php_

```
<?php
namespace App\Providers;

use Dingo\Api\Routing\Route;
use Illuminate\Http\Request;
use Illuminate\Auth\AuthManager;
use Dingo\Api\Auth\Provider\Authorization;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

class PassportDingoProvider extends Authorization
{
    protected $auth;

    protected $guard = 'api';

    public function __construct(AuthManager $auth)
    {
        $this->auth = $auth->guard($this->guard);
    }

    public function authenticate(Request $request, Route $route)
    {
        if (! $user = $this->auth->user()) {
            throw new UnauthorizedHttpException(
                get_class($this),
                'Unable to authenticate with invalid API key and token.'
            );
        }

        return $user;
    }

    public function getAuthorizationMethod()
    {
        return 'Bearer';
    }
}
```

将原来的`jwt`替换为`oauth`，值为刚才创建的 Provider，这样在 Controller 中我们使用`$this->user()`就能获取到令牌对应的用户模型了。

_config/api.php_

```
.
.
.
'auth' => [
    //'jwt' => 'Dingo\Api\Auth\Provider\JWT',
    'oauth' => \App\Providers\PassportDingoProvider::class,
],
.
.
```

使用 PostMan 访问`获取登录用户信息`接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/wIoDvcgRhV.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/wIoDvcgRhV.png)

设置好`access_token`, 可正确获取到用户信息。

## 删除 Token

_app/Http/Controllers/Api/AuthorizationsController.php_

```
.
.
.
public function destroy()
{
        if (!empty($this->user())) {
            $this->user()->token()->revoke();
            return $this->response->noContent();
        } else {
            return $this->response->errorUnauthorized('The token is invalid.');
        }
}
.
.
.
```

使用 PostMan 调试：  
[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/zZJ3RDiJNB.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/zZJ3RDiJNB.png)

使用已经删除的`access_token`再次访问`获取登录用户信息`接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/lzI20ADkXq.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/lzI20ADkXq.png)

结果为 401 授权错误，证明`access_token`已经失效。

## 代码版本控制

```
$ git add -A
$ git commit -m 'change jwt to passport'
```



