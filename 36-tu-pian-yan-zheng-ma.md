## 图片验证码的作用

为了保证短信验证码接口不会被攻击，我们使用`api.throttle`限制了接口访问频率，但是依旧不安全。我们限制了 IP，但是攻击者依然可以使用大量代理 IP 进行攻击。这个时候，就需要增加一些机器无法识别，或者说识别成本高的人为因素 —— 验证码。

回忆一下『知乎 APP』完整的注册流程，我们可以在发送短信验证码之前，增加一步图片验证码。

## 1. 安装 gregwar/captcha

图片验证码接口的流程是：

* 生成图片验证码
* 生成随机的 key，将验证码文本存入缓存。
* 返回随机的 key，以及验证码图片

Larabbs 项目中已经安装了[mews/captcha](https://github.com/mewebstudio/captcha)，对于网页应用来说，这个组件使用起来十分方便，但是它依赖 session，而且没法获取和设置验证码文本，不适用于 API 的用例。在 API 的开发中，我们将选择使用[gregwar/captcha](https://github.com/Gregwar/Captcha)来完成图片验证码的功能。

```
$ composer require gregwar/captcha
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/06/6351/hwoyos5x48.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/06/6351/hwoyos5x48.png)

## 2. 开发接口

### 1\). 新建路由

接下来，先来增加图片验证码路由：

_routes/api.php_

```
.
.
.
// 用户注册
$api->post('users', 'UsersController@store')
    ->name('api.users.store');
// 图片验证码
$api->post('captchas', 'CaptchasController@store')
    ->name('api.captchas.store');
```

### 2\). 新建控制器和表单验证类

创建`CaptchasController`以及`CaptchaRequest`

```
$ php artisan make:controller Api/CaptchasController
$ php artisan make:request Api/CaptchaRequest
```

修改文件如下

_app/Http/Requests/Api/CaptchaRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class CaptchaRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'phone' => 'required|regex:/^1[34578]\d{9}$/|unique:users',
        ];
    }
}
```

_app/Http/Controllers/Api/CaptchasController.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Gregwar\Captcha\CaptchaBuilder;
use App\Http\Requests\Api\CaptchaRequest;

class CaptchasController extends Controller
{
    public function store(CaptchaRequest $request, CaptchaBuilder $captchaBuilder)
    {
        $key = 'captcha-'.str_random(15);
        $phone = $request->phone;

        $captcha = $captchaBuilder->build();
        $expiredAt = now()->addMinutes(2);
        \Cache::put($key, ['phone' => $phone, 'code' => $captcha->getPhrase()], $expiredAt);

        $result = [
            'captcha_key' => $key,
            'expired_at' => $expiredAt->toDateTimeString(),
            'captcha_image_content' => $captcha->inline()
        ];

        return $this->response->array($result)->setStatusCode(201);
    }
}
```

### 3\). 代码分解

分析一下代码：

* 增加了
  `CaptchaRequest`
  要求用户必须通过手机号调用
  `图片验证码`
  接口。
* controller 中，注入
  `CaptchaBuilder`
  ，通过它的 build 方法，创建出来验证码图片
* 使用
  `getPhrase`
  方法获取验证码文本，跟手机号一同存入缓存。
* 返回 captcha\_key，过期时间以及
  `inline`
  方法获取的 base64 图片验证码

这里给图片验证码设置为 2 分钟过期，并且考虑到图片验证码比较小，直接以 base64 格式返回图片，大家可以考虑在这里返回图片 url，例如`http://larabbs.test/captchas/{captcha_key}`，然后访问该链接的时候生成并返回图片。

### 4\). 测试一下

调试一下接口，可以先删除数据库中前几节测试创建的用户，因为手机号是`unique`的。删除成功后再次发送请求：  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/3Py2OLPsmY.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/3Py2OLPsmY.png)  
请求成功，复制`captcha_image_content`的值，到浏览器中打开  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/IE6h4AGnPm.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/IE6h4AGnPm.png)

可以看到我们生成的验证码，我们先保存下接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/Wbq5NjBkfm.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/Wbq5NjBkfm.png)

### 5\). 集成到短信验证码接口里

接下来需要修改一下原来的`发送短信验证码接口`，通过`captcha_key`和`captcha_code`请求该接口，修改如下：

_app/Http/Requests/Api/VerificationCodeRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class VerificationCodeRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'captcha_key' => 'required|string',
            'captcha_code' => 'required|string',
        ];
    }

    public function attributes()
    {
        return [
            'captcha_key' => '图片验证码 key',
            'captcha_code' => '图片验证码',
        ];
    }
}
```

_app/Http/Controllers/Api/VerificationCodesController.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Overtrue\EasySms\EasySms;
use App\Http\Requests\Api\VerificationCodeRequest;

class VerificationCodesController extends Controller
{
    public function store(VerificationCodeRequest $request, EasySms $easySms)
    {
        $captchaData = \Cache::get($request->captcha_key);

        if (!$captchaData) {
            return $this->response->error('图片验证码已失效', 422);
        }

        if (!hash_equals($captchaData['code'], $request->captcha_code)) {
            // 验证错误就清除缓存
            \Cache::forget($request->captcha_key);
            return $this->response->errorUnauthorized('验证码错误');
        }

        $phone = $captchaData['phone'];

        if (!app()->environment('production')) {
            $code = '1234';
        } else {
            // 生成4位随机数，左侧补0
            $code = str_pad(random_int(1, 9999), 4, 0, STR_PAD_LEFT);

            try {
                $result = $easySms->send($phone, [
                    'content'  =>  "【Lbbs社区】您的验证码是{$code}。如非本人操作，请忽略本短信"
                ]);
            } catch (\Overtrue\EasySms\Exceptions\NoGatewayAvailableException $exception) {
                $message = $exception->getException('yunpian')->getMessage();
                return $this->response->errorInternal($message ?? '短信发送异常');
            }
        }

        $key = 'verificationCode_'.str_random(15);
        $expiredAt = now()->addMinutes(10);
        // 缓存验证码 10分钟过期。
        \Cache::put($key, ['phone' => $phone, 'code' => $code], $expiredAt);
        // 清除图片验证码缓存
        \Cache::forget($request->captcha_key);

        return $this->response->array([
            'key' => $key,
            'expired_at' => $expiredAt->toDateTimeString(),
        ])->setStatusCode(201);
    }
}
```

主要增加了验证图片验证码的步骤，用户手机号从缓存中获取，最后清除图片验证码缓存。

修改`发送验证码接口`参数，测试一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/ZpRM8bbfMV.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/08/6351/ZpRM8bbfMV.png)

最后调用用户注册接口，可以创建用户了。用户注册接口没有任何修改，这里就不截图了。记得保存修改后的接口。

## 代码版本控制

最后我们提交修改的代码

```
$ git add -A
$ git commit -m "add captcha"
```



