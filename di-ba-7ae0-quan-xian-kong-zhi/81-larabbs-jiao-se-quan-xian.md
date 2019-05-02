## Larabbs 角色权限

这一节我们主要来实现权限相关的功能，先来了解一下 Larabbs 中的角色权限，之前购买过第二本教程的用户可以复习一下[第七章。角色权限和管理后台](https://learnku.com/courses/laravel-intermediate-training/5.5/672/multi-role-user-rights)，没有购买过的用户只需要认真阅读本章节即可。

### 数据表结构

我们使用了[spatie/laravel-permission](https://github.com/spatie/laravel-permission)这个扩展包

下图是 laravel-permission 的数据库表结构：

[![](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/35YmbLHqjC.png "file")](https://iocaffcdn.phphub.org/uploads/images/201710/25/1/35YmbLHqjC.png)

数据表各自的作用：

* roles —— 角色的模型表；
* permissions —— 权限的模型表；
* model\_has\_roles —— 模型与角色的关联表，用户拥有什么角色在此表中定义，一个用户能拥有多个角色；
* role\_has\_permissions —— 角色拥有的权限关联表，如管理员拥有查看后台的权限都是在此表定义，一个角色能拥有多个权限；
* model\_has\_permissions —— 模型与权限关联表，一个模型能拥有多个权限。

从最后一张表中可以看出，laravel-permission 允许用户跳过角色，直接拥有权限。不过在本项目中，为了方便管理，我们设定：

> 用户只能通过角色来获取到权限，用户不单独拥有权限。例如：用户 Summer 必须是『站长』角色，才能行使『用户管理』权限。

### 用户身份

在我们的 LaraBBS 项目里有以下几种用户身份：

* **游客**
  可以随便浏览页面，但是无法发布内容；
* **用户**
  能够发布内容，却只能管理自己的内容；
* **管理员**
  可以管理所有用户的内容，然而不能管理用户；
* **站长**
  拥有最高权限，可以管理所有内容，包括用户。

### 用户权限

根据以上设定，需要设置三个权限

* **manage\_contents**
  维护社区的内容
* **manage\_users**
  修改用户密码、删除用户等
* **manage\_settings**
  可以在后台管理站点相关的设置，如 SEO 设置、联系邮箱等

### 用户角色

设置两个角色

* **管理员 Maintainer**
  拥有 manage\_contents 权限
* **站长 Founder**
  拥有 manage\_contents， manage\_users， manage\_settings 权限

游客和用户没有角色；将 1 号用户指派为『站长』拥有所有权限；将 2 号用户设置为管理员，拥有`manage_contents`权限，帮助维护社区内容。

## 客户端判断逻辑

客户端（iOS 或者 Android）的界面显示上，需要根据用户的权限以及用户与资源的关系，显示不同的内容。以下将以 Web 端为例子来讲解。

比如拥有`manage_contents`管理内容权限的用户，可以修改所有的话题和评论。

以某个普通用户登录为例，可以看到该用户只能编辑、删除自己发布的话题。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/bmbOAGzHRu.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/bmbOAGzHRu.png)

[![](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/YF6uzRYkLn.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/YF6uzRYkLn.png)

再以 1 号用户 Summer 登录，查看其它用户发布的帖子，依然可以编辑、删除。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/rAPvOWCYn6.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/07/3995/rAPvOWCYn6.png)

网页中，我们是通过 Blade 的 can 命令进行判断的

```
@can('update', $topic)
```

看一下判断逻辑，在`App\Policies\TopicPolicy`中

```
<?php

namespace App\Policies;

use App\Models\User;
use App\Models\Topic;

class TopicPolicy extends Policy
{
    public function update(User $user, Topic $topic)
    {
        return $user->isAuthorOf($topic);
    }

    public function destroy(User $user, Topic $topic)
    {
        return $user->isAuthorOf($topic);
    }
}
```

`App\Policies\Policy`基类中的`before`方法会优先执行

```
public function before($user, $ability)
{
    // 如果用户拥有管理内容的权限的话，即授权通过
    if ($user->can('manage_contents')) {
        return true;
    }
}
```

最终的判断逻辑是，如果用户拥有`manage_contents`权限，或者是话题的发布者，可以编辑某个话题。

对于客户端 APP 来说，执行的逻辑是相同的，只不过用户的权限是通过接口请求来的。下一章节中我们将编写『权限数据接口』，客户端在获取到权限数据后，即可在客户端完成权限的控制。

