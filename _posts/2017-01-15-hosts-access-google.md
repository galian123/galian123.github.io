---
layout: post
title: 修改hosts文件访问Google网站，下载Android代码
date: 2017-01-15 07:09 +0800
categories: 访问Google
tags: hosts
---

一个良心的网站：[laod.cn](https://laod.cn/)

# 修改hosts，访问Google网站，下载Android代码

* **第一步：获取hosts文件**

  去这个位置获取： [2017 Google hosts 持续更新](https://laod.cn/hosts/2017-google-hosts.html)

* **第二步：修改本机中的hosts文件**

  不同系统hosts文件的位置：（摘自laod.cn）

  <!--more-->

  ```
  Windows 系统hosts位于 C:\Windows\System32\drivers\etc\hosts
  Android（安卓）系统hosts位于 /etc/hosts
  Mac（苹果电脑）系统hosts位于 /etc/hosts
  iPhone（iOS）系统hosts位于 /etc/hosts
  Linux系统hosts位于 /etc/hosts
  绝大多数Unix系统都是在 /etc/hosts
  ```

> **注意： 若更新后，hosts 没有立即生效，请重置网络：
在系统设置内开关网络，或启用禁用飞行模式，或者重启、刷新DNS缓存、浏览器缓存。**

将下载到的hosts中的内容更新到你的的hosts中。

经验证，下载Android代码时速度还是很快的。

--------------分割线--------------

这篇blog原本发布在CSDN，无奈被CSDN删除，原因大家都懂的。

如果不懂，可以看看这个新闻：**谷歌是在封网站，能连就不正常**

![]({{ site.url }}/images/aliyun_reply.jpg)

没有责怪CSDN和阿里云的意思，大环境如此，都是为了生存。



在CSDN未能发出的blog：

被放到了回收站：

![]({{ site.url }}/images/deleted_by_csdn.jpg)

显示已删除：

![]({{ site.url }}/images/deleted_blog_about_hosts.jpg)
