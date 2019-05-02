## 删除话题

本章节我们将开发删除话题的 API 功能。

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
$api->delete('topics/{topic}', 'TopicsController@destroy')
    ->name('api.topics.destroy');
.
.
.
```

## 2. 修改 Controller

```
.
.
.
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

注意这里我们使用的是`destroy`的权限控制。

## 3. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/n6hmW94Bwk.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/23/3995/n6hmW94Bwk.png)

删除成功

## 提交代码

```
$ git add -A
$ git commit -m 'topic delete api'
```



