---
title: Git连接多个GitHub账号
tags:
  - git
categories: 正常的文章
date: 2020-04-15 19:11:50
---

## 前言

用ssh连接GitHub，需要在GitHub账号上传唯一的公钥。当我们需要连接两个或多个GitHub账号，上传同一个公钥是不允许的，那么该如何设置才能在一台电脑上连接多个GitHub账号呢？

<!-- more -->

## 创建密钥并上传到GitHub

假设我们已经有一对默认的密钥`id_rsa`、`id_rsa.pub`关联了a账号，现在我们来为b账号创建一个密钥：

```bash
ssh-keygen -t rsa -f ~/.ssh/id_rsa_b -C "youmail@example.com"
```

注意文件名不要与其他的密钥重复，否则会覆盖之前的密钥。
windows下使用绝对路径：`C:\Users\xxx\.ssh\id_rsa_b`

接下来我们就可以在用户目录下的`.ssh`文件夹中找到`id_rsa_b`和`id_rsa_b.pub`两个文件，将`id_rsa_b.pub`文件中的内容保存到b账号的ssh keys中。

## 配置身份验证使用的密钥

接下来我们要做的就是对不同账号配置使用不同的密钥去连接github。在`.ssh`文件夹中创建一个`config`文件（无扩展名），填入以下内容：

```txt
# default
Host github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa

# b
Host b.github.com
    HostName github.com
    PreferredAuthentications publickey
    IdentityFile ~/.ssh/id_rsa_b
```

在这部分配置中我们使用不同的主机标识去映射同一个`HostName`即github.com，但是我们使用不同的密钥文件去进行身份验证。

配置中的`Host`在不和其他`Host`重复的情况下可以随意填写，但是建议使用一眼就能看懂的标识（并且这个标识在后面是需要用到的）

到这里，其实所有的工作都已经完成。在进行下一步操作前我们先测试ssh是否能连接到github：

```bash
ssh -T git@github.com
ssh -T git@b.github.com
```

不出意外，我们将看到验证成功的提示。如果提示 CreateProcessW failed error:2，先将config文件中的代理设置注释便可，实际上并不影响后续对仓库的推送。

## 对b账号的仓库单独设置用户名和邮箱

现在我们要做的就是对b账号下的仓库进行设置，因为在原本只有一个账号时，我们应该都用过下面这个命令来对git设置一个全局的用户名和邮箱：

```bash
git config --global user.name "a"
git config --global user.email "a@example.com"
```

现在我们当然不能对b账号下的仓库也使用全局的用户名和邮箱。如果使用全局的邮箱将b的仓库push到github，那么在github提交记录中看到的提交者将会是a，所以说我们需要对b账号下的仓库单独配置用户名和邮箱，进入项目文件夹：

```bash
git config user.name "b"
git config user.email "b@example.com"
```

邮箱要使用b账号注册github时使用的邮箱，因为github提交记录中的提交者是根据这个邮箱来查找的，这也是不能使用全局邮箱后提交者是a的原因。

## 修改远程仓库的地址

接下来我们还要重新设置远程仓库的地址，因为克隆仓库或者创建github仓库添加远程仓库地址时使用的主机标识默认是`github.com`，然而在config文件中对这个`Host`是使用`~/.ssh/id_rsa`密钥去验证，所以当我们本地已有b账号下的仓库或者未来克隆b账号下的仓库时要修改默认的远程仓库地址：

```bash
git remote rm origin
git remote add origin git@b.github.com:b/repo.git
```

注意不要照抄，根据config文件中配置的`Host`，github账号以及仓库名去设置'@'符号后面的东西。

所有的工作都已经完成，push测试一下：

```bash
git push origin master
```

如果我们单纯的使用`git push`可能会报错，因为虽然添加了远程仓库，但是并没有设置本地分支具体跟踪哪个上游分支。使用`-u`或`--set-upstream-to`选项运行`git branch`来显式地设置：

```bash
git branch -u origin/master master
```

## 总结

总的来说，我们就是使用了不同的主机标识去映射github.com，并且对不同的标识在连接github时使用相应的密钥去验证身份。我们设置多个标识，相应的，我们也要修改远程仓库使用的Host标识（因为克隆下来的仓库的远程地址Host标识默认使用github.com）。虽然上面我们是拿两个账号来举例子，但是对于多个账号，设置的方法还是一模一样的。