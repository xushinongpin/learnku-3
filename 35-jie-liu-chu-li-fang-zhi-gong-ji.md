## 1. 接口安全

验证码轰炸，是最常见的攻击方式，恶意调用`发送短信验证码`接口，给一个用户手机或多个手机号码频繁发送验证码短信，对其造成非常负面的影响，如果接口被轰炸，会导致我们的短信供应商那里的账户余额很快被耗尽。而手机注册的接口，通过不断尝试短信验证码，很容易在短信验证码未过期前，就被破解出来。

接口安全很重要，我们需要有合理的节流机制，来防止以上提到的攻击。节流机制，说到底就是对调用频率的限制，限制每个 ip 的调用次数。

## 2. 增加调用频率限制

DingoApi 已经为我们提供了调用频率限制的中间件`api.throttle`，使用起来非常方便，修改路由

_routes/api.php_

```
.
.
.
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
], function($api) {

    $api->group([
        'middleware' => 'api.throttle',
        'limit' => 1, 
        'expires' => 1,
    ], function($api) {
        // 短信验证码
        $api->post('verificationCodes', 'VerificationCodesController@store')
            ->name('api.verificationCodes.store');
        // 用户注册
        $api->post('users', 'UsersController@store')
            ->name('api.users.store');
    });
});
```

我们暂时设置为 1 分钟 1 次，方便测试。使用 PostMan 请求用户注册接口，输入错误的验证码。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/exxlF6ED2h.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/exxlF6ED2h.png)

第一次请求，返回 401，验证码错误，进行第二次请求

[![](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/WnvKXQ1fUX.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/WnvKXQ1fUX.png)  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/ooIBmXTVNm.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/17/6351/ooIBmXTVNm.png)

看到结果返回`429 Too Many Requests`，查看`Headers`其中有`X_RateLimit`相关的头信息。客户端判断状态码为 429 返回`操作频率过快，请稍后再试`等提示即可。

> `X_Rate`相关的头信息，是通过中间件添加的，当我们抛出异常时，并不能正确的添加，所以当我们第一次请求，验证码错误时，并没有头信息返回。这是中间件以及异常的处理机制问题，我会在找到解决办法后更新在教程中。但是该问题并不影响正常使用。

## 3. 增加配置

写成配置，可以更方便的控制调用频率：

_config/api.php_

```
.
.
.
    /*
     * 接口频率限制
     */
    'rate_limits' => [
        // 访问频率限制，次数/分钟
        'access' => [
            'expires' => env('RATE_LIMITS_EXPIRES', 1),
            'limit'  => env('RATE_LIMITS', 60),
        ],
        // 登录相关，次数/分钟
        'sign' => [
            'expires' => env('SIGN_RATE_LIMITS_EXPIRES', 1),
            'limit'  => env('SIGN_RATE_LIMITS', 10),
        ],
    ],
];
```

我们增加了两种限制，一种登录相关的，一分钟可以调用 10 次，一种是访问相关的，一分钟调用 60 次。然后调整路由代码：

_routes/api.php_

```
 $api->group([
        'middleware' => 'api.throttle',
        'limit' => config('api.rate_limits.sign.limit'),
        'expires' => config('api.rate_limits.sign.expires'),
    ], function($api) {
        // 短信验证码
        $api->post('verificationCodes', 'VerificationCodesController@store')
            ->name('api.verificationCodes.store');
        // 用户注册
        $api->post('users', 'UsersController@store')
            ->name('api.users.store');
    });
```

> 在开发中，合理地抽象『程序配置信息』是一个合格的工程师的必备技能之一。

## 提交修改的代码

```
$ git add -A
$ git commit -m "rate limit"
```



