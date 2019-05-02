## 话题列表

参考以下界面：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/31/3995/rsAOw6xEri.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/31/3995/rsAOw6xEri.png)

可以通过分类搜索，根据创建时间及最新回复排序，列表显示了用户信息，话题信息，分类信息。

### 1. 添加路由

话题列表游客可以访问

_routes/api.php_

```
.
.
.
// 游客可以访问的接口
$api->get('categories', 'CategoriesController@index')
    ->name('api.categories.index');
$api->get('topics', 'TopicsController@index')
    ->name('api.topics.index');
.
.
.
```

### 2. 修改 Controller

_app/Http/Controllers/Api/TopicsController.php_

```
.
.
.
    public function index(Request $request, Topic $topic)
    {
        $query = $topic->query();

        if ($categoryId = $request->category_id) {
            $query->where('category_id', $categoryId);
        }

        // 为了说明 N+1问题，不使用 scopeWithOrder
        switch ($request->order) {
            case 'recent':
                $query->recent();
                break;

            default:
                $query->recentReplied();
                break;
        }

        $topics = $query->paginate(20);

        return $this->response->paginator($topics, new TopicTransformer());
    }
.
.
.
```

PostMan 调用  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/mrMQc2LYiU.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/mrMQc2LYiU.png)  
返回了话题列表，并且有分页数据

## Include 机制

话题数据我们有了，但是发布话题的`用户数据`，以及话题的`分类数据`还没有，那么

* 如何获取额外的资源数据？
* 资源数据该以什么样的结构返回？

其实 DingoApi 已经很好的解决了这两个问题。  
首先修改`TopicTransformer`如下

_app/Transformers/TopicTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Topic;
use League\Fractal\TransformerAbstract;

class TopicTransformer extends TransformerAbstract
{
    protected $availableIncludes = ['user', 'category'];

    public function transform(Topic $topic)
    {
        return [
            'id' => $topic->id,
            'title' => $topic->title,
            'body' => $topic->body,
            'user_id' => $topic->user_id,
            'category_id' => $topic->category_id,
            'reply_count' => $topic->reply_count,
            'view_count' => $topic->view_count,
            'last_reply_user_id' => $topic->last_reply_user_id,
            'excerpt' => $topic->excerpt,
            'slug' => $topic->slug,
            'created_at' => $topic->created_at->toDateTimeString(),
            'updated_at' => $topic->updated_at->toDateTimeString(),
        ];
    }

    public function includeUser(Topic $topic)
    {
        return $this->item($topic->user, new UserTransformer());
    }

    public function includeCategory(Topic $topic)
    {
        return $this->item($topic->category, new CategoryTransformer());
    }
}
```

上面的代码中，我们首先设置了`protected $availableIncludes = ['user', 'category']`，可以理解为可以嵌套的额外资源有`user`和`category`。那么额外的资源如何获取，如何转换，则通过`includeUser`和`includeCategory`确定，`availableIncludes`中的每一个参数都对应一个具体的方法，方法命名规则为`include + user`、`include + category`驼峰命名。

那么什么时候才会引入额外的资源呢，由客户端提交的 include 参数指定，多个参数通过逗号分隔。

url 中增加`include=user`参数，调用`话题列表`接口。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/NvDfPBuh66.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/NvDfPBuh66.png)

可以看到用户数据，并且是通过 UserTransformer 转换过后的数据

url 中增加`include=user,category`参数，调用`话题列表`接口。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/ISWxk3czpQ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/ISWxk3czpQ.png)

可以看到数据中增加了`user`和`category`。

`UserTransformer`和`CategoryTransformer`在这里被复用，通常情况，每个资源只需要写好一个对应的 Transformer 即可。

整理一下完整的流程

* 访问 api/topics?include=user,category
* TopicsController 中查询出来话题，最终使用
  `return $this-`
  `>`
  `response-`
  `>`
  `paginator($topics, new TopicTransformer());`
  返回响应。
* TopicTransformer 处理每一个
  `话题模型`
  ，使用 transform 方法转化模型数据
* 因为你请求中指定了 include=user，所以
  `TopicTransformer`
  自动调用 includeUser 方法。
* includeUser 方法中使用，查询到用户数据
  `$topic-`
  `>`
  `user`
  ，通过
  `UserTransformer`
  格式化用户数据
* 重复上面两步，处理 category 分类相关的数据
* 最终 Fractal 帮我们整理好资源之间的嵌套关系，返回响应。

在 Transformer 中，我们可以使用：

* $this-
  &gt;
  item \(\) 返回单个资源
* $this-
  &gt;
  collection \(\) 返回集合资源

Include 让资源与资源之间以一种合理的嵌套返回，同时什么时候返回完全由请求参数决定，这让资源数据更加灵活。

## N+1 问题以及查询日志

你可能会发现，我们并没有手动通过`with`或者`load`预加载模型关系，那么会不会带来`N+1`问题呢。首先我们需要输出 sql 查询日志，[laravel-query-logger](https://github.com/overtrue/laravel-query-logger)是安正超写的一个查询日志组件，先来安装它

```
$ composer require overtrue/laravel-query-logger --dev
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/U52DYE8lDc.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/U52DYE8lDc.png)

打开日志，再次调用接口[http://larabbs.test/api/topics?include=user,category](http://larabbs.test/api/topics?include=user,category)

```
$ tail -f ./storage/logs/laravel.log
```

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/udXZTR78XW.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/udXZTR78XW.png)

并没有产生 N+1 问题，因为当我们返回的是集合的时候，DingoApi 根据 include 参数以及`defaultInclude`帮助我们进行预加载，所以大部分情况我们不需要手动处理。如果遇到复杂的嵌套及关系加载，可以`app(\Dingo\Api\Transformer\Factory::class)->disableEagerLoading();`临时关闭 DingoApi 的预加载，手动处理。

## 某个用户发布的话题列表

除了首页不同分类的话题列表，我们还可能查看某个用户发布的所有话题  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/fY5CE4HlTA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/fY5CE4HlTA.png)

### 1. 添加路由

某个用户的发布的话题，同样是游客可访问的

_routes/api.php_

```
.
.
.
$api->get('topics', 'TopicsController@index')
    ->name('api.topics.index');
$api->get('users/{user}/topics', 'TopicsController@userIndex')
    ->name('api.users.topics.index');
.
.
.
```

### 2. 修改 Controller

按创建时间倒叙返回该用户所有的话题。

_app/Http/Controllers/Api/TopicsController.php_

```
.
.
.
use App\Models\User;
.
.
.
public function userIndex(User $user, Request $request)
{
    $topics = $user->topics()->recent()
        ->paginate(20);

    return $this->response->paginator($topics, new TopicTransformer());
}
.
.
.
```

PostMan 调用接口  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/dAxphB7rnY.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/dAxphB7rnY.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'topic index'
```



