---
title: Hugo：毕业设计 | 水墨风格化场景渲染（其一）
description: 毕业设计的开发日志，用于实时记录学到的trick。
date: 2025-11-11 10:12:30+0000
image: cover.jpg
categories:
    - 技术笔记
tags:
    - Note
weight: 2030       # You can add weight to some posts to override the default sorting (date descending)

---

## 1.插件：UnrealToUnity

在日常创作中，经常碰到一些好的UE美术资源，可以使用这个插件把它转换到Unity当中。

[百度网盘链接](https://pan.baidu.com/s/1P5AMJ177SIdE4-fjiRFEXg?pwd=6666)

![img](v2-c29b95f89891b425695cec5e69071dbd_1440w.jpg)

Unity：2019以上的版本

UE：版本根据自己的安装插件对应的版本自行选择，我选择的是5.3.2

**步骤：**

1. **解压对应版本的UE插件：**

![压缩包内部结构](image-20251111150443177.png)

2. **新建一个UE工程。**

![](image-20251111150049173.png)

找到工程目录，新建一个Plugins文件夹，如下：

![](image-20251111150636303.png)

3. **把解压后的插件文件夹整个复制到Plugins文件夹下。**

![](image-20251111150754424.png)

4. **重启UE，插件会在菜单栏上显示，可直接点击使用。**

![](image-20251111150908079.png)

把需要转成unity资产的ue美术素材uasset拖到世界中，即可。

![](image-20251111150924765.png)

默认设置即可导出。可以选择取消勾选shader等，因为会与unity本体的shader有冲突。

![image-20251111151019722](image-20251111151019722.png)

> 事实上，导入unity后大概率会进入报错的安全模式，按照console窗口的报错一步步注释掉UE转生过来的代码即可，很快就能清理完毕。另外，还是强调，尽量把ue带来的shader删除干净，在unity里重新写一遍shader。

## 2.