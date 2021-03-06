## 手机注册

无论是 App 还是网站，手机注册已经成为商业项目的最基本功能，下面我们通过『知乎 App』来拆解一下手机注册的流程。

## 1. 注册示例

首先来看一下『知乎 App』的注册流程，注意知乎『未注册手机验证后自动登录』，登录与注册流程相同。

[![](https://iocaffcdn.phphub.org/uploads/images/201712/27/6351/pNlEHPCu4Y.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/27/6351/pNlEHPCu4Y.png)

1. 输入手机号，点击登录
2. 输入正确的图片验证码，点击验证，知乎会发送验证码至刚刚填写的手机
3. 提交正确的短信验证码后，注册完成

## 2. 流程分解

拆解一下注册流程，如图所示

[![](https://iocaffcdn.phphub.org/uploads/images/201801/24/1/VdljFNRdrW.png "file")](https://iocaffcdn.phphub.org/uploads/images/201801/24/1/VdljFNRdrW.png)

流程详解：

1. 用户输入手机号，请求图片验证码接口；
2. 服务器返回图片验证码；
3. 使用正确的图片验证码，请求短信验证码；
4. 服务器调用短信运营商的接口，发送短信至用户手机；
5. 通过正确的短信验证码，请求用户注册接口；
6. 完成注册流程。

上面的流程中，我们需要下面三个资源

* captchas —— 图片验证码
* verificationCodes —— 短信验证码
* users —— 用户

对应着三个资源，我们设计出对应的三个接口

* POST api/captchas 创建图片验证码
* POST api/verificationCodes 发送短信验证码
* POST api/users 用户注册

> 关于注册，可能常见的注册接口会设计为`api/signUp`或`api/register`，这样的设计没什么问题，不过回忆一下我们的 RESTful 原则，资源尽量不要用动词，那么注册这件事，实质上是在服务器上创建了用户资源。所以设计成`POST api/users`更为合适。

我们可以像知乎这类 APP 一样，用户手机注册后，再跳转到用户信息设置页面，更新用户名，密码等信息。但是结合 Larabbs 已有的注册流程，我们可以在用户注册流程的最后一步，要求用户提交用户名，密码以及短信验证码来完成注册。

整理下来，我们的注册流程如下：

* 用户输入手机号
* 通过手机号请求图片验证码，显示给用户
* 用户输入正确的图片验证码，服务器发送验证码至用户手机
* 用户填写，姓名，密码，及正确的短信验证码，完成注册

接下来我们将一步步讲解。

