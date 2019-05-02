## 添加回复

在这个章节中，我们将开发话题回复功能，先参考下话题回复界面：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/30/1/LMjEAUQZCG.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/30/1/LMjEAUQZCG.png)

## 1. 增加路由

只有登录用户才可以进行回复

_routes/api.php_

```
.
.
.
$api->delete('topics/{topic}', 'TopicsController@destroy')
    ->name('api.topics.destroy');
// 发布回复
$api->post('topics/{topic}/replies', 'RepliesController@store')
    ->name('api.topics.replies.store');
.
.
.
```

回复一定属于某个话题，所以我们设计为`topics/{topic}/replies`，为某个话题添加回复，这样会让资源与资源的关系更加直观。

## 2. 增加 Request

创建 ReplyRequest：

```
$ php artisan make:request Api/ReplyRequest
```

如下修改：

_app/Http/Requests/Api/ReplyRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class ReplyRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'content' => 'required|min:2',
        ];
    }
}
```

大家应该注意到了一个不合理的地方，我们在重复地编写`authorize()`这个方法。Laravel 的`make:request`命令为我们生成的每一份表单验证类里都有一个`authorize()`，为了不违背 DRY 原则（Don't Repeat Yourself 不重复你自己），我们需要做重构。

增加 FormRequest

```
$ php artisan make:request Api/FormRequest
```

_app/Http/Requests/Api/FormRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest as BaseFormRequest;

class FormRequest extends BaseFormRequest
{
    public function authorize()
    {
        return true;
    }
}
```

再次修改`ReplyRequest.php`，删除`use Dingo\Api\Http\FormRequest`及`authorize`方法即可。

_app/Http/Requests/Api/ReplyRequest.php_

```
<?php

namespace App\Http\Requests\Api;

class ReplyRequest extends FormRequest
{
    public function rules()
    {
        return [
            'content' => 'required|min:2',
        ];
    }
}
```

有了`FormRequest`基类，我们的代码更加简洁了。大家可以自行修改其他的`Request`文件 。

## 3. 增加 Transformer

```
$ touch app/Transformers/ReplyTransformer.php
```

修改如下

_app/Transformers/ReplyTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Reply;
use League\Fractal\TransformerAbstract;

class ReplyTransformer extends TransformerAbstract
{
    public function transform(Reply $reply)
    {
        return [
            'id' => $reply->id,
            'user_id' => (int) $reply->user_id,
            'topic_id' => (int) $reply->topic_id,
            'content' => $reply->content,
            'created_at' => $reply->created_at->toDateTimeString(),
            'updated_at' => $reply->updated_at->toDateTimeString(),
        ];
    }
}
```

## 4. 增加 Controller

```
$ php artisan make:controller Api/RepliesController
```

修改文件

_app/Http/Controllers/Api/RepliesController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\Topic;
use App\Models\Reply;
use App\Http\Requests\Api\ReplyRequest;
use App\Transformers\ReplyTransformer;

class RepliesController extends Controller
{
    public function store(ReplyRequest $request, Topic $topic, Reply $reply)
    {
        $reply->content = $request->content;
        $reply->topic_id = $topic->id;
        $reply->user_id = $this->user()->id;
        $reply->save();

        return $this->response->item($reply, new ReplyTransformer())
            ->setStatusCode(201);
    }
}
```

## 5. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/LQgxKCUt82.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/LQgxKCUt82.png)

调试成功，状态码为 201， 响应 body 为回复数据。保存接口，新建话题回复目录。

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/3NhPZaxoc2.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/3NhPZaxoc2.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'replies store'
```



