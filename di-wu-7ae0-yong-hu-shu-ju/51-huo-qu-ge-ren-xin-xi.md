## Fractal

[Fractal](https://github.com/thephpleague/fractal)是一个转换层（transformer），API 开发中非常方便的一种开发方法，可以帮助我们处理响应数据的结构与复杂的嵌套关系，最后将数据返回给客户端。可以把 Fractal 理解为 Web 开发中视图，控制着 API 的最终数据输出。Laravel 5.5 的新功能`eloquent-resources`整体思路跟`Fractal`一致，用法也基本相同。

> 这里有相关的视频教程[https://learnku.com/courses/laravel-package/conversion-of-api-data-to-leaguefractal-using-dingoapi/2500](https://learnku.com/courses/laravel-package/conversion-of-api-data-to-leaguefractal-using-dingoapi/2500)可以参考一下

## `Fractal`还是`eloquent-resources`

* 首先 Fractal 是一个比较成熟的组件，我个人从 Laravel 5.1 开始一直在使用，有大量实践经验；
* 我们目的是让大家学会处理的思路，相信大家理解一个之后应该很容易掌握另一个；
* Fractal 有更加方便的数据嵌套过滤器，如：
  `?include=topics,topics.user`
  ，用例更广；
* DingoApi 已经安装了
  `Fractal`
  ，并且做了很多整合，基本解决了 N+1 问题，我们可以快速的开始使用；

基于上面几个原因，本教程会选择使用`Fractal`。

## 数据结构

Fractal 为我们提供了 3 种[基本结构](https://fractal.thephpleague.com/serializers/)。

* DataArraySerializer 结构类似 eloquent-resources 的 Data Wrapping
* ArraySerializer 结构类似 eloquent-resources 的 withoutWrapping
* JsonApiSerializer 结构出自
  [JSON-API](http://jsonapi.org/)
  ，是一套 json 接口响应规范。

我们常用的结构是 DataArraySerializer 和 ArraySerializer：

1. DataArraySerializer

   ```
   // 带着 meta 信息的单条数据
   [
       'data' => [
           'foo' => 'bar'
       ],
       'meta' => [
           ...
       ]
   ];

   // 带着 meta 信息的数据集合
   [
       'data' => [
           [
               'foo' => 'bar'
           ]
       ],
       'meta' => [
           ...
       ]
   ];
   ```

2. ArraySerializer

   ```
   // 带着 meta 信息的单条数据
   [
       'foo' => 'bar'
       'meta' => [
           ...
       ]
   ];

   // 带着 meta 信息的数据集合
   [
       'data' => [
           [
               'foo' => 'bar'
           ]
       ],
       'meta' => [
           ...
       ]
   ];
   ```

对比一下不难发现，`ArraySerializer`类似于`eloquent-resources`的`withoutWrapping`。对于集合来说，两者没有差别，有 data 和 meta 构成，对于 item 来说，就是少了一层`data`包裹。

* DataArraySerializer 将所有结果的
  `data`
  与
  `meta`
  区分来，结构统一。
* 当多个资源嵌套返回的时候，例如话题及发布话题的用户，多一层 data 会让结构更深一层。前端的同学可能会这么显示某个发布话题用户的姓名，
  `data.topics[0].user.data.name`
  ，所以
  `ArraySerializer`
  会减少数据嵌套层级。

上面的解释比较抽象，不要担心，接下来我们将会对 ArraySerializer 和 DataArraySerializer 一一进行说明。

## 创建 UserTransformer

我们首先为用户创建一个数据转换层 UserTransformer。没有找到好用的组件帮助我们创建 Transformer，我们暂时先手动创建。  
在`app`中增加了`Transformers`目录，新建一个文件`UserTransformer.php`

```
$ mkdir app/Transformers
$ touch app/Transformers/UserTransformer.php
```

修改文件如下

_app/Transformers/UserTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\User;
use League\Fractal\TransformerAbstract;

class UserTransformer extends TransformerAbstract
{
    public function transform(User $user)
    {
        return [
            'id' => $user->id,
            'name' => $user->name,
            'email' => $user->email,
            'avatar' => $user->avatar,
            'introduction' => $user->introduction,
            'bound_phone' => $user->phone ? true : false,
            'bound_wechat' => ($user->weixin_unionid || $user->weixin_openid) ? true : false,
            'last_actived_at' => $user->last_actived_at->toDateTimeString(),
            'created_at' => $user->created_at->toDateTimeString(),
            'updated_at' => $user->updated_at->toDateTimeString(),
        ];
    }
}
```

使用起来很简单，只需要给`transformer`方法传入一个模型实例，然后返回一个数据即可，这个数组就是返回给客户端的响应数据。  
`UserTransformer`是可以复用的，当前用户信息，发布话题用户信息，话题回复用户信息都可以用这一个`transformer`，这样我们所有的有关`用户`的资源都会返回相同的信息，客户端只需要解析一遍即可。因为是可复用的，特别需要注意一些敏感信息，如用户手机，微信的`union_id`等，我们可以使用另外的字段返回。

* `bound_phone`
  是否绑定手机
* `bound_wechat`
  是否绑定微信

或者可以返回`phone`但是部分手机数字用`*`替换，总之就是需要保护用户敏感信息。

## 获取用户信息

登录获取了 token 之后，第一件事就是需要换取用户信息，先来实现`获取登录用户信息`接口。增加路由：

_routes/api.php_

```
.
.
.
    $api->group([
        'middleware' => 'api.throttle',
        'limit' => config('api.rate_limits.access.limit'),
        'expires' => config('api.rate_limits.access.expires'),
    ], function ($api) {
        // 游客可以访问的接口

        // 需要 token 验证的接口
        $api->group(['middleware' => 'api.auth'], function($api) {
            // 当前登录用户信息
            $api->get('user', 'UsersController@me')
                ->name('api.user.show');
        });
    });
.
.
.
```

还记得我们添加的调用频率限制吗，先增加一个 group，剩下的接口统一 1 分钟只能调 用 60 次。接下来的接口我们可以分为两类

* 游客可以访问的接口
* 登录用户才可以访问的接口

DingoApi 为我们准备好了`api.auth`这个中间件，用来区分哪些接口需要验证 token，哪些不需要。最后我们增加了一个路由`/api/user`。注意这个 user 是单数，表示`我`的意思，主要参考了 Github 的设计思路。

_app/Http/Controllers/Api/UsersController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\User;
use Illuminate\Http\Request;
use App\Transformers\UserTransformer;
use App\Http\Requests\Api\UserRequest;
.
.
.
    public function me()
    {
        return $this->response->item($this->user(), new UserTransformer());
    }
}
```

还记得我们增加的`Dingo\Api\Routing\Helpers`这个 trait 吗，它提供了`user`方法，方便我们获取到当前登录的用户，也就是 token 所对应的用户，`$this->user()`等同于`\Auth::guard('api')->user()`。  
我们返回的是一个单一资源，所以使用`$this->response->item`，第一个参数是模型实例，第二个参数是刚刚创建的 transformer。

使用 PostMan 测试该接口

[![](https://iocaffcdn.phphub.org/uploads/images/201801/20/6351/BIFmjvxFK3.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/20/6351/BIFmjvxFK3.png)

没有传入 token 时，返回 401。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/20/6351/RYZNy53Gk5.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/20/6351/RYZNy53Gk5.png)

传入正确的 token 后，返回正确的用户信息。

## 响应结构保持统一

请注意上面返回用户数据的数据结构：

```
{
    "data": {
        "id": 1,
        "name": "Summer",
         ...
        "created_at": "2017-12-17 15:40:27",
        "updated_at": "2017-12-17 15:40:28"
    }
}
```

单一资源有一层`data`Key 进行包裹，是因为 DingoApi 默认使用的 Fractal 的`DataArraySerializer`，但是回忆一下我们返回的 token 数据，是通过`response->array`直接返回的，并没有`data`：

```
{
    "access_token": "xxx",
    "token_type": "Bearer",
    "expires_in": 3600
}
```

响应数据的结构统一是十分重要的，我们需要保持统一。

考虑到前端同事们对接的方便程度，我们选择少一层嵌套的`ArraySerializer`。这里有一个[中间件](https://github.com/liyu001989/dingo-serializer-switch)可以方便的切换两种数据结构。执行以下命令安装：

```
$ composer require liyu/dingo-serializer-switch
```

安装成功后，在路由文件里修改：

_routes/api.php_

```
.
.
.
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => 'serializer:array'
], function ($api) {
.
.
.
```

增加中间件`serializer`，参数为`array`。再次调用`获取登录用户信息`接口  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/20/3995/5cVu9Qwjlk.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/20/3995/5cVu9Qwjlk.png)

可以看到，此时我们的数据结构已经切换为`ArraySerializer`了。

## 用户注册响应数据

回忆一下用户[手机注册功能开发](https://learnku.com/courses/laravel-advance-training/5.5/792/development-of-mobile-phone-registration-function)这一节，最后我们只是返回了`return $this->response->created();`并没有返回数据。如果你的应用，用户注册后，跳转到登录页面，让用户重新输入用户名密码登录，这么做没什么问题。但是注册后直接登录该用户，那就需要返回一些数据。  
首先这里参考 Github 的接口，创建资源后，我们统一返回资源数据，状态码为 201。  
我们已经有了 UserTransformer，直接修改代码如下:

_app/Http/Controllers/Api/UsersController.php_

```
public function store(UserRequest $request)
{
.
.
.
    return $this->response->item($user, new UserTransformer())
        ->setStatusCode(201);
}
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/21/3995/Vh1Wp9b5Fu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/21/3995/Vh1Wp9b5Fu.png)

但是 token 数据应该放在哪里呢？这个时候，meta 就很好用了

_app/Http/Controllers/Api/UsersController.php_

```
public function store(UserRequest $request)
{
.
.
.
    return $this->response->item($user, new UserTransformer())
        ->setMeta([
            'access_token' => \Auth::guard('api')->fromUser($user),
            'token_type' => 'Bearer',
            'expires_in' => \Auth::guard('api')->factory()->getTTL() * 60
        ])
        ->setStatusCode(201);
}
```

测试一下  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/Sz67LOGooZ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/Sz67LOGooZ.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'api/user'
```



