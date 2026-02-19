---
title: Obsidian多端同步
description:
date: 2026-02-19 10:20:12+0000
image: cover1.png
categories:
    - 工具指南
tags:
    - Obsidian
weight: 1999       # You can add weight to some posts to override the default sorting (date descending)

---

c

春节假期间，更替了一下笔记软件。主要用于教程指路：
https://zhuanlan.zhihu.com/p/586431408

## 电脑端：

整体需要：

* 安装<u>坚果云</u>

* Obsidian安装Remotely Save插件

### 1.安装 Remotely Save

在社区插件市场搜索“Remotely Save”，安装并启用该插件。

![](image-20260219223218298.png)

### 2.坚果云

打开[坚果云](https://link.zhihu.com/?target=https%3A//www.jianguoyun.com/)，登录/注册账号。 创建一个个人同步文件夹，注意**文件夹名称要和被同步的 Obsidian 库的库名一致**。

比如，我的Obsidian 库名为：MyCalendar，那么新建的同步文件夹也要命名为：MyCalendar。

![](image-20260219225335753.png)

设置—— 第三方应用管理——添加应用密码

![](image-20260219223001298.png)

![image-20260219223026973](image-20260219223026973.png)

复制生成的密码。（也可以在刚刚建立的应用处点击“显示密码”来查看密码）

### 3 设置 Remotely Save

进入 Remotely Save 的设置界面，选择远程服务为 Webdav。

Webdav 设置中的“服务器地址”、“用户名”、“密码”分别对应输入刚刚“用户信息-安全选项-第三方应用管理”界面中的“服务器地址”、“账号”、“应用密码”（刚刚复制的密码）。

![](image-20260219225601456.png)

![](image-20260219225712476.png)

设置好后点击“检查”，测试服务器连接状况。

参考设置：

![image-20260219225824691](image-20260219225824691.png)

## ipad端

Obsidian 库就是一个文件夹，直接把它打包（必须zip）发送到移动端，然后解压——到移动端的obsidian打开即可。

打开移动端 Obsidian，选择“Open folder as vault”，选择刚刚解压的文件夹作为 Obsidian 库，然后 Obsidian 会自动进入主界面。

点击“同步”，测试服务器连接成功！

![](image-20260219231252346.png)
