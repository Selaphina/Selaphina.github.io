---
title: Blender-导出fbx动画
description: 需求：模型减面+fbx动画导出。
date: 2025-11-22 10:20:12+0000
image: cover1.png
categories:
    - 技术笔记
tags:
    - Blender
weight: 2038       # You can add weight to some posts to override the default sorting (date descending)
---

## 1. Blender导出带贴图模型

{{BV1384y1j7wt}}

1.文件—— 外部数据 ——打包资源——（第二项）将文件写至当前目录

![image-20251123213913006](image-20251123213913006.png)

2.着色器编辑器：

![image-20251123213936700](image-20251123213936700.png)

所有shader节点的贴图文件处，都需要：解包项——将文件写至当前目录（覆盖现有文件）

![image-20251123214055334](image-20251123214055334.png)

操作之后，确保图标变化：

![image-20251123214149107](image-20251123214149107.png)

3.导出——fbx

![image-20251123214226752](image-20251123214226752.png)

两处修改：

路径模式——复制

右侧小图标——激活

1.![image-20251123214312173](image-20251123214312173.png)

2.



## 形态键动画导出

1.分为两个动画：

动作动画

形态键动画

![image-20251123221119471](image-20251123221119471.png)

要求：

动作动画长度一定要>=形态键动画

导出：

1.![image-20251123223229361](image-20251123223229361.png)

2.![image-20251123223258583](image-20251123223258583.png)