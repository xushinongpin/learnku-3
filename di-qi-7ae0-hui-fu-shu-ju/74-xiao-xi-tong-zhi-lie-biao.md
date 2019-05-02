## 消息通知列表

接下来我们开发`消息通知`接口，在[第二本教程](https://learnku.com/courses/laravel-intermediate-training/5.7)中我们开发过消息通知的功能，就是当话题有新回复时，我们将通知话题作者。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/Pl8ALawHSw.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/Pl8ALawHSw.png)

## 1. 增加路由

登录用户可以查看自己收到的通知

_routes/api.php_

```
.
.
.
// 删除回复
$api->delete('topics/{topic}/replies/{reply}', 'RepliesController@destroy')
    ->name('api.topics.replies.destroy');
// 通知列表
$api->get('user/notifications', 'NotificationsController@index')
    ->name('api.user.notifications.index');
.
.
.
```

这里同`获取登录用户信息`的思路相同，`user`表示当前登录的用户，`user/notifications`就是`我的通知`。

## 2. 增加 Transformer

```
$ touch app/Transformers/NotificationTransformer.php
```

修改文件

_app/Transformers/NotificationTransformer.php_

```
<?php

namespace App\Transformers;

use League\Fractal\TransformerAbstract;
use Illuminate\Notifications\DatabaseNotification;

class NotificationTransformer extends TransformerAbstract
{
    public function transform(DatabaseNotification $notification)
    {
        return [
            'id' => $notification->id,
            'type' => $notification->type,
            'data' => $notification->data,
            'read_at' => $notification->read_at ? $notification->read_at->toDateTimeString() : null,
            'created_at' => $notification->created_at->toDateTimeString(),
            'updated_at' => $notification->updated_at->toDateTimeString(),
        ];
    }
}
```

注意这里我们需要格式化的模型是`Illuminate\Notifications\DatabaseNotification`。

## 3. 增加 Controller

```
$ php artisan make:controller Api/NotificationsController
```

修改文件

_app/Http/Controllers/Api/NotificationsController.php_

```
<?php

namespace App\Http\Controllers\Api;

use Illuminate\Http\Request;
use App\Transformers\NotificationTransformer;

class NotificationsController extends Controller
{
    public function index()
    {
        $notifications = $this->user->notifications()->paginate(20);

        return $this->response->paginator($notifications, new NotificationTransformer());
    }
}
```

用户模型的 notifications 方法是[Laravel 的消息通知系统](https://learnku.com/docs/laravel/5.5/notifications)为我们提供的方法，按通知创建时间倒叙排序。

## 4. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/Qy2SgYokIY.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/Qy2SgYokIY.png)

新增`消息通知`目录保存接口

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/DGAi8YJMDq.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/DGAi8YJMDq.png)

## 5. 关于返回的数据

大家注意到了我们返回的数据里，`reply_content`是 HTML 形式返回的，客户端可使用系统内置的 WebView UI 组件来渲染。iOS 有[UIWebView](https://developer.apple.com/documentation/uikit/uiwebview)，Android 有[WebView](https://developer.android.com/reference/android/webkit/WebView.html)。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/05/1/SXFdK0cK1k.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/05/1/SXFdK0cK1k.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'notifications index'
```



