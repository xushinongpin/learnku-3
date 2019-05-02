## Passport

JWT 是一种比较便捷的用户认证方式，也是我们推荐的方式。Laravel 也为我们提供了一套 API 授权认证方案 ——Passport，这一章我们将学习如何使用 Passport 。

## OAuth2 授权码模式

Passport 是对 OAuth2 的封装，可以快速方便的在服务端搭建完整的 OAuth2。回忆一下 OAuth2，其实我们在第四章[微信登录流程讲解](https://learnku.com/courses/laravel-advance-training/5.5/796/process-explanation)中已经介绍过 OAuth2 的授权码模式。

再次回忆一下授权码模式，主要应用在平台给第三方的应用进行用户授权，简单梳理一下流程，假设我们有 Larabbs 开放平台，第三方的应用 Larabbs-Blog，希望用户可以直接通过 Larabbs 中的账户密码登录，并且获取用户在 Larabbs 中发布的话题展示出来。

1. `Larabbs`
   为
   `Larabbs-Blog`
   创建客户端应用，并且分配
   `client_id`
   和
   `client_secret`
   给
   `Larabbs-Blog`
   ；
2. 用户在
   `Larabbs-Blog`
   的登录界面，点击
   `使用 Larabbs 登录`
   ；
3. `Larabbs-Blog`
   跳转到
   `Larabbs`
   的用户授权页面，用户在输入用户名密码登录，授权
   `Larabbs-Blog`
   可以获取
   `Larabbs`
   中的信息；
4. `Larabbs`
   跳转回
   `Larabbs-Blog`
   ，并返回授权码；
5. `Larabbs-Blog`
   通过
   `client_secret`
   以及授权码获取到
   `access_token`
   ，然后通过
   `access_token`
   获取
   `Larabbs`
   中的用户以及话题信息。

上面是一个完整的 OAuth2 授权流程，可以看到使用授权码模式，`Larabbs-Blog`没有任何机会获取到用户的密码，而且只有用户在`Larabbs`中同意授权以后，`Larabbs-Blog`才能获取用户的信息，保证了用户数据的安全。

授权码模式是完整基础的 OAuth2 流程，我们通常说的第三方登录都是指授权码模式。

## OAuth2 密码模式

但是对于我们自己的客户端，比如 Larabbs 的 IOS 客户端，中间的授权码流程就显得有些多余，这时 OAuth2 的另一个模式 ——[密码模式](https://learnku.com/docs/laravel/5.5/passport#password-grant-tokens)，就很好的解决了这个问题。

对于我们自己的客户端，用户应该直接在客户端中输入用户名和密码，客户端直接通过用户数据的用户名和密码获取`access_token`即可。

密码模式流程如下：

* 用户在客户端输入用户名和密码；
* 客户端提交用户名，密码，
  `client_id`
  和
  `client_secret`
  到服务器；
* 服务器直接返回
  `access_token`
  ；

可以看到密码模式的流程非常简洁，我们可以方便的向自己的客户端发出访问令牌，而不需要遍历整个 OAuth2 授权代码重定向流程。

下面的课程，我们会使用 OAuth2 密码模式，替换现在的 JWT，为接口提供认证。

