## 权限列表

在这个章节中，我们来开发权限数据接口。客户端可以在用户登录成功之后，请求`权限列表接口`，缓存在本地，渲染页面的时候，根据用户权限，以及用户与资源的关系，来完成对页面显示的控制。

## 1. 添加路由

首先需要获取『自己的』权限，我们使用`user`表示`我`的概念，所以设计为`user/permissions`。

```
.
.
.
// 标记消息通知为已读
$api->patch('user/read/notifications', 'NotificationsController@read')
    ->name('api.user.notifications.read');
// 当前登录用户权限
$api->get('user/permissions', 'PermissionsController@index')
    ->name('api.user.permissions.index');
.
.
.
```

## 2. 增加 Transformer

```
$ touch app/Transformers/PermissionTransformer.php
```

_app/Transformers/PermissionTransformer.php_

```
<?php

namespace App\Transformers;

use Spatie\Permission\Models\Permission;
use League\Fractal\TransformerAbstract;

class PermissionTransformer extends TransformerAbstract
{
    public function transform(Permission $permission)
    {
        return [
            'id' => $permission->id,
            'name' => $permission->name,
        ];
    }
}
```

## 3. 增加 Controller

```
$ php artisan make:controller Api/PermissionsController
```

_app/Http/Controllers/Api/PermissionsController.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use App\Transformers\PermissionTransformer;

class PermissionsController extends Controller
{
    public function index()
    {
       $permissions = $this->user()->getAllPermissions();

       return $this->response->collection($permissions, new PermissionTransformer());
    }
}
```

## 4. PostMan 调试

使用 id 为 1 的用户（站长）访问权限列表接口，看到该用户拥有全部权限。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/TpCY9Iw5bB.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/TpCY9Iw5bB.png)

使用 id 为 2 的用户（管理员）访问权限列表接口，看到该用户拥有`manage_contents`权限。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/L2X7bf1901.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/L2X7bf1901.png)

使用 id 为 3 的用户（普通用户）访问权限列表接口，该用户没有权限。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/F4IyMZ68ZO.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/F4IyMZ68ZO.png)

## 安全问题

**将用户权限请求后缓存在本地会不会引入安全问题呢？**

我们知道客户端的一切都是可以通过某种途径修改的，例如反编译 APP 后，修改代码，将某个用户权限修改为站长拥有的所有权限。这样是不是就代表用户可以修改或删除所有的话题了呢？

其实不是的，客户端缓存的权限列表，只是用于控制界面显示。数据的操作权限是在服务器端，接口服务器在执行某个操作时，始终会判断用户权限。例如`TopicsController`中的代码

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

public function destroy(Topic $topic)
{
    $this->authorize('destroy', $topic);  

    $topic->delete();
    return $this->response->noContent();
}
.
.
.
```

在这个例子中，`话题修改接口`和`话题删除接口`都会通过[授权策略](https://learnku.com/docs/laravel/5.5/authorization)提供的`authorize`方法来判断了用户是否具备某个操作的权限。

## 代码版本控制

```
$ git add -A
$ git commit -m 'permissions index'
```



