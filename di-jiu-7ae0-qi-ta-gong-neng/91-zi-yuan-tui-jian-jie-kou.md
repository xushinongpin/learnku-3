## 资源推荐接口

Larabbs 的侧边栏有个推荐资源的功能，这一节我们来开发对应的接口。因为该功能已经在上一本教程中完成，我们只是为其写个接口，实现起来非常方便。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/AiovXWBCVk.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/AiovXWBCVk.png)

## 1. 添加路由

推荐资源是游客可以访问的接口

_routes/api.php_

```
.
.
.
// 某个用户的回复列表
$api->get('users/{user}/replies', 'RepliesController@userIndex')
    ->name('api.users.replies.index');
// 资源推荐
$api->get('links', 'LinksController@index')
    ->name('api.links.index');
.
.
.
```

## 2. 添加 Transformer

```
$ touch app/Transformers/LinkTransformer.php
```

_app/Transformers/LinkTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Link;
use League\Fractal\TransformerAbstract;

class LinkTransformer extends TransformerAbstract
{
    public function transform(Link $link)
    {
        return [
            'id' => $link->id,
            'title' => $link->title,
            'link' => $link->link,
        ];
    }
}
```

## 3. 添加 Controller

```
$ php artisan make:controller Api/LinksController
```

_app/Http/Controllers/Api/LinksController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\Link;
use Illuminate\Http\Request;
use App\Transformers\LinkTransformer;

class LinksController extends Controller
{
    public function index(Link $link)
    {
        $links = $link->getAllCached();

        return $this->response->collection($links, new LinkTransformer());
    }
}
```

Link 模型已经存在，getAllCached 会对结果进行缓存，可以看一下代码：

_app\Models\Link.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Cache;

class Link extends Model
{
    protected $fillable = ['title', 'link'];

    public $cache_key = 'larabbs_links';
    protected $cache_expire_in_minutes = 1440;

    public function getAllCached()
    {
        // 尝试从缓存中取出 cache_key 对应的数据。如果能取到，便直接返回数据。
        // 否则运行匿名函数中的代码来取出 links 表中所有的数据，返回的同时做了缓存。
        return Cache::remember($this->cache_key, $this->cache_expire_in_minutes, function(){
            return $this->all();
        });
    }
}
```

## 4. PostMan 调试

[初始化整个项目](https://learnku.com/courses/laravel-advance-training/5.5/785/install-larabbs)的时候，我们执行了`php artisan migrate --seed`命令，links 表中应该已经生成了部分假数据，如果你想填充更多的数据，可以单独执行：

```
$ php artisan db:seed --class=LinksTableSeeder
```

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/r2G7yUpXlu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/r2G7yUpXlu.png)

可以看到接口返回了推荐的资源数据，结果正确。

## 5. 保存接口

可以将接口保存在`其他接口`目录中：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/pXqN0llbRp.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/pXqN0llbRp.png)

## Git 版本控制

```
$ git add -A
$ git commit -m "links index"
```



