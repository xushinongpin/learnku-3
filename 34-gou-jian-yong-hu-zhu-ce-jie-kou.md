## 1. 新增路由

添加用户注册路由

_routes/api.php_

```
.
.
.
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api'
], function($api) {
    // 短信验证码
    $api->post('verificationCodes', 'VerificationCodesController@store')
        ->name('api.verificationCodes.store');
    // 用户注册
    $api->post('users', 'UsersController@store')
        ->name('api.users.store');
});
```

## 2. 控制器和表单验证类

创建用户`controller`及`request`

```
$ php artisan make:controller Api/UsersController
$ php artisan make:request Api/UserRequest
```

修改文件如下：

_app/Http/Requests/Api/UserRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class UserRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'name' => 'required|between:3,25|regex:/^[A-Za-z0-9\-\_]+$/|unique:users,name',
            'password' => 'required|string|min:6',
            'verification_key' => 'required|string',
            'verification_code' => 'required|string',
        ];
    }

    public function attributes()
    {
        return [
            'verification_key' => '短信验证码 key',
            'verification_code' => '短信验证码',
        ];
    }
}
```

_app/Http/Controllers/Api/UsersController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\User;
use Illuminate\Http\Request;
use App\Http\Requests\Api\UserRequest;

class UsersController extends Controller
{
    public function store(UserRequest $request)
    {
        $verifyData = \Cache::get($request->verification_key);

        if (!$verifyData) {
            return $this->response->error('验证码已失效', 422);
        }

        if (!hash_equals($verifyData['code'], $request->verification_code)) {
            // 返回401
            return $this->response->errorUnauthorized('验证码错误');
        }

        $user = User::create([
            'name' => $request->name,
            'phone' => $verifyData['phone'],
            'password' => bcrypt($request->password),
        ]);

        // 清除验证码缓存
        \Cache::forget($request->verification_key);

        return $this->response->created();
    }
}
```

## 3. 代码详解

这里我们比对验证码是否与缓存中一致时，使用了[hash\_equals](http://php.net/manual/zh/function.hash-equals.php)方法。

```
hash_equals($verifyData['code'], $request->verification_code)
```

hash\_equals 是可防止时序攻击的字符串比较，那么什么是时序攻击呢？比如这段代码我们使用

```
$verifyData['code'] == $request->verification_code
```

进行比较，那么两个字符串是从第一位开始逐一进行比较的，发现不同就立即返回 false，那么通过计算返回的速度就知道了大概是哪一位开始不同的，这样就实现了电影中经常出现的按位破解密码的场景。而使用`hash_equals`比较两个字符串，无论字符串是否相等，函数的时间消耗是恒定的，这样可以有效的防止时序攻击。

验证码过期或者`verification_key`错误时，我们使用`$this->response->error`返回错误信息，状态码为 422，表明提交的参数错误。而验证码错误的情况，我们使用`errorUnauthorized`返回，状态码为 401，这里主要考虑到 401 的解释：

> 客户端在没有提供适当的身份认证凭证的时候向受保护的资源发送请求。他可能提供了错误的凭证或完全没有提供凭证。凭证可以是用户名和密码、一个 API Key 或者一个认证的 token-- 任何 API 质询时所期望的内容。

现在用户需要用正确的验证码来创建用户，验证码可以作为用户的身份凭证，401 是合适的状态码。

> 对状态码不熟悉的同学，请重温下[正确使用状态码](https://learnku.com/courses/laravel-advance-training/5.5/787/follow-github-to-learn-restful-http-api-design#7-%E6%AD%A3%E7%A1%AE%E4%BD%BF%E7%94%A8%E7%8A%B6%E6%80%81%E7%A0%81)一文。

注册成功后，我们暂时通过 DingoApi 提供的 created 方法返回，状态码为 201，之后的课程我们会修改为返回用户数据。

## 4. PostMan 测试一下

下面使用 PostMan 调试一下用户注册流程

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/yg10gtFtVc.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/yg10gtFtVc.png)  
我们得到的 key 为`verificationCode_CN48D8ctcuYoVh1`，带着 key 请求用户注册接口。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/s1doTDj9f5.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/s1doTDj9f5.png)  
注册成功了，但是我们打开数据库发现，用户的手机没有写入数据库  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/TaXt73R8sQ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/TaXt73R8sQ.png)  
这是因为 user 模型的 fillable 未设置 phone，修改如下

_app/Models/User.php_

```
protected $fillable = [
    'name', 'phone', 'email', 'password', 'introduction', 'avatar',
];
```

删除刚才插入的数据，再次尝试调用这两个接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/z3aEh7dRND.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/z3aEh7dRND.png)

数据写入正确，保存用户注册接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/HPPRr50bVH.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/HPPRr50bVH.png)

## 5. 代码版本控制

最后，提交我们的代码

```
$ git add -A
$ git commit -m "user can register using phone"
```



