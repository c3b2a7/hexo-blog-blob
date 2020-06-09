---
title: 解锁网易云音乐
categories: 不正常的文章
date: 2020-03-23 18:26:55
tags: ？？？
---

<!-- more -->

## 前言

通过配置下文中的代理，实现解锁网易云无版权音乐以及试听音乐，文中使用到的项目：[UnblockNeteaseMusic](https://github.com/nondanee/UnblockNeteaseMusic)。

使用前你需要知道：

代理端已做限制：*仅允许代理网易云相关域名和ip的请求，其他请求一律拒绝。*由于服务器带宽只有5Mbps，所以理论速度不会超过640kb/s。如果使用的人比较多，可能出现加载比较慢的现象。建议网易云音乐内**开启边听边存**，常听的歌**下载到本地**。最后，你也可以通过文末的打赏按钮对我进行打赏鼓励~（大雾

> *注意：*互联网并非法外之地，此代理完全免费并仅用作学习与交流，一切收费倒卖行为请及时举报并反馈。

## 使用方法

现提供两种方法，不想折腾的推荐`方式一`，但是只能用于wifi连接，流量不可用，否则选择`方式二`（本人更推荐方式二）

### 方法一：系统代理PAC

使用系统代理PAC解锁是最简单的方法，缺点是 android 和 ios 只能在连接WiFi的环境下使用。下面介绍不同平台系统代理设置方法（连接二选一即可）

#### Windows

以 Windows 10 为例，进入「Windows 设置」>「网络和 Internet」>「代理」>「自动设置代理」>「使用设置脚本」，填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。

#### MacOS

进入「系统偏好设置」>「网络」>「高级」>「代理」，填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

#### Android

进入「设置」>「WLAN」>「修改网络」>「高级选项」>「代理」>「代理自动配置」（不同机型设置的地方不一样，也可能在wifi右边的感叹号中），填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

#### iOS

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。（[详见官方教程](https://support.apple.com/zh-cn/HT204477)）

其次在「设置」>「无线局域网」>「当前连接网络」>「HTTP 代理」>「配置代理」>「自动」，填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

### 方法二：代理软件

使用代理软件，不管是无线网还是流量都是可用的。

> 下面放上不同平台的主流代理软件的使用步骤，请对号入座。

#### Clash for Windows

1. 👉[安装软件](https://lolico.griouges.cn/download/clash/Clash.for.Windows.Setup.0.10.1.exe)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2fsubscribe%2fClash%2fconfig.yaml)
3. 👉进入「General」，开启「System Proxy」
4. 👉进入网易云音乐「设置」>「工具」>「Http代理」，选择「使用 IE 代理设置」。
5. 😘Enjoy it！

#### Clash for MacOS

1. 👉[安装软件](https://lolico.griouges.cn/download/clash/ClashX.dmg)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2fsubscribe%2fClash%2fconfig.yaml)
3. 😘Enjoy it！

#### Clash for Android

1. 👉[安装软件](https://lolico.griouges.cn/download/clash/app-armeabi-v7a-release.apk)
2. 👉[点击导入节点配置文件](clash://install-config?url=https%3a%2f%2flolico.me%2fsubscribe%2fClash%2fconfig.yaml)
3. 👉保存后应用此配置
4. 👉回到主界面启动代理
5. 😘Enjoy it！

**注意：**

鉴于安卓端`导入节点配置文件`可能无法调起Clash进行自动导入，请手动导入配置：

1. 👉进入Clash应用，依次点击「配置」->「新配置」->「从URL导入」
2. 👉填写名称：`lolico.me`，URL地址：`https://lolico.me/subscribe/Clash/config.yaml`，自动更新：1440
3. 👉保存后选中此配置
4. 👉回到主界面启动代理
5. 😘Enjoy it！

> Clash配置中有多个节点，当某个节点不可用时，自行切换至其他节点或更新配置，怎么切换节点就不多说了，请自己琢磨下Clash如何使用。

#### iOS

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。（[详见官方教程](https://support.apple.com/zh-cn/HT204477)）

> iOS端提供`Shadowrocket`和`QuantumultX`软件的订阅，至于其他客户端，能力者自行根据配置文件修改。

##### Shadowrocket

1. 👉[安装软件](https://apps.apple.com/us/app/shadowrocket/id932747118)
2. 👉[点击导入节点](shadowrocket://add/sub://aHR0cHM6Ly9sb2xpY28ubWUvc3Vic2NyaWJlL1NoYWRvd3JvY2tldC9zZXJ2ZXIudHh0#%F0%9F%8E%B8%E7%BD%91%E6%98%93%E4%BA%91%E9%9F%B3%E4%B9%90)
3. 👉[点击导入配置](shadowrocket://config/add/https://lolico.me/subscribe/Shadowrocket/rules.conf)
4. 😘Enjoy it！

##### QuantumultX

- 方式一（导入全局配置文件）
    1. 👉[安装软件](https://apps.apple.com/us/app-bundle/quantumult-x-upgrade/id1482985563)
    2. 👉进入应用 -> 点击右下角 -> 滑至底部 -> 下载
    3. 👉填入地址：`https://lolico.me/subscribe/QuantumultX/pro.conf`
    4. 😘Enjoy it！

- 方式二（自行修改配置文件）
    1. 👉[安装软件](https://apps.apple.com/us/app-bundle/quantumult-x-upgrade/id1482985563)
    2. 👉进入应用 -> 点击右下角 -> 滑至底部 -> 编辑
    3. 👉在相应位置添加以下配置：
    ```
    [policy]
    static=🎸网易云音乐, direct, 🎵 解锁节点1, 🎵 解锁节点2, 🎵 解锁节点3, 🎵 解锁节点4, img-url=https://raw.githubusercontent.com/Koolson/Qure/master/IconSet/Netease_Music_Unlock.png

    [server_remote]
    https://lolico.me/subscribe/QuantumultX/NeteaseMusicServer.txt, tag=Netease Music, enabled=true, img-url=https://raw.githubusercontent.com/Koolson/Qure/master/IconSet/Netease_Music_Unlock.png

    [filter_remote]
    https://lolico.me/subscribe/QuantumultX/NeteaseMusicFilter.txt, tag=🎸网易云音乐, force-policy=🎸网易云音乐, enabled=true
    ```
    4. 😘Enjoy it！

> 部分解锁节点来自telegram频道，如有侵权，请联系删除，谢谢！

> *注意：*如果测试节点真实延迟显示`timeout/超时`是正常的，服务端开启严格模式后仅能通过网易云相关域名或ip的请求。

## FAQ

1. 为什么开启代理后，听数字专辑中的音乐会提示购买？

    直接搜数字专辑中的音乐在播放时可能出现这种现象，尝试从专辑中进入并播放。

2. 为什么开启代理后，登录网易云音乐提示网络异常？

    先关闭代理再进行登录，进入后再开启代理。

3. 为什么播放音乐提示网络不给力或者歌曲不存在？

    iOS端出现网络不给力时，请确保CA证书已信任；歌曲不存在，请尝试使用其他节点解锁，不同节点在搜索音源时使用的平台不同，部分歌曲找不到是正常的现象。

4. 为什么播放的音乐是live版或者完全不同的一首音乐？

    由于解锁服务是从其他平台搜索音源，并且选择策略不可能做到十全十美，在音乐重名并且火热程度不同的情况下，可能会出现这种现象，目前没有较好的解决方法。

5. 为什么开启代理后某些网站或应用加载资源很慢甚至失败？

    由于使用代理并且根据请求分流，所以说相比不开代理理论上的确会有延迟（基本忽略不计）。如果感觉有明显的网络延迟并且确定不是由于自己网络环境较差所致，请在*必要时*再开启代理，日常上网关闭即可。

6. 这写的都是些啥玩意？

    ......