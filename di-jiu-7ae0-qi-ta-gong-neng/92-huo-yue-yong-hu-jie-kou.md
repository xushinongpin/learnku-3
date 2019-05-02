## 活跃用户接口

Larabbs 的边栏会显示活跃用户，这一节我们会为这个功能开发接口：

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/EsuqSDjEmO.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/EsuqSDjEmO.png)

## 1. 生成测试数据

如果你的 Homestead 的 Cron 没有配置，你可能会发现，你本地的 Larabbs 并没有显示活跃用户

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/gfM9ecxTll.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/gfM9ecxTll.png)

这是因为活跃用户数据，是 artisan 命令生成的，如果 Cron 配置正确，会每个小时执行一次，计算出活跃用户。

### 算法

算法参考 Laravel China 社区：[关于「活跃用户」的算法](https://learnku.com/laravel/t/2902/algorithm-for-active-users)。

系统**每一个小时**计算一次，统计**最近 7 天**所有用户发的**帖子数**和**评论数**，用户每发一个帖子则得**4**分，每发一个回复得**1**分，计算出所有人的『得分』后再倒序，排名前八的用户将会显示在「活跃用户」列表里。

假设用户 A 在 7 天内发了 10 篇帖子，发了 5 条评论，则其得分为

```
10 * 4 + 5 * 1 = 45 
```

### 执行 Artisan 命令

由于算法计算的是 7 天内的用户发帖和评论数据，我们可以在填充一些帖子数据

```
$ php artisan db:seed --class=TopicsTableSeeder
```

执行生成活跃用户命令

```
$ php artisan larabbs:calculate-active-user
```

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/2LAdnsd6qs.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/2LAdnsd6qs.png)

再次访问 larabbs.test，应该能看到活跃用户数据了

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/EsuqSDjEmO.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/EsuqSDjEmO.png)

## 2. 添加路由

_routes/api.php_

```
.
.
.
// 资源推荐
$api->get('links', 'LinksController@index')
    ->name('api.links.index');
// 活跃用户
$api->get('actived/users', 'UsersController@activedIndex')
    ->name('api.actived.users.index');
.
.
.
```

该接口也是游客可以访问的。

## 3. 修改 Controller

_app/Http/Controllers/Api/UsersController.php_

```
.
.
.
public function activedIndex(User $user)
{
    return $this->response->collection($user->getActiveUsers(), new UserTransformer());
}
.
.
.
```

可以看到代码非常的简单，直接调用`$user->getActiveUsers()`即可，活跃用户的逻辑代码放置于在 Trait ——`app/Models/Traits/ActiveUserHelper.php`中，算法的讲解，代码里有注释，这里便不再做过多讲解，购买过第二本教程的用户可以复习一下[8.1. 边栏活跃用户](https://learnku.com/courses/laravel-intermediate-training/5.5/671/active-users#2.%E7%BC%96%E5%86%99%E9%80%BB%E8%BE%91%E4%BB%A3%E7%A0%81)这一节。

## 4. PostMan 调试

[![](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/XQDUkGtWiR.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/12/3995/XQDUkGtWiR.png)

## 代码版本控制

```
$ git add -A
$ git commit -m 'actived users'
```



