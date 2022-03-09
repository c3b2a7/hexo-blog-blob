---
title: 利用亚马逊aws一年EC2实例搭建酸酸乳科学上网
categories: 不正常的文章
date: 2018-07-01 15:21:36
---

## 搭建前的主机准备工作

### 注册aws账号
首先在[aws官网网](https://aws.amazon.com)
注册一个账号，并用信用卡进行验证。注册账号的方法就不详说，至于用信用卡进行认证，可以从淘宝上买一个visa信用卡进行验证，完成这些步骤后我们就可以免费使用aws一年EC2实例，搭建ss/ssr进行科学上网。

### 创建EC2实例并启动
搭建ss/ssr之前，我们需要先启动一个实例。
1. 选择创建实例的地区，**注意：在成功验证信用卡后aws会发送一封邮件到注册的邮箱，在邮件中会告诉我们可用地区的EC2实例，博主之前也是没注意到这封邮件，导致在不允许的地区创建实例时不能启动**

2. 点击服务选择EC2
![aws创建实例](https://i.loli.net/2020/03/09/7o869nKH34QjpfT.png)

3. 点击启动实例
![aws启动实例](https://i.loli.net/2020/03/09/BlbTyHwW28oqted.png)

4. 选择实例的操作系统，这里我选择的是第一个Linux系统
![选择EC2系统](https://i.loli.net/2020/03/09/Td3HWwtmQYzfC2x.png)

5. 2345步都可默认，点击下一步，进入配置安全组界面
![默认点击下一步](https://i.loli.net/2020/03/09/xLqn4TcitW13dJZ.png)

6. 在配置安全组页面我们需要添加一个入站规则，选择所有流量，来源中填写0.0.0.0，最后点击审核和启动
![添加入站规则](https://i.loli.net/2020/03/09/p32ShLE1tzk4uxU.png)

7. 选择登入实例的密钥对，没有的话点击创建密钥对，并下载这个密钥对，(密钥十分重要，这是登入aws实例的唯一钥匙)
![创建密钥对](https://i.loli.net/2020/03/09/7Bm5PjxpylLFKi3.png)
*——到这里，实例的创建就完成了，接下来就是连接我们刚才创建的实例并搭建ss/ssr*

## 使用Xshell登入实例
在这里我们选择使用Xshell 5连接实例
1. 点击查看实例，我们可以看见我们刚才创建的实例的ip，记下这个ip打开Xshell
![实例ip](https://i.loli.net/2020/03/09/Ymr1MnfOw8XB4Jx.png)
2. 点击工具中的用户密钥管理者，并导入上文说的十分重要的那个密钥
![导入密钥](https://i.loli.net/2020/03/09/t8VCA9bjwUJgK5G.png)
3. 导入密钥后点击Xshell右上角文件中的新建在主机项中填入刚才记下的ip
![填入ip](https://i.loli.net/2020/03/09/FIwq3plSZLVBvcx.png)
4. 再点击右边用户身份验证，选择图中的方法，并选择登入的密钥，在用户名项中填入ec2-user，并确认(Linux系统的填ec2-user，Ubuntu系统填Ubuntu，否则可能会连接失败)
![选择登入方法和密钥](https://i.loli.net/2020/03/09/8kRVorWCpzGZSPX.png)
5. 连接刚才新建的会话，第一次连接会有个提示，选择接受并保存，如图所示则连接成功
![连接成功](https://i.loli.net/2020/03/09/rpmBA9TidQa1vbz.png)
*——到这一步，我们就成功连接上了实例，现在就可以开始搭建ss/ssr了*

## 搭建ss/ssr
1. 输入```sudo su```获取root权限
2. 安装脚本

    - ss
    ```bash
        wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/ss-go.sh && chmod +x ss-go.sh && sudo bash ss-go.sh
    ```
    - ssr
    ```bash
    wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/ssr.sh && chmod +x ssr.sh && sudo bash ssr.sh
    ```

安装完成后是这样的
![安装完成](https://i.loli.net/2020/03/09/MquXNHZ3vcwjAsV.png)
3. 安装完成之后输入1回车，进行安装配置ShadowsocksR

①选择端口，博主这里设置的是8080，（据说配合混淆参数可以实现免流，但现在好像是不可以了）
![选择端口](https://i.loli.net/2020/03/09/jvRr4lx1qCtsaZA.png)
②设置密码
③设置加密方法
④设置协议，选择插件兼容原版填入y
![进行配置](https://i.loli.net/2020/03/09/JsV81SMkQr6jZuX.png)
⑤设置混淆插件，同样填入y选择兼容原版（③④⑤博主都是直接默认回车）
⑥⑦⑧配置端口和速度直接默认回车，当然也可以按自己的想法来设置

之后填入y开始安装，安装过程稍慢，需要等一等
![进行安装](https://i.loli.net/2020/03/09/DRO8EugFMqNsf7e.png)
安装成功！
![安装成功](https://i.loli.net/2020/03/09/gHDKkfMOTW7cyQt.png)
安装成功后就可以复制配置信息中的ss/ssr链接或二维码进行连接开始我们的FQ之旅

## 补充：BBR加速SS/SSR

1. ssr.sh脚本内置bbr加速

2. 四合一脚本

    ```bash
    wget -N --no-check-certificate https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh && chmod +x tcp.sh && sudo bash tcp.sh
    ```
