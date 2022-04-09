---
title: Chevereto-在服务器上搭建自己的图床
categories: 不正常的文章
date: 2018-08-09 15:43:03
---

![](https://lolico.griouges.cn/images/h4J.png)
## 介绍
 - 官网:[chevereto.com][2]
   github项目:[Chevereto-Free][3]
 - Chevereto 是一款采用 PHP 语言开发的网络相册脚本程序，支持多语言，提供中文语言包的下载的开源在线图片存储分享服务系统，支持本地上传和在线获取两种图像上传方式，并集成了 TinyURL 网址缩短服务。
 - Chevereto 这套程序可以像 Discuz 或 WordPress 一样随意架设在任何空间上。而它的功能除了一般图片空间单纯的从电脑上传图片外，也支援利用网址也可以上传，最屌的是还有 TinyURL 的缩短网址的功能可以使用，因此这套 Chevereto 可以说是比市面上一些免费图床好太多了。

## 安装
> 目前Chevereto有免费版和付费版两种,且当前最新版本已经支持PHP7,但是PHP7已经不支持mysql扩展,所以建议不要使用PHP7,使用PHP5.5,PHP5.6即可

 - ~~下载源码后解压并放在网站根目录,官网和github下载很慢,这里提供一个: [chevereto-3.10.13.zip][4]~~
 - 打开你的网站,若网站提示你`Chevereto can’t create the app/settings.php file. You must manually create this file.`在app/目录下创建一个setting.php文件即可.
 - 刷新网页,开始安装,安装界面是英文的,英语不差到离谱应该还是能够看懂的,填入对应数据库信息,登陆后台账号密码等就可以开始安装了.
 - 安装可能还会提示你某些文件夹权限不够或者setting.php文件无法写入,可以修改对应文件夹权限,或是自己在setting.php中写入安装程序提示你的代码即可继续安装.

*上述步骤完成后,我们的图床便搭建好了,现在就可以登录后台对一些基础功能进行设置和修改.*

## 演示站点
 - <a href="https://img.griouges.cn" target="_blank">img.griouges.cn</a>



  [2]: https://chevereto.com/
  [3]: https://github.com/Chevereto/Chevereto-Free
  [4]: https://griouges-1257226137.cos.ap-shanghai.myqcloud.com/usr/uploads/2018/08/1713663965.zip