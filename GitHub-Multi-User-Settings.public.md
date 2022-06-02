
```
Author: Weiqing
Time  : 202206024

Intro : config about multi-git-user
```


# GitHub多账号配置


在同一台机器上，为多个GitHub账号配置不同的config，包括includeif和ssh，互不打扰地访问。











--------------------------------------------------------------------------------



## 本地配置多账号gitconfig

当我们有工作和个人多个账号的时候，经常忘记手动设置user信息出现错误提交的情况。
但是现在git帮我们解决了这个问题，在git的`version >= 2.13`中支持了`includeIf`来为本地不同的git账号配置不同的config文件，这样就做到了不同身份的隔离。

首先了解一下Git配置文件生效的优先级。
对于一个Git仓库来说，配置优先级为 `仓库 > 全局 > 系统`。
操作Git时，首先会查找`/etc/gitconfig`(系统)，然后查找用户的全局配置`~/.gitconfig`，最后查找每个仓库的`.git/config`配置。
所有的配置项，从低优先级开始加载，出现冲突时，较高优先级的配置项会覆盖前面的配置。




### includeIf介绍

从 git 2.13.0 开始，git 配置文件开始支持 Conditional Includes 的配置。通过设置 includeIf..path，可以向命中 condition 的 git 仓库引入 path 指向的一个 git 配置文件中配置。

```
[includeIf “keyword:/your/path”] 
    path = path/to/gitconfig 
```

其中支持的 keyword 有：
- `gitdir`: 其中 是一个 glob pattern 如果代码仓库的.git目录匹配 指定的 glob pattern，那么条件命中； 
- `gitdir/i`：gitdir的大小写不敏感版本。 
- `onbranch`：其中 是匹配分支名的一个glob pattern。 假如代码仓库中分支名匹配，那么条件命中。 
就我们的需求，使用 gitdir 完全可以。

说到includeif，就不得不说先说一下include，先看下官方demo

```
[include]
    path = /path/to/foo.inc ; 绝对路径
    path = foo.inc ; 相对路径
    path = ~/foo.inc ; 查询$HOME路径

; 当仓库所在目录包含gitdir之后的路径才会使用.inc文件
[includeIf "gitdir:/path/to/foo/.git"]
    path = /path/to/foo.inc

; 所有仓库目录在gitdir之后的路径下的，都会使用.inc文件
[includeIf "gitdir:/path/to/group/"]
    path = /path/to/foo.inc

; 路径描述也可以用定义过的环境变量代替
[includeIf "gitdir:~/to/group/"]
    path = /path/to/foo.inc
```

由上可以看出，include可以将配置的文件给包含进去。
而includeif，就像demo写的一样，可以通过对gitdir的设置，把不同仓库，或者包含不同仓库的文件夹设置为不同的config文件。

至此就已经很清楚了，只需要将不同的username及email写在不同的config里，然后在global的config中把includeif路径设置好，对应上不同的gitconfig即可。


### 利用includeIf

先把之前的全局用户给取消掉。

```
git config --global --unset user.name
git config --global --unset user.email
git config --global --unset user.password
```

对账号`A`设置一个总目录`path/to/A`，新建`.A-config`隐藏文件夹，新建`.A.gitconfig`配置文件。

```
[user]
    name = A
    email = A@xxx.email
```

对账号`B`的，与`A`的同理。

在全局配置文件`~/.gitconfig`中添加`includeIf`信息。

```
[includeIf "gitdir:D:/path/to/A"]
    path = D:/path/to/A/.A-config/.A.gitconfig
[includeIf "gitdir:D:/path/to/B"]
    path = D:/path/to/B/.B-config/.B.gitconfig
```

大功告成。

这时候，在A总目录下新建临时仓库test，init之后查看user信息是自己设置的A，已经可以自动识别为A了。

这里建立了一个专用的隐藏文件夹`.A-config`是为了方便管理所有的A相关配置，而不易和其他正常的仓库文件夹易混。

### 遇到的问题以及解决方法

一开始配置完怎么也无法在账号文件夹内自动识别`user.name`, 搜了下，发现`gitdir`可以用`gitdir/i`来代替。
这样可以解决大小写敏感的问题，大概是说win中某个地方传递到时候驱动器符号会变成小写的。
然而，修改为`gitdir/i`后依然不行。

心血来潮，把相对路径改成了**绝对路径**，竟然生效了！

另外，这里备注一个没有尝试过的东西。
> 问题描述：
> 有两个GitHub账号，其中一个仓库想要推到新的远程账户上去。使用了git config进行了新的账号和邮箱配置，也添加了新的ssh。但是当使用git push时候显示无权限，因为仍使用的是旧账号。
> 解决办法：
> 应该不是最佳，但确实有效的方法。
> **到控制面板打开凭据管理器，删掉GitHub的账号。**











--------------------------------------------------------------------------------


## 配置多个账号的ssh

win10系统，配置多个GitHub账号的ssh，不全尽如人意。

据说直接用GitHub的客户端可以登录多个账号，鉴于流量不多，先不下载尝试了。

### 先分别为每个账号生成密钥对

```
# 生成密钥
ssh-keygen -t rsa -b 4096 -C "yourname@email.com" 
# 可以自定义路径和名称
# 可以增加密码

# 查看已经托管到ssh-agent的密钥
ssh-add -l 
# 添加新生成的密钥, 加-k似乎也行
ssh-add 路径和名称
```

如果查看的时候出错，可以到`服务`里面把`OpenSSH ...`的那个服务启动。

把新生成的密钥中的公钥，也就是带有`.pub`后缀的文件，把内容复制，打开GitHub相关SSH设置`new SSH key`，然后粘贴保存。

### 修改ssh的配置文件

打开`~/.ssh/config`文件，这里win10下是`C:/Users/YourName/.ssh/config`，添加如下别名配置。

```
# 配置多个GitHub账号
Host gh_A
    HostName github.com
    AddKeysToAgent yes
    IdentityFile D:/your/path/to/A_rsa
    User git

Host gh_B
    HostName github.com
    IdentityFile D:/your/path/to/B_rsa
    PreferredAuthentications publickey
    User git
```


接下来就可以进行测试了。

```
ssh -T git@gh_A
ssh -T git@gh_B
```

以后添加`git`的`url`的时候，需要进行一下小小的修改才能`push`成功。（没有试第二个但感觉也行）

```
git remote add origin git@gh_A:NameA/test.git
git remote add origin gh_B:NameB/test.git
```












--------------------------------------------------------------------------------



## 参考 

在[Git多账号配置](https://xblcity.github.io/blog/fe-engineering/git-account.html)中，有说可以把win凭据管理器中的github密码换成token，在ssh的config中的user可以填写自己的邮箱。
这两个目前都还没有尝试，但是我感觉token的方式也不一定支持多账号的。




### 已用参考

1. [配置Git多目录下区分不同用户的提交](https://www.jianshu.com/p/2627ab5742a5)
2. [【Git】关于config环境分离以及includeIf用法](https://www.taoism-one.com/articles/2019/01/06/1546774904436.html)
3. ssh参考的自己以前的笔记





