---
title: 解锁网易云音乐
categories: 不正常的文章
date: 2020-03-23 18:26:55
tags: ？？？
---

<!-- more -->

## 前言

通过配置下文中的代理，实现解锁网易云无版权音乐以及部分试听音乐。

使用前你需要知道：

代理端已做限制：*仅允许代理网易云相关域名和ip的请求，其他请求一律拒绝。*由于服务器带宽只有5Mbps，所以理论速度不会超过640kb/s。如果使用的人比较多，可能出现加载比较慢的现象。建议网易云音乐内**开启边听边存**，常听的歌**下载到本地**。最后，你也可以通过文末的打赏按钮对我进行打赏鼓励~（大雾

> *注意：互联网并非法外之地，此代理仅用作学习和交流。*

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

#### android

进入「设置」>「WLAN」>「修改网络」>「高级选项」>「代理」>「代理自动配置」（不同机型设置的地方不一样，也可能在wifi右边的感叹号中），填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

#### ios

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。

其次在「设置」>「无线局域网」>「当前连接网络」>「HTTP 代理」>「配置代理」>「自动」，填写以下地址：

```txt
http://music.griouges.cn:39000/proxy.pac
```

### 方法二：代理软件

使用代理软件，不管是无线网还是流量都是可用的。

> 下面放上不同平台的主流代理软件的使用步骤，请对号入座。

#### Clash for Windows

1. 👉[安装软件](https://lolico.griouges.cn/download/clash/Clash.for.Windows.Setup.0.9.9.exe)
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

鉴于安卓端`导入节点配置文件`可能无法调用Clash应用自动导入的问题，提供*手动导入*方法：

1. 👉进入Clash应用，依次点击「配置」->「新配置」->「从URL导入」
2. 👉填写名称：`lolico.me`，填写URL地址：`https://lolico.me/subscribe/Clash/config.yaml`
3. 👉保存后选中此配置
4. 👉回到主界面启动代理
5. 😘Enjoy it！

*^ Clash配置中有多个节点，当某个节点不可用时，自行切换至其他节点（怎么切换节点我就不多说了，自己琢磨下Clash的使用）*

#### ios

首先下载[CA证书](https://raw.githubusercontent.com/nondanee/UnblockNeteaseMusic/master/ca.crt)，进入「设置」>「通用」>「描述文件」，安装「UnblockNeteaseMusic Root CA」，并在「设置」>「通用」>「关于本机」>「证书信任设置」开启对「UnblockNeteaseMusic Root CA」的信任。

> ios端提供`Shadowrocket`和`QuantumultX`软件的订阅，至于其他客户端，能力者自行根据配置文件修改。

##### Shadowrocket

1. 👉[安装软件](https://apps.apple.com/us/app/shadowrocket/id932747118)
2. 👉[点击导入节点](shadowrocket://add/sub://aHR0cHM6Ly9sb2xpY28ubWUvc3Vic2NyaWJlL1NoYWRvd3JvY2tldC9zZXJ2ZXIudHh0#%F0%9F%8E%B8%E7%BD%91%E6%98%93%E4%BA%91%E9%9F%B3%E4%B9%90)
3. 👉[点击导入配置](shadowrocket://config/add/https://lolico.me/subscribe/Shadowrocket/rules.conf)
4. 😘Enjoy it！

##### QuantumultX

- 方式一（导入全局配置文件）【推荐】
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

建议选择方式一*导入全局配置文件*，其中还包含了**去广告**的配置（默认启用），和一个**解锁b站大会员**的可选配置（*默认禁用，直接开启无效*）

> 部分解锁节点来自telegram频道，如有侵权，请联系删除，谢谢！

> *注意：*如果测试节点真实延迟显示`timeout/超时`是正常的，服务端仅通过网易云相关域名和ip的请求。

## FAQ

1. 为什么开启后，听数字专辑中的歌还是提示要购买？

    直接搜出来的数字专辑中的歌曲是不能直接听的，需要到专辑中听。

2. 为什么会提示无法缓冲或者找不到资源？

    无法缓冲的问题一般是由于网络造成的，请换个网络或者更换节点；找不到资源，请尝试使用其他节点解锁，某些节点搜索资源时使用的平台不同，有的找不到，是正常的。

3. 为什么代理设置并且开启后，登录网易云音乐时提示网络异常？

    先关闭代理再进行登录，进入后再开启代理。

4. 为什么点击导入节点配置文件没反应啊？

    首先确保已下载代理软件，再尝试，安卓可能出现下载了软件还是无法导入的情况，请手动导入，见1.2.3节Clash for Android

5. 为什么安卓系统代理没找到输入地址的地方？

    不同机型设置代理的地方不一样，可能在高级设置中也可能在wifi右边的感叹号中，确保代理方式选择自动代理。

6. 为什么打开代理后某些网站或应用加载不出资源？

    规则代理是根据请求的特征来进行匹配，看是否需要代理，所以说相比不开代理理论上是会有延迟（但基本忽略不计），并且上面给出的配置十分精简，如果感觉到有明显的网络延迟并且能确定不是因为自己网络较差所造成，请在*必要时*再开启代理，日常上网关闭即可。

7. 这写的都是些啥玩意啊？

    ......