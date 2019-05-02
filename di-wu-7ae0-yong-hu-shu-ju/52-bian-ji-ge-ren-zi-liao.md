## 编辑个人资料

在本章节中，我们将开发用户的编辑接口，允许用户对自己的用户名、邮箱、简介和头像进行修改。

## 数据的提交方式

HTTP 提交数据有两种方式

* application/x-www-form-urlencoded \(默认值\)
* multipart/form-data

大家应该记得，form 表单提交文件的时候，需要增加`enctype="multipart/form-data"`，才能正确传输文件，因为默认的`enctype`是`enctype="application/x-www-form-urlencoded"`。

需要明确的是，只有当 POST 配合`multipart/form-data`时才能正确传输文件。

## 图片资源

我们设计 API 时，修改相关的 API 通常会使用`put`或`patch`，但是因为要修改用户头像，又必须使用 POST 的`multipart/form-data`，难道所有涉及到文件的接口我们都必须设计为 POST 吗？  
其实一般有关文件上传的接口，我们一般会设计为两个，例如 Larabbs 的业务，我们可以设计一个图片资源 ——images，修改头像的逻辑为

* 调用 POST api/images 在服务器创建图片资源
* 通过图片资源的
  `id`
  或路径，请求修改头像接口

我们首先可以添加一个图片资源

```
$ php artisan make:migration create_images_table --create=images
```

修改文件，_database/migrations/&lt; your\_date &gt;\_create\_images\_table.php_注意替换文件日期

```
<?php

use Illuminate\Support\Facades\Schema;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Database\Migrations\Migration;

class CreateImagesTable extends Migration
{
    public function up()
    {
        Schema::create('images', function (Blueprint $table) {
            $table->increments('id');
            $table->integer('user_id')->index();
            $table->string('type')->index();
            $table->string('path');
            $table->timestamps();
        });
    }

    public function down()
    {
        Schema::dropIfExists('images');
    }
}
```

images 表记录了用户 id，图片路径，以及图片类型。图片类型有两种 'avatar' 和 'topic'，分别用于用户头像以及话题中的图片。记录图片类型是因为不同类型的图片有不同的尺寸，以及不同的文件目录，修改个人头像所使用的`image`必须为`avatar`类型。  
执行 migrate

```
$ php artisan migrate
```

添加路由

_routes/api.php_

```
.
.
.
// 需要 token 验证的接口
$api->group(['middleware' => 'api.auth'], function($api) {
    // 当前登录用户信息
    $api->get('user', 'UsersController@me')
        ->name('api.user.show');
    // 图片资源
    $api->post('images', 'ImagesController@store')
        ->name('api.images.store');
});
.
.
.
```

创建模型，request 以及 controller。

```
$ php artisan make:model Models/Image
$ php artisan make:request Api/ImageRequest
$ php artisan make:controller Api/ImagesController
```

修改如下

_app\Models\Image.php_

```
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Image extends Model
{
    protected $fillable = ['type', 'path'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

_app/Http/Requests/Api/ImageRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class ImageRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {

        $rules = [
            'type' => 'required|string|in:avatar,topic',
        ];

        if ($this->type == 'avatar') {
            $rules['image'] = 'required|mimes:jpeg,bmp,png,gif|dimensions:min_width=200,min_height=200';
        } else {
            $rules['image'] = 'required|mimes:jpeg,bmp,png,gif';
        }

        return $rules;
    }

      public function messages()
      {
          return [
              'image.dimensions' => '图片的清晰度不够，宽和高需要 200px 以上',
          ];
      }
}
```

参考 Larabbs 网页部分的头像处理，我们要求如果是头像类型的图片资源，宽和高必须在 200px 以上。

创建 ImageTransformer

```
$ touch app/Transformers/ImageTransformer.php
```

_app/Transformers/ImageTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\Image;
use League\Fractal\TransformerAbstract;

class ImageTransformer extends TransformerAbstract
{
    public function transform(Image $image)
    {
        return [
            'id' => $image->id,
            'user_id' => $image->user_id,
            'type' => $image->type,
            'path' => $image->path,
            'created_at' => $image->created_at->toDateTimeString(),
            'updated_at' => $image->updated_at->toDateTimeString(),
        ];
    }
}
```

_app/Http/Controllers/Api/ImagesController.php_

```
<?php

namespace App\Http\Controllers\Api;

use App\Models\Image;
use Illuminate\Http\Request;
use App\Handlers\ImageUploadHandler;
use App\Transformers\ImageTransformer;
use App\Http\Requests\Api\ImageRequest;

class ImagesController extends Controller
{
    public function store(ImageRequest $request, ImageUploadHandler $uploader, Image $image)
    {
        $user = $this->user();

        $size = $request->type == 'avatar' ? 362 : 1024;
        $result = $uploader->save($request->image, str_plural($request->type), $user->id, $size);

        $image->path = $result['path'];
        $image->type = $request->type;
        $image->user_id = $user->id;
        $image->save();

        return $this->response->item($image, new ImageTransformer())->setStatusCode(201);
    }
}
```

两种图片类型，头像和话题，会利用`ImageUploadHandler`进行存储和裁剪。使用 PostMan 测试一下图片接口。

[![](https://iocaffcdn.phphub.org/uploads/images/201808/08/3995/3KkChTgwQc.png?imageView2/2/w/1240/h/0 "file")](https://iocaffcdn.phphub.org/uploads/images/201808/08/3995/3KkChTgwQc.png?imageView2/2/w/1240/h/0)

记得保存接口，可以新建一个图片目录  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/21/3995/gIfVj70j88.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/21/3995/gIfVj70j88.png)

### 编辑个人资料接口

添加编辑个人资料接口

_routes/api.php_

```
.
.
.
// 编辑登录用户信息
$api->patch('user', 'UsersController@update')
    ->name('api.user.update');
// 图片资源
$api->post('images', 'ImagesController@store')
    ->name('api.images.store');
.
.
.
```

注意这里使用的方法是 patch，patch 与 put 的区别为

* put 替换某个资源，需提供完整的资源信息
* patch 部分修改资源，提供部分资源信息

修改 UserRequest

_app/Http/Requests/Api/UserRequest.php_

```
<?php

namespace App\Http\Requests\Api;

use Dingo\Api\Http\FormRequest;

class UserRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }

    public function rules()
    {
        switch($this->method()) {
            case 'POST':
                return [
                    'name' => 'between:3,25|regex:/^[A-Za-z0-9\-\_]+$/|unique:users,name',
                    'password' => 'required|string|min:6',
                    'verification_key' => 'required|string',
                    'verification_code' => 'required|string',
                ];
                break;
            case 'PATCH':
                $userId = \Auth::guard('api')->id();
                return [
                    'name' => 'between:3,25|regex:/^[A-Za-z0-9\-\_]+$/|unique:users,name,' .$userId,
                    'email' => 'email',
                    'introduction' => 'max:80',
                    'avatar_image_id' => 'exists:images,id,type,avatar,user_id,'.$userId,
                ];
                break;
        }
    }

    public function attributes()
    {
        return [
            'verification_key' => '短信验证码 key',
            'verification_code' => '短信验证码',
            'introduction' => '个人简介',
        ];
    }

    public function messages()
    {
        return [
            'name.unique' => '用户名已被占用，请重新填写',
            'name.regex' => '用户名只支持英文、数字、横杆和下划线。',
            'name.between' => '用户名必须介于 3 - 25 个字符之间。',
            'name.required' => '用户名不能为空。',
        ];
    }
}
```

修改头像时，我们先创建 avatar 类型的图片资源，然后提交`avatar_image_id`即可。

_app/Http/Controllers/Api/UsersController.php_

```
.
.
.
use App\Models\Image;
.
.
.
public function update(UserRequest $request)
{
    $user = $this->user();

    $attributes = $request->only(['name', 'email', 'introduction']);

    if ($request->avatar_image_id) {
        $image = Image::find($request->avatar_image_id);

        $attributes['avatar'] = $image->path;
    }
    $user->update($attributes);

    return $this->response->item($user, new UserTransformer());
}
.
.
.
```

客户端提交什么，服务器就修改对应的资源。使用 PostMan 调用接口，别忘了设置`Bearer Token`，之前教程中我们增加的几个 token 变量可以直接使用，例如`{{jwt_user1}}`，指定你要修改哪个用户的资料。  
[![](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/TW6aFDbgX1.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/22/3995/TW6aFDbgX1.png)

注意这里需要使用`x-www-form-urlencoded`

## 代码版本控制

```
$ git add -A
$ git commit -m 'user update'
```



