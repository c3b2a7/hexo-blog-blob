---
title: 2020年JetBrains Quest 第一弹
categories: 不正常的文章
date: 2020-03-13 15:53:12
tags: JetBrains Quest
---

2020年3月9日，JetBrains官方推特发布了这么一条消息：

![](https://lolico.griouges.cn/images/20200313155924.png)

> 48 61 76 65 20 79 6f 75 20 73 65 65 6e 20 74 68 65 20 73 6f 75 72 63 65 20 63 6f 64 65 20 6f 66 20 74 68 65 20 4a 65 74 42 72 61 69 6e 73 20 77 65 62 73 69 74 65 3f

观察这些数字，不难猜测是字符对应的ascii码的十六进制数，转换后，得到这样的提示：

> Have you seen the source code of the JetBrains website?

问我们是否看过JetBrains网站的源码
打开[JetBrains官网](https://www.jetbrains.com)查看源码，搜索`JetBrains Quest`：

![](https://lolico.griouges.cn/images/20200313160807.png)

>  Welcome to the JetBrains Quest.
 ​
 What awaits ahead is a series of challenges. Each one will require a little initiative, a little thinking, and a whole lot of JetBrains to get to the end. Cheating is allowed and in some places encouraged. You have until the 15th of March at 12:00 CET to finish all the quests.
 Getting to the end of each quest will earn you a reward.
 Let the quest commence!
 ​
 JetBrains has a lot of products, but there is one that looks like a joke on our Products page, you should start there... (hint: use Chrome Incognito mode)
 It’s dangerous to go alone take this key: Good luck! == Jrrg#oxfn$

大概说的就是在3月15日欧洲中部时间12:00前完成谜题就可以拿到奖励，并且给了我们一个密钥：`Good luck! == Jrrg#oxfn$`

*谜题*：JetBrains有很多产品，但是在我们的产品页面上有一个看起来像joke的产品，您应该从那里开始...（提示：使用谷歌无痕模式）

谷歌浏览器无痕模式访问[JetBrains产品页](https://www.jetbrains.com/products.html)（实际上我不开无痕模式进入也是可以的）
在产品里找到一个叫`JK`的产品，点击`Learn More`：

![](https://lolico.griouges.cn/images/20200313162013.png)

>  You have discovered our JetBrains Quest! If you don’t know what this is, you should start from Twitter, Facebook or LinkedIn.
 ​
 To continue to the next challenge you need to go to the following link… But there is a problem, the last 3 digits are missing:
 ​
 https://jb.gg/###
 ​
 To get these digits you need to know how many prime numbers there are between 500 and 5000
 ​
 Good Luck!

500到5000里有多少个质数，简单，574个。补充到链接上也就是让我们去这个地方：[https://jb.gg/574](https://jb.gg/574)

重定向来到了这里：

![](https://lolico.griouges.cn/images/20200313163046.png)

如果你对JetBrains网站很了解你就会知道这个Logo是youtrack，也就是JebBrains网站的问题区，将后面的问题编码输进去，
得到链接：[https://youtrack.jetbrains.com/issue/MPS-31816](https://youtrack.jetbrains.com/issue/MPS-31816)

跳转到这个页面：

![](https://lolico.griouges.cn/images/20200313162923.png)

> JetBrains Quest
 ​
 “The key is to think back to the beginning.” -- The JetBrains Quest team
 ​
 Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟WkhGulyhWrGhyhors†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2

告诉我们需要回头想一下密钥，那不就是网站源码里的`Good luck! == Jrrg#oxfn$`嘛。那后面那一串东西和这个有什么关系呢？
我就不卖关子了，其实也就是凯撒加密，源码里告诉我们的`Good luck! == Jrrg#oxfn$`其实就相当于一个输入输出案例：

明文`Good luck!`经凯撒加密后得到`Jrrg#oxfn$`

> 凯撒加密：将明文字符以某个数字移位得到另一个字符，说人话也就是把字符的ascii码加或者减某个数得到另一个字符的ascii也就是密文

所以说给出的这个输入输出我们可以知道这个数字是3。

所以说将给出的这个密文还原成明文只需要将每个字符的ascii码减三得到的字符串就是明文了：

```python
if __name__ == "__main__": 
    s = "Qlfh$#Li#|rx#duh#uhdglqj#wklv#|rx#pxvw#kdyh#zrunhg#rxw#krz#wr#ghfu|sw#lw1#Wklv#lv#rxu#lvvxh#wudfnhu#ghvljqhg#iru#djloh#whdpv1#Lw#lv#iuhh#iru#xs#wr#6#xvhuv#lq#Forxg#dqg#iru#43#xvhuv#lq#Vwdqgdorqh/#vr#li#|rx#zdqw#wr#jlyh#lw#d#jr#lq#|rxu#whdp#wkhq#zh#wrwdoo|#uhfrpphqg#lw1#|rx#kdyh#ilqlvkhg#wkh#iluvw#Txhvw/#qrz#lw“v#wlph#wr#uhghhp#|rxu#iluvw#sul}h1#Wkh#frgh#iru#wkh#iluvw#txhvw#lv#‟WkhGulyhWrGhyhors†1#Jr#wr#wkh#Txhvw#Sdjh#dqg#xvh#wkh#frgh#wr#fodlp#|rxu#sul}h1#kwwsv=22zzz1mhweudlqv1frp2surpr2txhvw2"
    step = ord('J')-ord('G')    #Good luck! == Jrrg#oxfn$
    list = list(s)
    for i in range(len(list)):
        list[i] = chr(ord(list[i]) - step)
    print("".join(list))
```

输出：

> Nice! If you are reading this you must have worked out how to decrypt it. This is our issue tracker designed for agile teams. It is free for up to 3 users in Cloud and for 10 users in Standalone, so if you want to give it a go in your team then we totally recommend it. you have finished the first Quest, now it’s time to redeem your first prize. The code for the first quest is “TheDriveToDevelop”. Go to the Quest Page and use the code to claim your prize. https://www.jetbrains.com/promo/quest/

得到一个网址和兑换码。

接下来的兑奖环节应该就不用多说了。