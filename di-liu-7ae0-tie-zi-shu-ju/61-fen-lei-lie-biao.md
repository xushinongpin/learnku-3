## 分类列表

在这个章节中，我们将开发帖子分类的获取接口，方便我们在下一章节『发布话题』中使用。

## 1. 添加路由

回忆一下 Larabbs 的模型关系，每一个话题一定会属于某个分类，所以发布话题的时候，我们会从列表中选择一个分类，APP 首页也会显示出来不同分类便签，便于切换。

新增`分类列表`接口，分类列表是游客可访问的接口，不需要 token 验证。

_routes/api.php_

```
.
.
.
// 游客可以访问的接口
$api->get('categories', 'CategoriesController@index')
    ->name('api.categories.index');

// 需要 token 验证的接口
.
.
.
```

## 2. 创建 Transformer

```
$ touch app/Transformers/CategoryTransformer.php
```

修改如下

_app/Transformers/CategoryTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Category;
use League\Fractal\TransformerAbstract;

class CategoryTransformer extends TransformerAbstract
{
    public function transform(Category $category)
    {
        return [
            'id' => $category->id,
            'name' => $category->name,
            'description' => $category->description,
        ];
    }
}
```

## 3. 创建 controller

```
php artisan make:controller Api/CategoriesController
```

修改如下

_app/Http/Controllers/Api/CategoriesController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\Category;
use Illuminate\Http\Request;
use App\Transformers\CategoryTransformer;

class CategoriesController extends Controller
{
    public function index()
    {
        return $this->response->collection(Category::all(), new CategoryTransformer());
    }
}
```

分类数据是集合，所以我们使用`$this->response->collection`返回数据。

## 4. 测试一下

使用 PostMan 调试一下接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/Hww1mqfZL5.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/Hww1mqfZL5.png)

保存接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/wQTEGCEPMY.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/wQTEGCEPMY.png)

也许你也注意到了，实现数据读取的接口非常的方便，大部分时候遵循以下流程：

1. 增加路由
2. 创建 transformer
3. controller 处理数据，使用 transformer 转换后返回

## 提交代码

```
$ git add -A
$ git commit -m 'categories index'
```



