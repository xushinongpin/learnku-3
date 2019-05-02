## 1. 修改数据结构

接下来我们要准备开始手机注册功能的开发，开始之前我们需要对 LaraBBS 做一些修改。

现在的 Larabbs 是通过邮箱注册的，用户表中还没有手机字段，所以我们首先需要在`users`表中增加`phone`字段。因为是手机注册，还需要修改`email`字段为`nullable`。

```
$ php artisan make:migration add_phone_to_users_table --table=users
```

修改文件如以下，注意文件名中的变量：

_database/migrations/`{your_date}`\_add\_phone\_to\_users\_table.php_

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class AddPhoneToUsersTable extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->string('phone')->nullable()->unique()->after('name');
            $table->string('email')->nullable()->change();
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::table('users', function (Blueprint $table) {
            $table->dropColumn('phone');
            $table->string('email')->nullable(false)->change();
        });
    }
}
```

执行 migrate

```
$ php artisan migrate
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/gEwJzqinAj.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/gEwJzqinAj.png)

发现报错了，因为我们修改数据表字段的属性，这个功能需要`doctrine/dbal`组件，我们先安装它：

```
$ composer require doctrine/dbal
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/UU35AzIXcV.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/UU35AzIXcV.png)

再次执行 migrate，执行成功  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/QKhpGN65VL.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/QKhpGN65VL.png)

## 2. 构建短信验证码接口

我们先将注册流程简化一下，将发送图片验证码的流程去掉，流程如下：

[![](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/Hpetk4ofnu.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/Hpetk4ofnu.png!large)

### 1\). 新建基类

首先来搭建一下基础环境，创建一个基础 Controller，此类作为所有 API 请求控制器的『基类』。

```
$ php artisan make:controller Api/Controller
```

注意我们增加了一个命名空间`Api`，以后接口相关的控制器，统一会放在`Api`目录中，会让代码结构更清晰。前面提到过接口版本控制的重要性，我们还可以在`Api`目录中增加 V1，V2 等目录，进一步区分不同版本的接口，为了教学方便，我们暂时不做进一步区分。 将 Controller.php 文件替换为以下的内容。

_app/Http/Controllers/Api/Controller.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use Dingo\Api\Routing\Helpers;
use App\Http\Controllers\Controller as BaseController;

class Controller extends BaseController
{
    use Helpers;
}
```

注意上面的基础代码中，我们增加了 DingoApi 的 helper，这个 trait 可以帮助我们处理接口响应，后面我们在使用到的时候，我们会对其提供的功能做一一讲解。我们新建的 Api 控制器都会继承这个基础控制器。

### 2\). 新增路由

下面我们开始写第一个接口`发送短信验证码`，先添加路由

_routes/api.php_

```
<?php

use Illuminate\Http\Request;

$api = app('Dingo\Api\Routing\Router');

$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api'
], function($api) {
    // 短信验证码
    $api->post('verificationCodes', 'VerificationCodesController@store')
        ->name('api.verificationCodes.store');
});
```

上面的代码中，我们增加了一个参数`namespace`，使 v1 版本的路由都会指向`App\Http\Controllers\Api`，方便我们书写路由。

### 3\). 构建控制器

接下来创建`VerificationCodes`的控制器。

```
$ php artisan make:controller Api/VerificationCodesController
```

修改文件如以下：

_app/Http/Controllers/Api/VerificationCodesController.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;

class VerificationCodesController extends Controller
{
    public function store()
    {
        return $this->response->array(['test_message' => 'store verification code']);
    }
}
```

> 通过 artisan 创建出来的控制器，默认会继承 App\Http\Controllers\Controller，我们只需要删除`use App\Http\Controllers\Controller`这一行即可，这样会继承相同命名空间下的 Controller，也就是我们上一步中添加的那个控制器。

### 4\). PostMan 里测试一下

增加了 store 方法，利用 DingoApi 的 Helpers trait，我们可以使用`$this->response->array`返回一个测试用的响应。接下来使用 PostMan 访问`发送短信验证码`接口，注意使用`POST`方式提交。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/tEzSCGbC6v.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/01/6351/tEzSCGbC6v.png)

访问正常。

### 5\). 创建 API 表单请求验证类

我们通过手机号请求接口，获得短信验证码。每当我们接收用户提交的参数时，我们都需要对数据做验证，以保证数据的准确性，接下来我们创建属于 API 的表单请求验证类：

```
$ php artisan make:request Api/VerificationCodeRequest
```

同样增加了命名空间`Api`，用于区分 API 与 Web 的 Request 文件。修改文件：

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
            'phone' => [
                'required',
                'regex:/^((13[0-9])|(14[5,7])|(15[0-3,5-9])|(17[0,3,5-8])|(18[0-9])|166|198|199)\d{8}$/',
                'unique:users'
            ]
        ];
    }
}

```

注意这里的 FormRequest 是 DingoApi 为我们提供的基类。必须提交`phone`参数，必须是一个合法的电话格式，而且该手机号未注册过。

### 6\). 继续编写控制器逻辑

再次编辑`VerificationCodesController`：

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
        $phone = $request->phone;

        // 生成4位随机数，左侧补0
        $code = str_pad(random_int(1, 9999), 4, 0, STR_PAD_LEFT);

        try {
            $result = $easySms->send($phone, [
                'content'  =>  "【Lbbs社区】您的验证码是{$code}。如非本人操作，请忽略本短信"
            ]);
        } catch (\Overtrue\EasySms\Exceptions\NoGatewayAvailableException $exception) {
            $message = $exception->getException('yunpian')->getMessage();
            return $this->response->errorInternal($message ?: '短信发送异常');
        }

        $key = 'verificationCode_'.str_random(15);
        $expiredAt = now()->addMinutes(10);
        // 缓存验证码 10分钟过期。
        \Cache::put($key, ['phone' => $phone, 'code' => $code], $expiredAt);

        return $this->response->array([
            'key' => $key,
            'expired_at' => $expiredAt->toDateTimeString(),
        ])->setStatusCode(201);
    }
}
```

思路是：

* 生成 4 位随机码
* 用 easySms 发送短信到用户手机
* 发送成功后，生成一个 key，在缓存中存储这个 key 对应的手机以及验证码，10 分钟过期
* 将
  `key`
  以及
  `过期时间`
  返回给客户端

> 做接口的思路与我们做网页应用不同，网站中处理验证码，通常是存入 session，注册的时候验证用户输入的验证码与 session 中的验证码是否相同。但是接口是无状态，相互独立的，处理这种相互关联，有先后调用顺序的接口时，常常是第一个接口返回一个随机的 key，利用这个 key 去调用第二个接口。

关于发送成功后返回的随机 key 这里有几种思路：

1. 使用手机号作为 key，不推荐这样，两个并发请求，如果使用了相同的手机号，无论是什么原因导致的这种情况的发生，都会造成某一个验证不通过；
2. 为了防止上面一种情况，可以使用
   `手机号`
   +
   `几位随机字符串`
   ，例如 4 位随机字符串，或者当前时间戳，冲突的概率很小；
3. 使用一个 x 位的随机字符串，一般短时间内的随机字符串，例如 15 位，冲突的概率也很小。

第 2，3 种方式皆可，课程中使用第三种方式。

通过 PostMan 调试一下：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/AONpwfNIJX.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/AONpwfNIJX.png)

如果成功的话，此时你的手机应该能接收到验证短信。

记得将接口保存下来，方便下次调试，我们可以增加一个二级目录`用户`，接口名称为`发送短信验证码`。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/1r1Cf9mS9n.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/07/6351/1r1Cf9mS9n.png)

## 3. 测试环境验证码

我们在本地或者测试环境，其实不必每次都真实发送验证码，为了方便测试，节约短信费用，我们可以增加如下代码

_app/Http/Controllers/Api/VerificationCodesController.php_

```
.
.
.
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
        return $this->response->errorInternal($message ?: '短信发送异常');
    }
}
```

除了正式环境外，其他环境，默认不真实发送短信，短信验证码默认为`1234`。大家也可以考虑增加一个配置，控制是否真实发送短信。

## 4. 代码版本控制

做下代码版本控制：

```
$ git add -A
$ git commit -m "verificationCode store"
```



