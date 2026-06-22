---
title: UE常见组合技列表！（一）
description: 
date: 2126-02-19 10:20:12+0000
image: CUC兑换点2.png
categories:
    - 工具指南
weight: 2029       # You can add weight to some posts to override the default sorting (date descending)
---

# UE常见组合技列表

## 常见



## 附录

### TexCoord

**标准范围是 [0, 1]**

**原点 (0,0)**：位于纹理的左下角。

**终点 (1,1)**：位于纹理的右上角。

**注：**超出 [0, 1] 的情况（纹理平铺/重复）

当你在 TexCoord 节点中设置 **UTiling** 或 **VTiling > 1** 时，范围会变大。

- 例如：设置 **UTiling = 2**，则 U 轴的范围是 **[0, 2]**。
- 此时，如果你连接一张贴图，默认会看到纹理在水平方向重复了 2 次。