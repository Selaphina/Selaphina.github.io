---
title: Planar阴影系统的实现
description: 将 caster 的顶点沿光照方向投影到一个平面（plane normal + plane height）上，生成“扁平化”的投影几何来模拟阴影。
date: 2025-11-14 10:12:30+0000
image: CUC兑换点2.png
categories:
    - 技术笔记
weight: 1998       # You can add weight to some posts to override the default sorting (date descending)

---

##   实现原理（PlanarShadowSystem）

  - 基本思路：将 caster 的顶点沿光照方向投影到一个平面（plane normal + plane height）上，生成“扁平化”的投影几何来模拟阴
    影。
  - 投影发生在顶点着色器 src/game/systems/planar-shadow/shaders.ts：
      - 根据 u_lightDir 和 u_planeNormal 求射线与平面的交点。
      - u_planeHeight 设定平面高度（目前来自 planeHeight + planeHeightOffset）。
      - 使用 u_planeBias 做轻微的 z-bias，防止 z-fighting。
  - 片元着色器输出固定色 u_shadowColor，不采样纹理，不做软边。
  - Stencil（可选）：将 receiver 写入模板缓冲，shadow 只绘制在 receiver 上，避免重叠变黑或穿帮。

## 工作流程

1. 配置入口
     src/config/data/shadows.json 里的 planarShadow 控制是否启用、平面法线/高度/偏移/颜色等。
  2. 初始化入口
     src/game/App.ts 在 preload() 中调用 initPlanarShadowSystem()。
  3. 构建系统配置
     initPlanarShadowSystem() 读取 planarShadow，拼成 PSPlanarShadowConfig：
      - plane.height = planeHeight + planeHeightOffset（你要调的是这里）
      - plane.normal、plane.bias、appearance.color、stencil
      - excludePatterns 合并默认和 shadowMeshes.json 的排除项
      - 注册 shader
      - 需要时启用 stencil
      - 创建 base shader material
      - 在 onBeforeRender 每帧更新 uniform（lightDir/planeNormal/planeHeight/bias/color）
  5. 添加 caster/receiver
      - Player 模型在 initPlanarShadowSystem() 中手动遍历 child mesh 加 caster
      - 自动检测：enableAutoDetection() 按 name pattern 自动加入
      - Receiver：applyPlanarShadowReceivers() 从 shadowMeshes.json 匹配网格名
  6. 渲染阶段
      - 影子网格是原网格 clone 的几何体（或骨骼 mesh 的特制材质）
      - 渲染组/深度/模板设置确保阴影在 receiver 上、被主体遮挡

  ———

## 开放控制点

  - planeHeight / planeHeightOffset：阴影平面高度（位置不对主要调这个）
  - planeBias：投影深度偏移（防止 z-fighting）
  - planeNormal：平面朝向
  - shadowColor.a：阴影强度
  - useStencil：是否启用 stencil 遮罩

# 附录

## 漂浮在所有物体上的原因

```
平面阴影的渲染层级（RenderingGroupId）改回了默认值（0）。

之前的设置（1）可能导致阴影强制绘制在所有物体（层级0）之上，造成“漂浮在桌子上”的错觉（实际上是因为深度遮挡失效，阴影盖过了桌子）。

现在阴影应当能正确被桌子、收银台等物体遮挡（在它们下方）。请刷新查看效果。
```



## stencil 遮罩有什么作用？

stencil 遮罩的作用是“限制阴影只画在指定接收面上，并避免重叠阴影叠加变黑”。具体在这套实现里有两点：

  - 只在接收面上显示：receiver 先把模板缓冲写成 1，shadow pass 只在 stencil == 1 的像素上绘制，防止阴影投到不该有阴影的
    地方（比如角色身上或空中）。
  - 避免叠加过黑：通过模板逻辑控制“同一像素只算一次阴影”，让多个 caster 重叠时不无限叠加变黑（更接近 Cocos 的 planar
    shadow 表现）。

  如果不启用 stencil，阴影会渲染到所有通过深度测试的位置，看起来更容易“漂浮/穿帮”。

## 投下的阴影属于网格体？

每个 caster 会 clone 一份 shadow mesh（_planarShadow 后缀），挂在一个 shadow root 下，使用专用的 planar shadow shader；真正绘制的是这份投影网格，而不是原始网格本身。

## shadow mesh会和原始的网格重叠/导致游戏运行实时面数增加吗


  - 顶点/三角形数增加：每个 caster 多一份几何，等于多画一次（含骨骼的话更贵）。
  - draw call 增加：影子网格是独立材质/渲染组，会多一次绘制。
  - CPU 也有成本：每帧更新 uniform、跟随变换、管理 caster/receiver。

  - 减少 caster 数量、排除小物体（excludePatterns/minVolume）。
  - 只给关键角色投影。
  - 关闭 stencil（略省但可能穿帮）。

## 对比传统 shadow mapping（深度贴图）

###   优势

  - 更省性能：不需要渲染光源视角的 shadow map，也无需采样深度贴图。
  - 稳定、无闪烁：没有阴影贴图分辨率/抖动问题，移动时也更“干净”。
  - 控制简单：直接用高度/偏移/颜色调效果，适合移动端或风格化项目。

###   区别 / 局限

  - 只适用于平面或近似平面：阴影被投到一个固定平面，无法贴合起伏地形或复杂表面。
  - 不能产生真实遮挡关系：不会被其他物体遮挡，也没有自阴影。
  - 阴影形状简单：本质是投影几何体的“扁平投影”，没有软硬边真实变化。
  - 精度与物理真实性较弱：只适合“地面贴影子”的视觉提示用途。

  一句话：planar shadow 是“便宜稳定的视觉提示”，shadow mapping 是“真实但更重、更复杂”的通用方案。