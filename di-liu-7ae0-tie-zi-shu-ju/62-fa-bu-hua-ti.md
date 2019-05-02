## 发布话题

参考以下发布话题的界面：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/ACj892ryDW.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/22/3995/ACj892ryDW.png)

需要填写话题标题，话题内容，选择话题分类。如果这个话题内容，可以上传图片，插入 markdown 图片链接，则可以调用我们上一节写的`添加图片`接口，type 为 topic，将返回的图片 path 插入到话题内容中。

## 1. 增加路由

只有登录用户才可以发布话题，添加路由。

_routes/api.php_

```
.
.
.
// 图片资源
$api->post('images', 'ImagesController@store')
    ->name('api.images.store');
// 发布话题
$api->post('topics', 'TopicsController@store')
    ->name('api.topics.store');
.
.
.
```

## 2. 增加 Request

创建 TopicRequest：

```
$ php artisan make:request Api/TopicRequest
```

如下修改：

_app/Http/Requests/Api/TopicRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class TopicRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        return [
            'title' => 'required|string',
            'body' => 'required|string',
            'category_id' => 'required|exists:categories,id',
        ];
    }

    public function attributes()
    {
        return [
            'title' => '标题',
            'body' => '话题内容',
            'category_id' => '分类',
        ];
    }
}
```

## 3. 增加 Transformer

```
$ touch app/Transformers/TopicTransformer.php
```

修改如下

_app/Transformers/TopicTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Topic;
use League\Fractal\TransformerAbstract;

class TopicTransformer extends TransformerAbstract
{
    public function transform(Topic $topic)
    {
        return [
            'id' => $topic->id,
            'title' => $topic->title,
            'body' => $topic->body,
            'user_id' => (int) $topic->user_id,
            'category_id' => (int) $topic->category_id,
            'reply_count' => (int) $topic->reply_count,
            'view_count' => (int) $topic->view_count,
            'last_reply_user_id' => (int) $topic->last_reply_user_id,
            'excerpt' => $topic->excerpt,
            'slug' => $topic->slug,
            'created_at' => $topic->created_at->toDateTimeString(),
            'updated_at' => $topic->updated_at->toDateTimeString(),
        ];
    }
}
```

## 4. 增加 Controller

```
$ php artisan make:controller Api/TopicsController
```

修改文件

_app/Http/Controllers/Api/TopicsController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\Topic;
use Illuminate\Http\Request;
use App\Transformers\TopicTransformer;
use App\Http\Requests\Api\TopicRequest;

class TopicsController extends Controller
{
    public function store(TopicRequest $request, Topic $topic)
    {
        $topic->fill($request->all());
        $topic->user_id = $this->user()->id;
        $topic->save();

        return $this->response->item($topic, new TopicTransformer())
            ->setStatusCode(201);
    }
}
```

## 5. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/mNCYZTSubs.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/27/3995/mNCYZTSubs.png)

调试成功，保存接口，新建话题目录。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/56PEMwatHY.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/56PEMwatHY.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'topics store'
```



