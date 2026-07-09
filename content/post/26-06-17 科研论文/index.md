---
title: 科研论文自留文档
description: 
date: 2126-02-19 10:20:12+0000
image: CUC兑换点2.png
categories:
    - 工具指南
weight: 2019       # You can add weight to some posts to override the default sorting (date descending)
---

# 科研论文自留文档

本文用作自留档常用的科研论文阅读文献网站/软件/DDL等教程。

* DDL：https://ccfddl.com/
* CVPR、ICCV、WACV论文检索：https://openaccess.thecvf.com/

## 论文检索

### **CVPR、ICCV、WACV论文检索：**

https://openaccess.thecvf.com/

* ### 操作步骤

* 进入

```
https://openaccess.thecvf.com/
```

* 点击：

```
CVPR 2026
CVPR 2025
ICCV 2025
ECCV 2024
```

* 然后使用页面搜索：

```
Color
Color Grading
LUT
Look-Up Table
Video Color
Video Editing
Video Stylization
```

-- 推荐搜索词：

```
Color Transfer

Color Mapping

Color Enhancement

Color Harmonization

Image Retouching

Photo Enhancement

Video Enhancement

Video Stylization

Video Editing

LUT

3D LUT

4D LUT

Look-Up Table

Color Consistency

Temporal Consistency
```

### Papers With Code

进入：

```
https://paperswithcode.com/
```

## 开源算法试运行

### 长期更新的github代码与老版本checkpoint问题

> 前提：我在实验室新主机上跑一个8个月前试运行过的开源项目，沿用的是8个月前的checkpoint。然而，作为iccv 2025的夯作在此期间进行了数次更新，因此绝不可以用过去下载好的.ckpt

报错几百行（截取）：

> ```
> RuntimeError: Error(s) in loading state_dict for UNet2DConditionModel
> ```
>
> 这个报错**不是 Diffusers 版本的问题，也不是 PyTorch 的问题**，而是**模型权重和UNet网络结构完全对不上**。

并且同时出现了三类典型错误：

1. **Missing key(s)**（缺少几百个参数）
2. **size mismatch**（参数维度不一致）
3. **stable-diffusion-v1-5 初始化成功，但是随后加载 checkpoint 失败** 

有几种可能：

1. checkpoint下载错了。（比如把L_Diffuser.ckpt下成了stable-diffusion-v1-5等等）
2. GitHub代码更新了、但是：checkpoint还是老版本。

3. 没看README：`git clone --recursive`,而直接：`git clone`

## 下载Google Drive模型

在国内（或者学校实验室服务器在国内），从 Google Drive 下载 **十几二十GB** 的模型权重，直接浏览器下载往往只有几十 KB/s 到几 MB/s，甚至中途断掉。

### gdown

安装：

```
pip install gdown
```

然后：

```
gdown https://drive.google.com/uc?id=FILE_ID
```

例如：

下载单个文件：

```
gdown https://drive.google.com/uc?id=xxxxxxxx -O model.ckpt
```

-O model.ckpt 表示：

> 下载后保存为 `model.ckpt`。

下载多个文件：

```
gdown --folder --continue -O checkpoints "https://drive.google.com/drive/folders/1GX3Q0kti6WpmZPKdzToR2sv9qEps6cmK"
```

`--folder`：告诉 gdown 这是一个**文件夹**而不是单个文件，这是必须的。

`--continue`：支持下载中断后继续。

`--fuzzy`：对于**新版 gdown（v6.x）已经不需要了**，因为现在会自动识别各种 Google Drive 链接格式。

`-O` 是 **Output（输出）** 的缩写，用来指定**下载后的保存位置**。

-O checkpoints 表示：

> **把整个 Google Drive 文件夹下载到当前目录下的 `checkpoints/` 文件夹中。**

### 国内镜像

例如：

- ModelScope（魔搭）
- OpenXLab
- 百度网盘（部分作者提供）
- 清华云盘
- 阿里云OSS

速度通常远高于Google Drive。