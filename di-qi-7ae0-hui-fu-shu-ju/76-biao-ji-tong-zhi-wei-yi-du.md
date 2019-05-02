## 标记通知为已读

我们还没有标记通知数据为已读，有些同学可能会在`消息通知列表`接口中将所有未读消息标记为已读，只要调用了列表接口，就意味着消息已读。  
这么做看似没有什么问题，但是违背了一些原则，也带来了一些问题。回忆一下[Github 的 Restful HTTP API 设计分解](https://learnku.com/courses/laravel-advance-training/5.5/787/follow-github-to-learn-restful-http-api-design)这一节我们提到了 GET 是安全的请求。

> 另外需要注意的是，GET 请求是安全的，不允许通过 GET 请求改变（更新或创建）资源。

`消息通知列表`接口是 GET 请求，不应该在这时候改变资源数据，而且客户端可能会有其他的方式标记已读，例如有个按钮`标记所有通知为已读`。我们需要让接口符合规范，而且更加通用，所以一般需要客户端主动调用接口，来标记消息已读。

## 1. 增加路由

_routes/api.php_

```
.
.
.
// 通知统计
$api->get('user/notifications/stats', 'NotificationsController@stats')
    ->name('api.user.notifications.stats');
// 标记消息通知为已读
$api->patch('user/read/notifications', 'NotificationsController@read')
    ->name('api.user.notifications.read');
.
.
.
```

这里我们参考了 Github Api[Starring](https://developer.github.com/v3/activity/starring/#star-a-repository)的部分，`PUT /user/starred/:owner/:repo`为 star 某个仓库，同样标记单个通知为已读我们可以设计为`PUT /user/read/notifications/{notification_id}`，但是这里我们会批量将所有未读消息标记为已读，考虑到幂等性原则，使用 PATCH 更为合适，最终设计为`PATCH /user/read/notifications`。

## 2. 修改 Controller

_app/Http/Controllers/Api/NotificationsController.php_

```
.
.
.
public function read()
{
    $this->user()->markAsRead();

    return $this->response->noContent();
}
.
.
.
```

`markAsRead`是上一本教程中已经处理好的方法，会将用户`notification_count`设置为 0，将所有未读消息设置为已读。代码如下：

_app\Models\User.php_

```
public function markAsRead()
{
    $this->notification_count = 0;
    $this->save();
    $this->unreadNotifications->markAsRead();
}
```

## 3. 使用 PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/fCYDntYnlb.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/fCYDntYnlb.png)

再次访问`消息通知统计接口`

[![](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/u5Pk0ax6xf.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/28/3995/u5Pk0ax6xf.png)

未读消息已经清零。

## 代码版本控制

```
$ git add -A
$ git commit -m 'notifications read'
```



