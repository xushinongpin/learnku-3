## 修改话题

本章节我们将一起开发修改话题信息的 API。

## 1. 添加路由

_routes/api.php_

```
.
.
.
// 发布话题
$api->post('topics', 'TopicsController@store')
    ->name('api.topics.store');
$api->patch('topics/{topic}', 'TopicsController@update')
    ->name('api.topics.update');
.
.
.
```

## 2. 修改 Request

_app/Http/Requests/Api/TopicRequest.php_

```
.
.
.
public function rules()
{
    switch($this->method()) {
        case 'POST':
            return [
                'title' => 'required|string',
                'body' => 'required|string',
                'category_id' => 'required|exists:categories,id',
            ];
            break;
        case 'PATCH':
            return [
                'title' => 'string',
                'body' => 'string',
                'category_id' => 'exists:categories,id',
            ];
            break;
    }
}
.
.
.
```

## 3. 修改 Controller

_app/Http/Controllers/Api/TopicsController.php_

```
.
.
.
public function update(TopicRequest $request, Topic $topic)
{
    $this->authorize('update', $topic);

    $topic->update($request->all());
    return $this->response->item($topic, new TopicTransformer());
}
.
.
.
```

## 4. PostMan 调试

首先我们输入一个不存在的话题 id，想象中应该会返回 404

[![](https://iocaffcdn.phphub.org/uploads/images/201803/13/3995/JNZk1xdZ8F.png "file")](https://iocaffcdn.phphub.org/uploads/images/201803/13/3995/JNZk1xdZ8F.png)

但是结果却是 TopicTransformer 报错了，好像是路由模型绑定出了问题。没错，因为路由交给 DingoApi 来处理了，所以模型绑定的中间件并没有注册上。手动增加`bindings`中间件。

_routes/api.php_

```
.
.
.
$api->version('v1', [
    'namespace' => 'App\Http\Controllers\Api',
    'middleware' => ['serializer:array', 'bindings']
], function ($api) {
.
.
.
```

再次访问：

[![](https://iocaffcdn.phphub.org/uploads/images/201904/15/3995/ljW1MuMa83.png!large "修改话题")](https://iocaffcdn.phphub.org/uploads/images/201904/15/3995/ljW1MuMa83.png!large)

## 5. API 的异常处理

状态码正确，但是 message 不是太好，暴露了一些代码细节，关于异常处理，DingoApi 提供了方法，让我们手动处理异常。

_app/Providers/AppServiceProvider.php_

```
.
.
.
public function register()
{
    if (app()->isLocal()) {
        $this->app->register(\VIACreative\SudoSu\ServiceProvider::class);
    }

    \API::error(function  (\Symfony\Component\HttpKernel\Exception\NotFoundHttpException  $exception)  {
        throw  new  \Symfony\Component\HttpKernel\Exception\HttpException(404,  '404 Not Found');  
    });
}
.
.
.
```

再次调用，符合预期：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/BXMjNxlxVj.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/BXMjNxlxVj.png)

> 如果你使用的是`dingo 2.2.3`以下的版本，状态码是会是 500。  
> [![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/CuRAuv0aNx.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/CuRAuv0aNx.png)  
> 可以使用如下代码 :
>
> \API::error\(function \(\Illuminate\Database\Eloquent\ModelNotFoundException $exception\) {  
> abort\(404\);  
> }\);

## 6. 用户权限控制

但是我们尝试修改别人的话题，这里我们使用 id 为 14 的用户，修改 id 为 1 的话题。

> 注意不要使用有`manage_contents`权限的用户，也就是 id 为 1 ，2 的用户，他们有管理内容的权限，所以可以修改任何人的话题。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/PpXXbPIvoV.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/PpXXbPIvoV.png)

同样的，状态码不太符合预期，403 可能更加合适，依然可以在`AppServiceProvider`中手动处理。

_app/Providers/AppServiceProvider.php_

```
.
.
.
public function register()
{
    if (app()->isLocal()) {
        $this->app->register(\VIACreative\SudoSu\ServiceProvider::class);
    }

    \API::error(function (\Illuminate\Database\Eloquent\ModelNotFoundException $exception) {
        abort(404);
    });

    \API::error(function (\Illuminate\Auth\Access\AuthorizationException $exception) {
        abort(403, $exception->getMessage());
    });
}
.
.
.
```

再次调用，符合预期。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/rnM7hFuEdZ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/rnM7hFuEdZ.png)

最后尝试修改自己的话题  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/wo8dc1fOA7.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/wo8dc1fOA7.png)

修改成功，注意 patch 需要使用`x-www-form-urlencoded`。

## 代码版本控制

```
$ git add -A
$ git commit -m 'topic update api'
```



