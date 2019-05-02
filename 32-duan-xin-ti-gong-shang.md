## 1. 服务商注册

短信的服务商有很多，我们这里为了教学方便，选择[云片](https://www.yunpian.com/)作为我们的短信服务商，注册成功后，会有 10 条短信的免费额度，作为教学测试应该足够使用了。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/O9EuhMMk7N.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/O9EuhMMk7N.png)

点击注册按钮，进入云片的注册页面，注册云片账户。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/WIzrbih3qD.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/WIzrbih3qD.png)

注册成功后，会要求填写姓名及公司名称，大家可以填一下自己所在的公司，如果是学生可以填写你班级的信息，编一个也行，我们就是想试用一下而已。之后会有云片的客服打电话回访，回访内容大概是公司使用情况，回访内容不会影响账户的使用的。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/zbMf40hXlg.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/zbMf40hXlg.png)

根据相关政策的要求，各个短信服务商都会要求我们设置『签名』以及『模板』后，才允许发送验证码，即使你不使用云片，流程也大体相同。

首先来添加『签名』，签名一般是在短信内容开始或者末尾跟的品牌或者应用名称，比如刚才注册云片大家收到的云片网的短信验证码，`【云片网】`就是签名。

```
【云片网】云片网验证功能码：xxxxxx
```

不过首先我们需要先填写开发者信息。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/JBA7ThAhcA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/JBA7ThAhcA.png)

选择个人，提交一下证件照片即可。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/PV4SFb7VfO.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/PV4SFb7VfO.png)

回到添加签名的界面，新增一个签名，输入`Lbbs社区`。提交后等待云片审核通过。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/ZvXheXbg6U.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/12/6351/ZvXheXbg6U.png)

> 你应该会显示『签名已存在』，这时候请选择一个别的名称即可，不要求一样。

点击模板管理，如图所示添加短信模板。等待云片审核通过后就可以使用了。

## 2. 安装 easy-sms

[easy-sms](https://github.com/overtrue/easy-sms)是安正超写的一个短信发送组件，利用这个组件，我们可以快速的实现短信发送功能。

```
$ composer require "overtrue/easy-sms"
```

由于该组件还没有 Laravel 的 ServiceProvider，为了方便使用，我们可以自己封装一下。  
首先在 config 目录中增加 easysms.php 文件，

```
$ touch config/easysms.php
```

填入如下内容。  
config/easysms.php

```
<?php
return [
    // HTTP 请求的超时时间（秒）
    'timeout' => 5.0,

    // 默认发送配置
    'default' => [
        // 网关调用策略，默认：顺序调用
        'strategy' => \Overtrue\EasySms\Strategies\OrderStrategy::class,

        // 默认可用的发送网关
        'gateways' => [
            'yunpian',
        ],
    ],
    // 可用的网关配置
    'gateways' => [
        'errorlog' => [
            'file' => '/tmp/easy-sms.log',
        ],
        'yunpian' => [
            'api_key' => env('YUNPIAN_API_KEY'),
        ],
    ],
];
```

然后创建一个 ServiceProvider

```
$ php artisan make:provider EasySmsServiceProvider
```

修改文件

app/providers/EasySmsServiceProvider.php

```
<?php

namespace App\Providers;

use Overtrue\EasySms\EasySms;
use Illuminate\Support\ServiceProvider;

class EasySmsServiceProvider extends ServiceProvider
{
    /**
     * Bootstrap the application services.
     *
     * @return void
     */
    public function boot()
    {
        //
    }

    /**
     * Register the application services.
     *
     * @return void
     */
    public function register()
    {
        $this->app->singleton(EasySms::class, function ($app) {
            return new EasySms(config('easysms'));
        });

        $this->app->alias(EasySms::class, 'easysms');
    }
}
```

最后 打开 config/app.php 在 providers 中增加`App\Providers\EasySmsServiceProvider::class,`

```
.
.
.
App\Providers\AppServiceProvider::class,
App\Providers\AuthServiceProvider::class,
// App\Providers\BroadcastServiceProvider::class,
App\Providers\EventServiceProvider::class,
App\Providers\RouteServiceProvider::class,

App\Providers\EasySmsServiceProvider::class,
.
.
.
```

接下来我们到云片后台获取 API 的 KEY：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/24/3995/OCWJXrJnpr.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/24/3995/OCWJXrJnpr.png)

在`.env`中配置`YUNPIAN_API_KEY`，注意下面需要替换为你自己的 key：

```
.
.
.
# 云片
YUNPIAN_API_KEY=9c60bdd**********
```

在`.env.example`中也加入配置示例，提交到版本库，方便以后部署

```
# 云片
YUNPIAN_API_KEY=
```

## 3. 调试短信

我们使用 artisan 调试一下，试试能否收到短信。  
打开 tinker

```
$ php artisan tinker
```

输入如下代码，注意将`13212345678`替换为你自己的手机。

```
$sms = app('easysms');
try {
    $sms->send(13212345678, [
        'content'  => '【Lbbs社区】您的验证码是1234。如非本人操作，请忽略本短信',
    ]);
} catch (\Overtrue\EasySms\Exceptions\NoGatewayAvailableException $exception) {
    $message = $exception->getException('yunpian')->getMessage();
    dd($message);
}
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/06/6351/BsWzjU3k7X.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/06/6351/BsWzjU3k7X.png)

相信你的手机上已经收到验证码了。

## 4. 代码版本控制

最后提交代码提交。

```
$ git add -A
$ git commit -m "add easysms"
```



