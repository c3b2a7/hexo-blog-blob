---
title: 解决JetBrains产品中无法使用非商店发行版本的WSL的问题
categories: 正常的文章
date: 2020-05-19 17:48:45
tags: [WSL,IDEA,WebStorm,PyCharm]
---

JetBrains产品对于WSL提供了一定的支持，但是其只支持Microsoft Store中发行的WSL，对于类似[ArchWSL](https://github.com/yuk7/ArchWSL)这种非商店发行版，在配置工具链时却不能够被发现。下面给出两个方法来解决这个问题。

> 由于我目前只使用过[ArchWSL](https://github.com/yuk7/ArchWSL)这个非商店发行版的WSL，所以下面以ArchWSL为例，对于其他版本的WSL，理论上也行得通。

1. 第一种方法很简单，只需要复制执行文件到指定目录

    只需将`Arch.exe`文件拷贝一份到`C:\Users\lolico\AppData\Local\Microsoft\WindowsApps`目录下即可。

    *注意，如果你安装了多个Arch，在拷贝时需要将每个版本对应的执行文件都拷贝到`WindowsApps`目录下。*

2. 第二种方法是网上搜索到的一个方法，并且在[RubyMine官方文档](https://www.jetbrains.com/help/ruby/configuring-remote-interpreters-using-wsl.html#custom_wsl)中提到了这个方法

    以RubyMine为例，修改`wsl.distributions.xml`文件：

    ```xml
    <descriptor>
        <id>Arch</id>
        <microsoft-id>Arch</microsoft-id>
        <executable-path>Arch.exe</executable-path>
        <presentable-name>Arch Linux</presentable-name>
    </descriptor>
    ```

    - `executable-path`使用执行文件的绝对路径。
    - `wsl.distributions.xml`文件位于产品对应的配置文件夹下，对于不同版本，配置文件夹位置可能不一样。
    
  > 旧版本软件的配置文件夹在`%HOMEPATH%`下，此时该文件位于配置文件夹下的`config\options`目录中。在2020.1版本，配置文件夹迁移到了`%HOMEPATH%\AppData\Roaming\JetBrains`下，此时该文件位于配置文件夹下的`options`目录中。由于我一直使用ToolBox来安装并管理这些软件，所以说如果你并不是从ToolBox中安装的软件，新版本的配置文件夹可能仍然在`%HOMEPATH%`下。

最后，还想说一句，在使用商店发行版时我们可以用类似`\\wsl$\Ubuntu`这种路径来访问WSL的根目录，在使用非商店发行版时却不行，解决的办法还是上面提到第一个方法，拷贝文件到`WindowsApps`之后即可使用`\\wsl$\Arch`来访问Arch的根目录。

---

*参考：*

^ [*Custom WSL distributions*](https://www.jetbrains.com/help/ruby/configuring-remote-interpreters-using-wsl.html#custom_wsl)
^ [*Support custom WSL distributive (not from Microsoft Store)*](https://youtrack.jetbrains.com/issue/Py-32424#focus=streamItem-27-3332472.0-0)