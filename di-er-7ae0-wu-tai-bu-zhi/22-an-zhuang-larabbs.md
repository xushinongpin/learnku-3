## 安装 LaraBBS

[LaraBBS](https://github.com/summerblue/larabbs)是在[Laravel 教程 - Web 开发实战进阶 \(Laravel 5.7\)](https://learnku.com/courses/laravel-intermediate-training-5.7)中一步一步搭建的论坛系统，接下来我们基于这套论坛系统，完成 API 部分的训练。

> 学习过[课程二](https://learnku.com/courses/laravel-intermediate-training-5.7)的同学，如果你已经部署好了 LaraBBS 的开发环境，可以跳过这一步。

## 做好准备

由于我们接下来的开发都会在 Homestead 上进行，因此，在开始本章教程之前，请保证你的 Homestead 虚拟机已成功开启并登录。使用下面命令来启动和登录 Homestead：

```
> cd ~/Homestead && vagrant up
> vagrant ssh
```

在虚拟机中进入 Code 文件夹：

```
$ cd 
~
/
Code
```

> 注意：本书中因为虚拟机的存在，我们会有两个运行命令行的环境，一个是主机，另一个是 Homestead 虚拟机。我们会在命令的前面使用『命令行提示符』来区分主机和 Homestead。请记住以 &gt; 开头的命令是运行在主机里，$ 开头的命令是运行在 Homestead 虚拟机里。详见 写作约定 - 命令行提示符。

## Composer 加速

在创建项目之前，我们先在虚拟机中运行以下命令来实现 Composer 安装加速 ：

```
$ composer config 
-
g repo
.
packagist composer https
:
//packagist.phpcomposer.com
```

## 安装 LaraBBS 应用

如果你已经购买并完成了上一本教程，那么你的 github 上应该有一个自己的 larabbs 项目，比如`git@github.com:<username>/larabbs.git`，或者你可以 fork 一份，总之，你需要在 github 上有一个自己的 larabbs 项目，这样有助于你进行接下来的练习。

[![](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/Dz30nMWgEB.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/Dz30nMWgEB.png!large)

假设你已经准备好了自己的 larabbs 的项目，注意请把下面的`<username>`替换为你的用户名。

```
$ cd 
~
/
Code
$ git 
clone
 git@github
.
com
:
<
username
>
/
larabbs
.
git
$ cd larabbs
$ composer install
$ cp 
.
env
.
example 
.
env
```

[![](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/h5KhYUgM5d.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/h5KhYUgM5d.png)  
中间省略掉安装细节...  
[![](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/6GdoK8D3mw.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/6GdoK8D3mw.png)

## 修改 hosts

每个 Laravel 项目创建完成后的第一步，即是对 Homestead 进行配置，让应用能在 Homestead 的开发环境上跑起来。

为了方便记忆，一般我们都会将 IP 映射为域名，我们能够通过设置 hosts 文件来指定 IP 与域名之间的映射关系，由于我们在 Homestead 上默认使用`192.168.10.10`来作为虚拟机的 IP 的地址，因此我们需要在系统的`hosts`文件中将域名指向该 IP 上。

Mac 下打开 Hosts 文件：

```
>
 subl 
/
etc
/
hosts
```

Windows 下打开 Hosts 文件：

```
>
 subl 
C
:
/
Windows
/
System32
/
Drivers
/
etc
/
hosts
```

> Windows 下，如果你没有集成 subl 命令的话， 请使用编辑器直接打开文件，文件路径在 C:\Windows\System32\Drivers\etc\hosts 。

文件成功打开后，在 hosts 文件最后面新增下面一行以完成设置：

```
192.168
.10
.10
   larabbs
.
test
```

因为最近`.dev.app`域名在谷歌浏览器中被强制转跳到 https，所以我们使用 .test。

## 新增站点

如果你安装了 Sublime Text，可通过运行下面命令打开 Homestead.yaml 文件：

```
>
 subl 
~
/
Homestead
/
Homestead
.
yaml
```

在 Homestead.yaml 文件中新增 larabbs 应用的 sites 和 databases 的相关设置：

```
--
-

ip
:
"192.168.10.10"

memory
:
2048

cpus
:
1

provider
:
 virtualbox

authorize
:
~
/
.
ssh
/
id_rsa
.
pub

keys
:
-
~
/
.
ssh
/
id_rsa

folders
:
-
 map
:
~
/
Code
      to
:
/
home
/
vagrant
/
Code

sites
:
-
 map
:
 homestead
.
app
      to
:
/
home
/
vagrant
/
Code
/
Laravel
/
public
-
 map
:
 larabbs
.
test 
# 
<
--- 这里

      to
:
/
home
/
vagrant
/
Code
/
larabbs
/
public
# 
<
--- 这里


databases
:
-
 homestead
    
-
 larabbs 
# 
<
--- 这里


variables
:
-
 key
:
APP_ENV

      value
:
 local


# blackfire:
#     - id: foo
#       token: bar
#       client-id: foo
#       client-token: bar
# ports:
#     - send: 93000
#       to: 9300
#     - send: 7777
#       to: 777
#       protocol: udp
```

我们主要设置了`sites`和`databases`两项。`sites`会将域名`larabbs.test`映射到虚拟机的`/home/vagrant/Code/larabbs/public`文件夹上，而`databases`则为新创建的项目指定数据库名。

## 重启虚拟机

在我们每次对 Homestead.yaml 文件进行了更改之后，都需要运行下面命令来使其更改生效：

```
>
 cd 
~
/
Homestead 
&
&
 vagrant provision 
&
&
 vagrant reload
```

* `vagrant provision`
  是命令 Vagrant 重新加载
  `Homestead.yaml`
  配置；
* `vagrant reload`
  是重启虚拟机使更改生效。

## .env 文件

接下来，我们还需要对应用根目录下的 .env 文件进行设置，为应用指定数据库名称 larabbs。

.env

```
DB_DATABASE
=
larabbs
```

## 初始化命令

进入 homestead

```
$ cd 
~
/
Code
/
larabbs
$ php artisan key
:
generate
$ php artisan migrate 
--
seed
```

[![](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/XhcmcBmdqm.png "file")](https://iocaffcdn.phphub.org/uploads/images/201712/17/6351/XhcmcBmdqm.png)

## 访问应用

现在让我们在 Chrome 浏览器中打开[http://larabbs.test](http://larabbs.test/)你应该能看到有如下界面显示：

[![](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/nKDmnTdat2.png!large "file")](https://iocaffcdn.phphub.org/uploads/images/201810/30/1/nKDmnTdat2.png!large)

