## 话题详情

除了列表，我们还有可能获取单个话题的数据，下面来开发`话题详情`接口。

### 1. 添加路由

_routes/api.php_

```
.
.
.
$api->get('topics', 'TopicsController@index')
    ->name('api.topics.index');
$api->get('topics/{topic}', 'TopicsController@show')
    ->name('api.topics.show');
.
.
.
```

### 2. 修改 controller

_app/Http/Controllers/Api/TopicsController.php_

```
.
.
.
public function show(Topic $topic)
{
    return $this->response->item($topic, new TopicTransformer());
}
.
.
.
```

PostMan 调用接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/9vV2xoWqlk.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/9vV2xoWqlk.png)

可以看到，当 Transformer 准备好了之后，开发相关的接口是很方便的。详情接口依然可以使用 include 参数。

## 代码版本控制

```
$ git add -A
$ git commit -m 'topics show'
```



