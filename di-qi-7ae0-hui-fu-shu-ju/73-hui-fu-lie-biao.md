## 某个话题的回复列表

### 1. 添加路由

第一步我们先添加路由，请注意该接口游客是可以访问的：

_routes/api.php_

```
.
.
.
// 某个用户发布的话题
$api->get('users/{user}/topics', 'TopicsController@userIndex')
    ->name('api.users.topics.index');
// 话题回复列表
$api->get('topics/{topic}/replies', 'RepliesController@index')
    ->name('api.topics.replies.index');
.
.
.
```

### 2. 修改 Controller

_app/Http/Controllers/Api/RepliesController.php_

```
public function index(Topic $topic)
{
    $replies = $topic->replies()->paginate(20);

    return $this->response->paginator($replies, new ReplyTransformer());
}
```

代码很简单，分页查询话题的所有评论，使用`ReplyTransformer`转换评论数据并返回。

### 3. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/VNbmNDZZr6.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/VNbmNDZZr6.png)

响应数据中包括中该话题的评论数据，及分页数据。

### 4. 调整 Include 参数

我们需要的不仅仅是回复数据，还需要显示回复人姓名，头像等用户数据。

再次阅读并回忆一下[Include 机制](https://learnku.com/courses/laravel-advance-training/5.5/923/list-of-posts#Include-%E6%9C%BA%E5%88%B6)，当我们需要在资源数据中，嵌套返回该资源`相关的其他资源`时，可以利用这个机制快速的实现。

设置`Transformer`中的`availableIncludes`参数

_app/Transformers/ReplyTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Reply;
use League\Fractal\TransformerAbstract;

class ReplyTransformer extends TransformerAbstract
{
    protected $availableIncludes = ['user'];
.
.
.
    public function includeUser(Reply $reply)
    {
        return $this->item($reply->user, new UserTransformer());
    }
}
```

增加`include=user`再次使用 PostMan 调试  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/LL08bW7bWx.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/LL08bW7bWx.png)

因为多了 include 参数，数据中多了用户数据。

## 某个用户回复列表

除了某个话题的回复，我们还可能查看某个用户发布的所有回复

[![](https://iocaffcdn.phphub.org/uploads/images/201801/30/3995/seMVOW4YSx.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/30/3995/seMVOW4YSx.png)

### 1. 添加路由

_routes/api.php_

```
.
.
.
// 话题回复列表
$api->get('topics/{topic}/replies', 'RepliesController@index')
    ->name('api.topics.replies.index');
// 某个用户的回复列表
$api->get('users/{user}/replies', 'RepliesController@userIndex')
    ->name('api.users.replies.index');
.
.
.
```

### 2. 修改 Controller

_app/Http/Controllers/Api/RepliesController.php_

```
.
.
.
use App\Models\User;
.
.
.
public function userIndex(User $user)
{
    $replies = $user->replies()->paginate(20);

    return $this->response->paginator($replies, new ReplyTransformer());
}
.
.
.
```

分页查询用户的所有评论，使用`ReplyTransformer`转换评论数据并返回。

### 3. 修改 Transformer

注意回复列表中，需要显示回复话题的标题，也就是我们需要`回复资源`关联的`话题资源`。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/30/3995/fftlljOyDa.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/30/3995/fftlljOyDa.png)

_app/Transformers/ReplyTransformer.php_

```
.
.
.
protected $availableIncludes = ['user', 'topic'];
.
.
.
public function includeTopic(Reply $reply)
{
    return $this->item($reply->topic, new TopicTransformer());
}
.
.
.
```

`availableIncludes`中增加了`topic`，增加了对应的`includeTopic`方法，查询出回复关联的话题模型，使用`TopicTransformer`转换并返回。

### 4. 使用 PostMan 调试

注意设置变量  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/r0znnJW4qS.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/r0znnJW4qS.png)

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/5oFaKsiQfm.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/5oFaKsiQfm.png)

返回`回复数据`以及`回复的话题数据`。

### 发布话题的用户数据

假设现在的客户端界面进行了调整，某个用户的回复列表页面，不仅需要显示话题的标题，还需要显示`发布话题`的用户的头像及姓名，也就是除了回复关联的话题资源，还需要话题关联的用户资源。

**数据该如何嵌套，客户端界面变化了，我们需要调整接口吗**？

其实代码我们已经完成了，客户端只需要调整请求参数即可

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/UlQpuF47D0.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/UlQpuF47D0.png)

注意我们传入的`include`参数为`topic.user`，意思是包含`话题资源`关联的`用户资源`，用户数据嵌套在话题数据中。  
相信你应该能发现 include 参数中`逗号`与`点`的区别。

* 逗号 —— 是当前资源所关联的资源，如
  `include=topic,user`
  ；
* 点 —— 当前资源所关联的资源，及其所关联的资源，相当于下一级资源，如
  `include=topic.user`
  ；

因为回复的话题是通过`TopicTransformer`格式化的：

```
public function includeTopic(Reply $reply)
{
    return $this->item($reply->topic, new TopicTransformer());
}
```

所以 TopicTransformer 中`$availableIncludes`包含的资源，我们都可以使用`点`继续嵌套关联，例如`include=topic.user,topic.category`。

我们是在面向资源处理数据，接口需要做的是，利用资源之间的关联，让客户端通过不同的参数组合，获取需要的资源，可以看到 Include 机制非常灵活和方便。

#### 是否有 N+1 问题呢？

查看一下日志

```
$ tail -f ./storage/logs/laravel.log
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/GtJQN8W9x2.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/GtJQN8W9x2.png)

DingoApi 已经帮我们处理好了

## 代码版本控制

```
$ git add -A
$ git commit -m 'replies index'
```



