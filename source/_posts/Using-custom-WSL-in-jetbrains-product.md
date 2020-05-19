---
title: 解决JetBrains产品中无法使用非商店发行版本的WSL的问题
categories: 正常的文章
date: 2020-05-19 17:48:45
tags: [WSL,IDEA,WebStorm,PyCharm]
---

JetBrains产品对于WSL提供了一定的支持，但是其只支持Microsoft Store中发行的WSL，对于类似[ArchWSL](https://github.com/yuk7/ArchWSL)这种非商店发行版，在配置工具链时却不能够被发现。下面给出两个方法来解决这个问题。

> 由于我目前只使用过[ArchWSL](https://github.com/yuk7/ArchWSL)这个非商店发行版的WSL，所以下面以ArchWSL为例，对于其他版本的WSL，理论上也行得通。

1. 第一种方法很简单，而且在我测试中也只有这个方法有用

    只需将`Arch.exe`文件拷贝一份到`C:\Users\lolico\AppData\Local\Microsoft\WindowsApps`目录下即可。

    注意，如果你安装了多个Arch，在拷贝时需要将每个版本对应的执行文件都拷贝到`WindowsApps`目录下。

2. 第二种方法是网上搜索到的一个方法，并且在[RubyMine官方文档](https://www.jetbrains.com/help/ruby/configuring-remote-interpreters-using-wsl.html#custom_wsl)中也提到了这个方法，但是使用IDEA和WebStorm测试发现并没有用

    修改文件`%HOMEPATH%\.RubyMine2020.1\config\options\wsl.distributions.xml`

    ```xml
    <descriptor>
        <id>Arch</id>
        <microsoft-id>Arch</microsoft-id>
        <executable-path>Arch.exe</executable-path>
        <presentable-name>ArchLinux for WSL</presentable-name>
    </descriptor>
    ```
    - `executable-path`路径使用执行文件绝对路径。
    - 配置文件夹一般在`%HOMEPATH%`下，不同产品的文件夹名不同，并且2020.1版本迁移到了`%HOMEPATH%\AppData\Local\JetBrains`下，我一直是使用ToolBox来管理这些软件，所以说如果你并不是从ToolBox中安装的软件，配置文件夹可能还是在`%HOMEPATH%`下。修改`wsl.distributions.xml`文件建议在两个地方都试一下。

最后，还想说一句，在使用商店发行版时我们可以用类似`\\wsl$\Ubuntu`这种路径来访问WSL根目录，在使用非商店发行版时却不行，解决的办法还是上面提到第一个方法，拷贝文件到`WindowsApps`之后使用`\\wsl$\Arch`来访问Arch的根目录。

---

*参考：*

^ [*Custom WSL distributions*](https://www.jetbrains.com/help/ruby/configuring-remote-interpreters-using-wsl.html#custom_wsl)
^ [*Support custom WSL distributive (not from Microsoft Store)*](https://youtrack.jetbrains.com/issue/Py-32424#focus=streamItem-27-3332472.0-0)