---
title: 交互雪梳理笔记
description: 
date: 2126-02-19 10:20:12+0000
image: CUC兑换点2.png
categories:
    - 工具指南
weight: 1997       # You can add weight to some posts to override the default sorting (date descending)
---

# 虚幻引擎

本文用作记录UE5制作交互雪的基本流程，以及需要注意的事项。

```
UE版本：5.7
IDE：Visual Studio 2022
操作系统：windows 11
```

## 1.制作画板

画板需要：

- BP_画板`蓝图类（Blueprint Class）`

  - 作为场景中的交互主体；继承自Actor，包含Scene Component作为根组件

- RT_Snow `Render Target`

  - 分辨率：根据需求设置（如1024 x 1024）

    用作雪地状态的“画布”，实时存储雪的高度/凹陷信息；初始化为白色（代表未踩踏的雪）

* M_Snow `Material`

  * 主材质，应用在雪地网格上

    通过材质函数读取RT_Snow的实时数据

    实现视觉表现：雪的高度、凹陷处的阴影/颜色变化



### 1.BP_画板

![](image-20260328185819328.png)

![](image-20260328185549074.png)

其中包括：

【世界位置toUV】设置为纯函数。

![](image-20260328185653573.png)

### 2.M_Snow

![](image-20260328190324812.png)

首先规定一个2000 x 2000 世界空间尺寸大小的雪地空间。



## 2.映射位置

需要新增变动：

* MPC_Snow `材质参数集（Material Parameter Collection）`
  * 用于向蓝图类传递材质参数Position、size

* MF_World2SnowUV`Material Function`
  * 用于将世界位置(x,y)转换为UV[0,1]
* BP_画板：新增Plane组件；新增变量Size、Position
  * 用于显示雪地的范围

### 1.MPC_Snow

![](image-20260328194643558.png)

用4D向量存储PositionSize，所以：

![MF_World2SnowUV](image-20260328195700862.png)

![BP_画板](image-20260328195833201.png)

### 2.MF_World2SnowUV

`Material Function`

![](image-20260328190805313.png)

`Material`

![](image-20260328191004511.png)

### 3.BP_画板

画板中的函数【世界位置toUV】和材质函数类似：

![](image-20260328193536626.png)

UV = （WorldPos - Pos）/ Size + （0.5, 0.5）

（先减后乘）

### 4.画板新建Plane

![](image-20260328193928048.png)

如图，在构造函数图里，针对plane的位置进行映射。注意，材质为简单的（线框模式+自发光）。

*务必记得把plane设置成no collision（无碰撞）

## 3.走向无限

需要新增变动：

* M_平移：
  * Offset：存储上一帧到这一帧的位置。
  * MF_currentUV2Previous:计算位移差。

* BR_画板：
  * 新增变量Lastposition
  * 新增函数【更新位置】
  * 在tick中创建动态材质实例并计算offset
  * 新增【复制RT】的函数
* 新增RT_SnowSave
* 新增M_平移
  * 防止雪地轨迹重复平铺
  * 防止雪地轨迹模糊（像素对齐）

### 1.M_平移

![](image-20260328204429429.png)

*注意：

`Floor`：向下取整。砍掉小数部分，保留整数。

```
输入 2.7→ 输出 2.0
输入 0.8→ 输出 0.0
```

` Abs:`绝对值

`Saturate`:钳制到0-1：将数值限制在 `[0, 1]`区间。

```
输入 -0.5→ 输出 0.0
输入 1.2→ 输出 1.0
```

`1-x`:（One Minus，1 减）

```
输入 0.0→ 输出 1.0
输入 1.0→ 输出 0.0
```

`U multiply V`: 

总结整个链路流程：当且仅当u和v在[0,1）时，输出的值（alpha）为1，其余为0.

`MF_currentUV2Previous`:

![](image-20260328204504139.png)



### 2.BP_画板

![](image-20260328201753328.png)

#### 在tick中计算材质实例的Offset :

![BP：创建动态材质实例传递offset](image-20260328204132256.png)

#### 更新位置函数

![](image-20260328204902024.png)

![总预览](image-20260328205039292.png)

#### 复制RT: 

![将当前帧 RT_Snow保存至RT_SnowSave上](image-20260328205636300.png)

#### 像素对齐：

在更新位置的函数中：
![](image-20260328212642732.png)

Size：Plane的尺寸

dpi：分辨率，即RT的尺寸（如1024 x 1024）

## 4.大地画板

需要：

* 新建BP_Brush
* 新建S_BrushInfo（Blueprint——Structure）
* 

### BP_Brush

新建Blueprint class，选择Scene Component。（创建后也可以在Class Settings里右侧栏的【Parent Class】改）

![](image-20260328220641754.png)

![](image-20260328220618873.png)

#### S_BrushInfo

![](image-20260328215459290.png)

Position：3D向量

SIze：2D向量



![折叠函数](image-20260328213724846.png)

