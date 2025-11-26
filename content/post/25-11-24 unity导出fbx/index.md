---
title: unity的anim格式的动画转成FBX
description: 开发过程中的格式转换问题
date: 2025-11-14 10:12:30+0000
image: CUC兑换点2.png
categories:
    - 工具指南
tags:
    - Note
weight: 2040       # You can add weight to some posts to override the default sorting (date descending)

---

要把 **Unity 的 `.anim` 动画文件转换成 FBX 内嵌动画**，可以通过 Unity 官方提供的工具 **FBX Exporter** 来完成。操作流程如下：

------

## ✅ **方法 1：使用 Unity FBX Exporter（推荐）**

Unity 的 FBX Exporter 支持把 Animator Controller 或 AnimationClip 导出成带动画的 FBX。

### **步骤：**

1. **安装插件**

   ![image-20251126184816021](image-20251126184816021.png)

   - Unity → **Window → Package Manager**
   - 搜索：**FBX Exporter**
   - 点击 **Install**

2. **准备模型和动画**

   ![image-20251126184929745](image-20251126184929745.png)

   - 确保模型（带骨骼）已绑定 AnimationClip（`.anim`）
   - 模型需要有 **Animator 或 Animation Component**

3. **右键模型 → Export to FBX**
    !> 必须确保动画能在 Inspector 中 Preview 播放，否则无法导出

4. 在导出窗口中勾选：

5. 

   ```
   Export Animations: ✔
   Bake Animation: ✔
   Sample Rate: 30 或默认
   ```

6. 导出完成的 `.fbx` 就会包含动画，外部软件（Blender / Maya / MotionBuilder）可以直接看到。
