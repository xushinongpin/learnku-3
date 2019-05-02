## 用户角色列表

这个章节我们来处理用户角色的部分，客户端有可能在某些地方显示用户角色信息，例如某个用户的个人页面里，显示出用户是站长，还是管理员。用户的角色信息没必要再单独请求接口了，我们可以通过 Include 机制实现。

## 1. 增加 Transformer

```
$ touch app/Transformers/RoleTransformer.php
```

_app/Transformers/RoleTransformer.php_

```
<?php

namespace App\Transformers;

use Spatie\Permission\Models\Role;
use League\Fractal\TransformerAbstract;

class RoleTransformer extends TransformerAbstract
{
    public function transform(Role $role)
    {
        return [
            'id' => $role->id,
            'name' => $role->name,
        ];
    }
}
```

## 2. 修改 UserTransformer

_app/Transformers/UserTransformer.php_

```
<?php

namespace App\Transformers;

use App\Models\User;
use League\Fractal\TransformerAbstract;

class UserTransformer extends TransformerAbstract
{
    protected $availableIncludes = ['roles'];

.
.
.
    public function includeRoles(User $user)
    {
        return $this->collection($user->roles, new RoleTransformer());
    }
}
```

用户与角色的关系是一对多的，我们通过`$this->collection`返回用户权限。

## 3. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/4gTOKZRnEJ.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/08/3995/4gTOKZRnEJ.png)

访问某个用户发布的话题列表接口，修改参数为`include=user.roles,category`，回忆一下 Include 机制，这里请求的是`话题资源`，`话题的发布用户信息`，`话题发布用户的角色资源`，`话题的分类资源`。

可以看到结果中，用户信息数据中嵌套 roles 资源。任何一个需要用户资源的地方都可以通过`include=roles`返回用户对应的角色，是不是非常的灵活方便。

## 代码版本控制

```
$ git add -A
$ git commit -m 'include user roles'
```



