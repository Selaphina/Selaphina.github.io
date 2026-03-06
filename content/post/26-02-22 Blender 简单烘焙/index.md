---
title: Blender烘焙
description:
date: 2026-02-19 10:20:12+0000
image: cover1.png
categories:
    - 工具指南
tags:
    - Obsidian
weight: 1999       # You can add weight to some posts to override the default sorting (date descending)
---

出于将程序化纹理烘焙到贴图上导出的目的，记录插件simplebake的使用方法。

新建UV map，命名改为simplebake（大小写不敏感；不能有空格）

![](image-20260222225557548.png)

进入编辑模式；全选面；快捷键U；

![](image-20260222225411516.png)



注意，simplebake插件在右侧渲染的面板上。

1.物体模式；选中需要的物体；（先清空）添加；

![](image-20260222230439936.png)

2.漫射/金属度/粗糙度/法线比较常用（若都勾选上则输出4张图）

![](image-20260222231107650.png)

3.（可选）这里可以输出AO。一般用不着勾选。

![](image-20260222231301468.png)

4.尺寸1k/2k/3k/4k

![](image-20260222231330960.png)

5.填写导出位置

![](image-20260222232046819.png)

6.注意不要勾选UDIMs，勾选以下的两个蓝色部分

* prefer existing…意思是优先烘焙到名为simplebake的贴图上
* restore original UVs：意思是烘焙结束后仍然恢复到原UV的状态（而不是覆盖原uv）

![](image-20260222232145978.png)

7.选中前景色；bake

![](image-20260222232643177.png)

注意：烘焙需要时间，在烘焙过程中不要操作blender。
