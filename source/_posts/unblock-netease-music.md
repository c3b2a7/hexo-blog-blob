---
title: 解锁网易云音乐
categories: 正常的文章
date: 2020-03-23 18:26:55
tags: ？？？
---

通过配置下文中的代理，实现解锁网易云无版权音乐以及部分试听音乐。

<!-- more -->

## 使用方法

使用前你需要知道：

> 程序是部署在我个人服务器上的，并且服务端已限制：*仅允许代理网易云相关域名和ip的请求，其他请求一律丢弃。*由于服务器带宽只有1Mbps，所以速度不会超过128kb/s的，所以使用的人比较多时，可能会出现加载比较慢的现象，请不要进行下载操作，频繁占用服务器资源将会被ban，当然，你可以通过文末的打赏按钮对我进行打赏~，⚽⚽（大雾

<br/>

> *注意：*互联网并非法外之地

### 方法一：系统代理PAC

使用系统代理PAC解锁是最简单的方法，缺点是 Android 和 iOS 只能在连接WiFi的环境下使用。下面介绍不同平台系统代理设置方法。

#### Windows

以 Windows 10 为例，进入「Windows 设置」>「网络和 Internet」>「代理」>「自动设置代理」>「使用设置脚本」，填写以下地址：

```txt
http://unblock.griouges.cn:39001/proxy.pac
```

进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。

#### MacOS

进入「系统偏好设置」>「网络」>「高级」>「代理」，填写以下地址：

```txt
http://unblock.griouges.cn:39001/proxy.pac
```

#### android

进入「设置」>「WLAN」>「修改网络」>「高级选项」>「代理」>「代理自动配置」，填写以下地址：

```txt
http://unblock.griouges.cn:39001/proxy.pac
```

#### ios

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。

其次在「设置」>「无线局域网」>「当前连接网络」>「HTTP 代理」>「配置代理」>「自动」，填写以下地址：

```txt
http://unblock.griouges.cn:39001/proxy.pac
```

### 方法二：代理软件

使用代理软件，不管是无线网还是流量都是可用的。

<br/>

> 下面放上代理软件的使用步骤

#### Clash for Windows

1. 👇[安装软件](https://github.com/Fndroid/clash_for_windows_pkg/releases/download/0.9.2/Clash.for.Windows.Setup.0.9.2.exe)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。
4. 😘Enjoy it!

#### ClashX

1. 👇[安装软件](https://github.com/yichengchen/clashX/releases/download/1.18.3/ClashX.dmg)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 😘Enjoy it!

#### Clash for Android

1. 👇[安装软件](https://github.com/Kr328/ClashForAndroid/releases/download/1.1.10/app-universal-release.apk)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2ffiles%2fsubscribe%2fClash%2fUnblockNeteaseMusic.yaml)
3. 😘Enjoy it!

#### ios

- Shadowrocket
    1. 👇[安装软件](https://apps.apple.com/us/app/shadowrocket/id932747118)
    2. 👉[点击导入节点](shadowrocket://add/sub://aHR0cHM6Ly9sb2xpY28ubWUvZmlsZXMvc3Vic2NyaWJlL1NoYWRvd3JvY2tldC9zaGFkb3dyb2NrZXQtc2VydmVyLnR4dA#UnblockNeteaseMusic)
    3. 👉[点击导入配置](shadowrocket://config/add/https://lolico.me/files/subscribe/Shadowrocket/UnblockNeteaseMusic.conf)
    4. 😘Enjoy it!

- Quantumult
    1. 👇[安装软件](https://apps.apple.com/us/app/quantumult/id1252015438)
    2. 👉[点击导入节点配置文件]()
    3. 😘Enjoy it!

- QuantumultX
    1. 👇[安装软件](https://apps.apple.com/us/app-bundle/quantumult-x-upgrade/id1482985563)
    2. 👉[点击导入节点配置文件]()
    3. 😘Enjoy it!


> *注意：*如果节点测试延迟显示`timeout/超时`是正常的，服务端只会通过网易云相关域名和ip的请求
    