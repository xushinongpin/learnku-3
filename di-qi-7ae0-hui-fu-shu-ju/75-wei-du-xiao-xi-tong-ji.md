## 未读消息数量

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/YnaQeHJCKa.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/YnaQeHJCKa.png)

在网页端，用户有未读消息了，会在 header 中有红色提示。对于 APP 来说，需要一个接口查询当前用户`未读消息数量`。

## 1. 增加路由

_routes/api.php_

```
.
.
.
// 通知列表
$api->get('user/notifications', 'NotificationsController@index')
    ->name('api.user.notifications.index');
// 通知统计
$api->get('user/notifications/stats', 'NotificationsController@stats')
    ->name('api.user.notifications.stats');
.
.
.
```

这里我们设计为`user/notifications/stats`，stats 是 statistics 的缩写，意思是统计，这个接口可以直观的表述为 —— 我的通知数据统计。

## 2. 修改 Controller

_app/Http/Controllers/Api/NotificationsController.php_

```
.
.
.
public function stats()
{
    return $this->response->array([
        'unread_count' => $this->user()->notification_count,
    ]);
}
.
.
.
```

当有新的通知时，`App\Observers\ReplyObserver.php`已经帮我们进行了统计。

```
// 如果评论的作者不是话题的作者，才需要通知
if ( ! $reply->user->isAuthorOf($topic)) {
    $topic->user->notify(new TopicReplied($reply));
}
```

notify 方法会将 notification\_count 进行 +1。所以`$this->user()->notification_count;`就是用户未读消息数。

## 3. PostMan 调试

可以先登录`larabbs.test`回复某个用户的话题，为该用户新增几个未读通知。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/4oZAUsKdNb.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/4oZAUsKdNb.png)

新增`消息通知`目录，保存接口。

## 代码版本控制

```
$ git add -A
$ git commit -m 'notifications stats'
```



