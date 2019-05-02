## Passport 安装

### 1. 创建新分支

由于是将原有的认证方式 JWT，替换为 OAuth2，所以我们新建一个分支来进行代码开发

```
$ git checkout -b passport
```

> 需要明白 api 分支实现的是 JWT 的认证逻辑，passport 分支实现的是 Passport 的逻辑，根据你的实际情况使用

### 2. Composer 安装

使用 Composer 安装 Passport ：

```
$ composer require laravel/passport
```

> 如果安装过程中遇到`paragonie/random_compat`版本冲突的问题，可以先执行`composer require paragonie/random_compat:^2.0`将其降级为`2.0`的版本。

[![](https://iocaffcdn.phphub.org/uploads/images/201812/13/3995/oD8D5MzUG7.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201812/13/3995/oD8D5MzUG7.png!large)

### 3. 生成数据表

Passport 扩展包里已经自动注册了迁移文件加载，执行`migrate`，会自动运行扩展包里的迁移文件，由此来创建存储客户端和令牌的数据表：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/HW0YabSCH9.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/HW0YabSCH9.png)

### 4. 创建加密秘钥

接下来，运行`php artisan passport:keys`命令来创建生成安全访问令牌时所需的加密密钥：

```
$ php artisan passport:keys
```

[![](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/RMGMsO8ZaA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/RMGMsO8ZaA.png)

执行成功后，会在`storage`目录中看到两个以`oauth`开头的秘钥文件：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/GSVR5lDwDN.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/GSVR5lDwDN.png)

### 5. 创建客户端

```
$ php artisan passport:client --password --name='larabbs-ios'
```

`passport:client`命令可以创建一个客户端，由于我们使用的是密码模式，所以需要增加`--password`参数。同时还可以增加`--name`参数为客户端起个名字，我们这里起名为`larabbs-ios`：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/zkgqqERElu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/28/3995/zkgqqERElu.png)

命令行中已经输出了创建的`client_id`和`client_secret`，我们找个地方复制保存起来。

## Passport 调试

### 1. 注册路由

安装好了 Passport 我们来调试一下，`Passport::routes`是 Passport 为我们提供了基础的路由，我们先注册一下路由。

_app/Providers/AuthServiceProvider.php_

```
.
.
.
use Carbon\Carbon;
use Laravel\Passport\Passport;
.
.
.
public function boot()
{
    $this->registerPolicies();

    // Passport 的路由
    Passport::routes();
    // access_token 过期时间
    Passport::tokensExpireIn(Carbon::now()->addDays(15));
    // refreshTokens 过期时间
    Passport::refreshTokensExpireIn(Carbon::now()->addDays(30));

    \Horizon::auth(function ($request) {
        // 是否是站长
        return \Auth::user()->hasRole('Founder');
    });
}
.
.
.
```

我们注册了路由，同时通过 Passport 的`tokensExpireIn`和`refreshTokensExpireIn`定义了访问令牌的过期时间，否则访问令牌是永久有效的。这里我们定义`access_token`15 天内有效，`refresh_token`30 天内有效。

### 2. 获取访问令牌

密码模式我们通过[larabbs.test/oauth/token](http://larabbs.test/oauth/token)这个路由获取访问令牌。提交的参数如下

* grant\_type —— 密码模式固定为
  `password`
  ；
* client\_id —— 通过
  `passport:client`
  创建的客户端
  `id`
  ；
* client\_secret —— 通过
  `passport:client`
  创建的客户端
  `secret`
  ；
* username —— 登录的用户名，数据库中任意用户邮箱；
* password —— 用户密码；
* scope —— 作用域，可填写
  `*`
  或者为空；

[![](https://iocaffcdn.phphub.org/uploads/images/201803/01/3995/1TSLzkYzwn.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/01/3995/1TSLzkYzwn.png)

提交正确的`client`信息以及任意已存在用户的用户名和密码，可以正确的获取到访问令牌。

* token\_type —— 令牌类型；
* expires\_in—— 多长时间后过期；
* access\_token —— 访问令牌；
* refresh\_token —— 刷新令牌；

### 3. 刷新访问令牌

`刷新访问令牌`接口与`获取访问令牌`接口一样，只是参数不同。

* grant\_type —— 刷新令牌固定为
  `refresh_token`
  ；
* client\_id —— 通过
  `passport:client`
  创建的客户端
  `id`
  ；
* client\_secret —— 通过
  `passport:client`
  创建的客户端
  `secret`
  ；
* refresh\_token —— 刷新令牌；
* scope —— 作用域，可填写
  `*`
  或者为空；

[![](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/oWsGcs5rNA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/02/3995/oWsGcs5rNA.png)

刷新令牌不用提交用户的用户名和密码，而是直接使用`refresh_token`换取新的访问令牌，注意修改`grant_type`为`refresh_token`。

## 关于 Refresh Token

[![](https://iocaffcdn.phphub.org/uploads/images/201806/15/1/BgL43J7fdM.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201806/15/1/BgL43J7fdM.png?imageView2/2/w/1240/h/0)

为什么要刷新 Access Token 呢？

* 一是因为 Access Token 是有过期时间的，到了过期时间这个 Access Token 就会失效，需要刷新；
* 二是因为一个 Access Token 会关联一定的用户权限，如果用户授权更改了，这个 Access Token 需要被刷新以关联新的权限。

为什么要专门用一个 Token 去更新 Access Token 呢？如果没有 Refresh Token，要获取一个新的 Access Token，就必须重新让用户输入登录用户名与密码，这肯定是不可取的。有了 refresh Token，客户端就可以直接拿着 Refresh Token 去更新 Access Token，继续访问需要的接口。

Refresh Token 也有过期时间但是时间相对较长。 Refresh Token 对存储的要求通常会非常严格，以确保它不会被泄漏；它们也可以被授权服务器列入黑名单。

## 代码版本控制

```
$ git add -A
$ git commit -m 'add passport'
```



