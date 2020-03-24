---
title: 解锁网易云音乐
categories: 正常的文章
date: 2020-03-23 18:26:55
tags: ？？？
---

<!-- more -->

通过配置下文中的代理，实现解锁网易云无版权音乐以及部分试听音乐。

## 使用方法

不想折腾的推荐`方式一`，但是只能用于wifi连接，流量不可用，否则选择`方式二`。

使用前你需要知道：

> 程序部署在服务器上，并且服务端已做限制：*仅允许代理网易云相关域名和ip的请求，其他请求一律丢弃。*由于服务器带宽只有5Mbps，所以速度不会超过640kb/s，如果使用的人比较多，可能出现加载比较慢的现象。建议网易云音乐内开启**自动缓存**，常听的歌建议**下载到本地**。当然，你也可以通过文末的打赏按钮对我进行打赏~（大雾

<br/>

> *注意：互联网并非法外之地，且用且珍惜*

### 方法一：系统代理PAC

使用系统代理PAC解锁是最简单的方法，缺点是 android 和 ios 只能在连接WiFi的环境下使用。下面介绍不同平台系统代理设置方法（连接二选一即可）

#### Windows

以 Windows 10 为例，进入「Windows 设置」>「网络和 Internet」>「代理」>「自动设置代理」>「使用设置脚本」，填写以下地址：

```txt
首选：http://music.desperadoj.com:30000/proxy.pac
备用：http://music.griouges.cn:39000/proxy.pac
```

进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。

#### MacOS

进入「系统偏好设置」>「网络」>「高级」>「代理」，填写以下地址：

```txt
首选：http://music.desperadoj.com:30000/proxy.pac
备用：http://music.griouges.cn:39000/proxy.pac
```

#### android

进入「设置」>「WLAN」>「修改网络」>「高级选项」>「代理」>「代理自动配置」（不同机型设置的地方不一样，也可能在wifi右边的感叹号中），填写以下地址：

```txt
首选：http://music.desperadoj.com:30000/proxy.pac
备用：http://music.griouges.cn:39000/proxy.pac
```

#### ios

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。

其次在「设置」>「无线局域网」>「当前连接网络」>「HTTP 代理」>「配置代理」>「自动」，填写以下地址：

```txt
首选：http://music.desperadoj.com:30002/proxy.pac
备用：http://music.griouges.cn:39000/proxy.pac
```

### 方法二：代理软件

使用代理软件，不管是无线网还是流量都是可用的。

<br/>

> 下面放上代理软件的使用步骤，请对号入座

#### Clash for Windows

1. 👇[安装软件](https://lolico.griouges.cn/uploads/Clash.for.Windows.Setup.0.9.2.exe)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 👉进入「General」，开启「System Proxy」
4. 👉进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。
5. 😘Enjoy it!

#### ClashX for MacOS

1. 👇[安装软件](https://lolico.griouges.cn/uploads/ClashX.dmg)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 😘Enjoy it!

> Windows 和 Mac 备用Clash配置文件👉[一键导入](clash://install-config?url=https%3a%2f%2fraw.githubusercontent.com%2fDesperadoJ%2fRules-for-UnblockNeteaseMusic%2fmaster%2fClash%2fUnblockNeteaseMusic.yaml)
#### Clash for Android

1. 👇[安装软件](https://lolico.griouges.cn/uploads/app-universal-release.apk)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 😘Enjoy it!

**鉴于安卓端`导入节点配置文件`可能无法调用Clash应用自动导入的问题，现可手动导入：**
 
1. 进入Clash应用，点击配置 -> 新配置 -> URL导入
2. 名称随意，url首选https://raw.githubusercontent.com/DesperadoJ/Rules-for-UnblockNeteaseMusic/master/Clash/UnblockNeteaseMusic.yaml
其次https://lolico.me/files/subscribe/Clash/UnblockNeteaseMusic.yaml
3. 保存后应用此配置
4. 回到主界面启动代理
5. Enjoy it

<br/>

#### ios

- Shadowrocket
    1. 👇[安装软件](https://apps.apple.com/us/app/shadowrocket/id932747118)
    2. 👉[点击导入节点](shadowrocket://add/sub://aHR0cHM6Ly9sb2xpY28ubWUvZmlsZXMvc3Vic2NyaWJlL1NoYWRvd3JvY2tldC9zaGFkb3dyb2NrZXQtc2VydmVyLnR4dA#UnblockNeteaseMusic)
    3. 👉[点击导入配置](shadowrocket://config/add/https://lolico.me/files/subscribe/Shadowrocket/UnblockNeteaseMusic.conf)
    4. 😘Enjoy it!

- Quantumult
    1. 👇[安装软件](https://apps.apple.com/us/app/quantumult/id1252015438)
    2. 👉[点击导入节点配置文件](quantumult://configuration?server=aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL0Rlc3BlcmFkb0ovUnVsZXMtZm9yLVVuYmxvY2tOZXRlYXNlTXVzaWMvbWFzdGVyL1F1YW50dW11bHQvcXVhbnR1bXVsdC1zZXJ2ZXIudHh0&filter=aHR0cHM6Ly9yYXcuZ2l0aHVidXNlcmNvbnRlbnQuY29tL0Rlc3BlcmFkb0ovUnVsZXMtZm9yLVVuYmxvY2tOZXRlYXNlTXVzaWMvbWFzdGVyL1F1YW50dW11bHQvVW5ibG9ja05ldGVhc2VNdXNpYy5jb25m)
    3. 😘Enjoy it!


> *注意：*如果节点测试延迟显示`timeout/超时`是正常的，服务端只会通过网易云相关域名和ip的请求

## FAQ

1. 为什么开启后，听数字专辑中的歌还是提示要购买？
    
    直接搜出来的数字专辑中的歌曲是不能直接听的，需要到专辑中听。

2. 为什么提示无法缓冲？

    无法缓冲的问题一般是由于网络造成的，请重试，如果还是不行，可能服务器也没找到资源。

3. 为什么代理设置并且开启后，登录网易云音乐时提示无网络连接？

    关闭代理后登录，登陆进入了再开启代理。

4. 导入配置文件的链接点了没反应啊？

    首先确保已下载代理软件，再尝试，安卓可能出现下载了软件还是无法导入的情况，请手动导入，见1.2.3节Clash for Android

5. 为什么安卓系统代理没找到输入地址的地方？

    不同机型设置代理的地方不一样，可能在高级设置中也可能在wifi右边的感叹号中，确保代理方式选择自动代理。
