---
title: 2020年JetBrains Quest 第二弹
categories: 不正常的文章
date: 2020-03-14 15:33:33
tags: ？？？
---

2020年3月11，JetBrains Quest第二弹到来：

![](https://lolico.griouges.cn/images/20200314153806.png)

> .spleh A+lrtC/dmC .thgis fo tuo si ti semitemos ,etihw si txet nehw sa drah kooL .tseretni wohs dluohs uoy ecalp a si ,dessecorp si xat hctuD erehw esac ehT .sedih tseuq fo txen eht erehw si ,deificeps era segaugnal cificeps-niamod tcudorp ehT

看着这一串数字不难看出来是经过reverse过的，python解决：

```python
    s = '.spleh A+lrtC/dmC .thgis fo tuo si ti semitemos ,etihw si txet nehw sa drah kooL .tseretni wohs dluohs uoy ecalp a si ,dessecorp si xat hctuD erehw esac ehT .sedih tseuq fo txen eht erehw si ,deificeps era segaugnal cificeps-niamod tcudorp ehT'
    print(s[::-1])
```

输出：

> The product domain-specific languages are specified, is where the next of quest hides. The case where Dutch tax is processed, is a place you should show interest. Look hard as when text is white, sometimes it is out of sight. Cmd/Ctrl+A helps.

来到[MPS产品页](https://www.jetbrains.com/mps/)，搜索`Dutch tax`：

![](https://lolico.griouges.cn/images/20200314155001.png)

点击`Read MPS case study`跳转到一个PDF文件：

![](https://lolico.griouges.cn/images/20200314155326.png)

还记的这个提示吗：

> Look hard as when text is white, sometimes it is out of sight. Cmd/Ctrl+A helps.

`Ctrl+A`试一下：

![](https://lolico.griouges.cn/images/20200314155427.png)

右边有4行文字隐藏了，复制出来康康：

> This is our 20th year as a company,
 we have shared numbers in our JetBrains
 Annual report, sharing the section with
 18,650 numbers will progress your que

这句话的提到了`JetBrains Annual report`，看看去[https://www.jetbrains.com/company/annualreport/2019/](https://www.jetbrains.com/company/annualreport/2019/)
在这个页面兜兜转转，搜索18650，怎么都发现不了什么线索，参考这个[帖子评论区](https://v2ex.com/t/651961)，发现这些数字加起来其实就是18650
而线索就藏在分享的编辑栏里面：

![](https://lolico.griouges.cn/images/20200314160727.png)

点击分享：

![](https://lolico.griouges.cn/images/20200314160831.png)

> I have found the JetBrains Quest! Sometimes you just need to look closely at the Haskell language, Hello,World! in the hackathon lego brainstorms project https://blog.jetbrains.com/blog/2019/11/22/jetbrains-7th-annual-hackathon/#JetBrainsQuest

给了个博客链接，点进去搜索`lego brainstorms`：

![](https://lolico.griouges.cn/images/20200314161059.png)

还是没有头绪，F12开发者模式查看这种图片：

![](https://lolico.griouges.cn/images/20200314161734.png)

发现在alt属性中有这么一串文字：

> d1D j00 kN0w J378r41n2 12 4lW4Y2 H1R1N9? ch3CK 0u7 73h K4r33r2 P493 4nD 533 1f 7H3r3 12 4 J08 F0r J00 0R 4 KW357 cH4LL3n93 70 90 fUr7h3r @ l3457.

这玩意有的单词看着想是某些字母用数字代替，有的又看不出来，在gist上看了下才知道这是嘤文的火星文🙄🙄🙄，其实在第一弹的邮件里也有一串差不多的彩蛋。
翻译过来就是：

> Did you know Jetbrains is always hiring? Check out the kareers(careers) page and see if there is a job for you or for kwest(quest) challenge to go further at least.

去[职位发布页面](https://www.jetbrains.com/careers/jobs/)看看：

搜索`Quest`并点进去：

![](https://lolico.griouges.cn/images/20200314162653.png)

看一下啥是`cheat at Konami games`：

![](https://lolico.griouges.cn/images/20200314162956.png)

`上上下下左右左右BA`，原来还有个名字叫做`Konami Code`，相当于一个隐藏游戏的启动代码。
魂斗罗30条命！我来辽！！！

![](https://lolico.griouges.cn/images/20200314163313.png)

↑↑↓↓←→←→BA

![](https://lolico.griouges.cn/images/20200314163436.png)

wow，打掉所有块，又白嫖到3个月的奖励，哈哈哈哈哈哈
