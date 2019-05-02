## 第三方黑盒测试

除了单元测试以及集成测试之外，还可以利用第三方工具，这一节我们将学习如何利用 PostMan 进行第三方黑盒测试，这也是我们最推崇的测试方案。

第三方黑盒测试的好处是可以最大程度测试**整套系统**，我们的 API 接口，从 PHP 代码解析开始，以下涉及因素都会影响到接口的可用性：

1. 软件代码级别的错误；
2. 程序使用的第三方软件发生错误，如：Redis 缓存和队列系统、MySQL 数据库等；
3. API 服务器上的系统软件，如 Nginx、Cron 等；
4. API 服务器上的物理问题，如硬盘坏了；
5. 域名解析问题，如 DNS 解析出错；

代码级别的自动化测试，能测试的范围有限。而第三方黑盒测试，模拟的是真实用户的请求，将 API 服务器看成**完全的系统**，系统里任何一个部件坏了，都能被检测出来。并且这种测试方法与服务器端环境彻底解耦，后期维护成本较低。

## 分享接口数据

PostMan 支持我们导出保存的接口，在团队协作中，后端工程师可以方便的将 PostMan 的接口数据分享给客户端工程师，客户端工程师可以自行测试接口，真实的模拟请求。

### 导出 Collection

[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/Ied9wxlxvb.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/Ied9wxlxvb.png)

有多种导出格式可选，我们选择 PostMan 推荐的`Collection v2.1`。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/lNwN2t8hHs.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/lNwN2t8hHs.png)

选择导出后，就得到了`Larabbs`的接口文件，文件名类似`Larabbs.postman_collection.json`。

### 导出环境变量

除了接口数据外，我们还定义了一些环境变量，例如`{{host}}`，`{{jwt_user1}}`等，我们也需要将其导出，分享给他人，这样才是一个完整的环境。

点击右上角的设置，选择`Manage Environments`。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/EMpRFogT6q.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/EMpRFogT6q.png)

点击对应环境后面的下载即可。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/OiZ3MmHakG.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/OiZ3MmHakG.png)

下载的文件名类似`larabbs-local.postman_environment.json`。

这里要注意我们现在的环境`larabbs-local`是我们本地的环境，为了方便客户端工程师使用，可以在某个线上可访问的测试服务器搭建完整的 Larabbs 环境，增加测试服务器的环境`larabbs-test`，设置测试环境的环境变量`{{host}}`，`{{jwt_user1}}`等，这样分享出去的环境，别人可以直接使用。

### 导入 Collection 及环境

我们将导出的`Larabbs.postman_collection.json`和`larabbs-local.postman_environment.json`两个文件分享给客户端的工程师。

点击左上角的`Import`即可导入 Collection 文件。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/uWqM69iuIA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/uWqM69iuIA.png)

在`Manage Environments`中点击`Import`即可导入环境变量。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/k3BI06RtYn.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/k3BI06RtYn.png)

导入成功后，我们就可以直接对接口进行真实调试。

## PostMan 自动化测试

PostMan 为我们提供了自动化测试的功能，类似于 Laravel 的接口测试，PostMan 可以请求接口，并且断言响应结果和响应数据，接下来我们以`发布话题`和`话题列表`两个接口为例，进行自动化测试。

### 测试发布话题

打开`发布话题`接口，可以看到有个`Tests`的选项卡，点击该选项卡会出现一个空白区域，在这里我们可以添加一些断言，判断请求结果。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/fyTeKGh5cA.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/fyTeKGh5cA.png)

填入如下内容：

```
pm.test("响应状态码正确", function () { 
    pm.response.to.have.status(201);
});

pm.test("接口响应数据正确", function () { 
    pm.expect(pm.response.text()).to.include("id");
    pm.expect(pm.response.text()).to.include("title");
    pm.expect(pm.response.text()).to.include("body");
    pm.expect(pm.response.text()).to.include("user_id");
    pm.expect(pm.response.text()).to.include("category_id");
});
```

PostMan 为我们提供了`pm.test`方法，相当于一个测试用例，第一个参数是执行正确后的提示文字，第二个参数是个闭包，执行我们的断言。

第一个测试用户我们判断响应的状态码，`pm.response.to.have.status(201);`断言响应结果的状态码是`201`。

第二个测试用户，我们判断响应数据，通过`pm.expect(pm.response.text()).to.include("");`断言响应数据中一定会包含某个字段。

点击`Send`进行调试。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/7ar0pAgMnx.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/7ar0pAgMnx.png)

切换到`Test Results`，可以看到两个测试用例均通过了。

### 测试话题列表

同样的，我们为`话题列表`接口增加测试用例：

```
// example using response assertions
pm.test("响应状态码正确", function () { 
    pm.response.to.have.status(200);
});

pm.test("接口响应数据正确", function () { 
    pm.expect(pm.response.text()).to.include("data");
    pm.expect(pm.response.text()).to.include("meta");
});
```

同样增加了两个测试用例，断言响应状态码为`200`，断言响应数据中包含`data`和`meta`。

点击`Send`进行调试。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/QbjeNDGaJw.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/QbjeNDGaJw.png)

测试通过。

### 批量测试

每个接口都完成测试用户后，我们就可以通过 PostMan 的测试工具进行自动化测试了。

[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/wviLpuOBdS.gif "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/wviLpuOBdS.gif)

点击 PostMan 左上角的`Runner`，我们可以看到 PostMan 的自动化测试界面，我们可以选择测试整个项目，或者测试某个目录。这里我们选择`话题`目录，选择`larabbs-local`环境，执行测试。

我们可以看到 PostMan 依次请求了`话题`目录下的所有接口，因为我们为`发布话题`和`话题列表`添加了测试用例和断言，所以看到这两个接口的测试用户均已通过。  
[![](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/toptSClMiD.png "file")](https://iocaffcdn.phphub.org/uploads/images/201802/26/3995/toptSClMiD.png)

我们可以为所有的接口增加测试用例，这样当接口升级之后，可以方便的通过 PostMan 的自动化测试工具进行测试，快速定位不符合预期的接口。

