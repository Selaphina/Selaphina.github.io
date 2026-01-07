---
title: 图集打包器
description: 图集（Atlas）是将多个小纹理打包成一张大图（制作图集），是优化游戏性能非常关键的一步。
date: 2025-11-14 10:12:30+0000
image: CUC兑换点2.png
categories:
    - 工具指南
weight: 1997       # You can add weight to some posts to override the default sorting (date descending)

---

**图集（Atlas）**：将多个小纹理（如UI图标、道具图片）打包到一张大图中。这样，渲染使用同一图集的不同精灵时，可以被合并到一个Draw Call中完成。

**注意渲染顺序**：尽量让使用相同材质的节点在场景树中连续排列，避免不同材质的节点插入其中打断合批



## 需求：是否可以项目自动化打包图集

 工具推荐

  - free-tex-packer-cli（开源、易集成、支持常见 JSON 格式）
    适合自动化打包，产出 atlas.png + atlas.json。
  - 备选：TexturePacker（商业，功能强）

  我可以写一个简单的自动化脚本
  比如用 free-tex-packer-cli：

  - 输入：src/assets/ui/**
  - 输出：src/assets/atlases/ui.png + src/assets/atlases/ui.json
  - 加 npm script：npm run atlas:ui

  我需要你确认这几点再动手（也需要你允许我写文件/装依赖，因为当前是只读）：

  1. 资源目录和输出目录分别是？
  2. 你需要的 JSON 格式（TexturePacker/Pixi/Hash）？
  3. 是否要裁边、去透明边、强制 2 的幂尺寸、最大纹理尺寸？
  4. 你会用在 Babylon 的 SpriteManager/SpritePackedManager 还是给网格手动 UV？

## 1.SpriteManager/SpritePackedManager是什么

简短说法：它们都是 Babylon.js 里管理“二维精灵”的工具。

  - SpriteManager：用一张 spritesheet 贴图 + 固定行列布局（等宽等高的格子）。你用 index 选格子。
  - SpritePackedManager：用“图集 + JSON”的打包结果（每个子图大小不一），根据 JSON 里的 UV/尺寸来取图。

  如果你的图集是自动打包出来的（子图大小不一），就用 SpritePackedManager。如果是规则网格的 spritesheet，就用
  SpriteManager。



## 2. JSON 格式（TexturePacker/Pixi/Hash）之间有什么区别?

主要区别在 JSON 的结构和字段命名，内容大体相同（每张子图的 x/y/w/h、是否旋转、裁剪信息等）：

  - TexturePacker JSON (Array)
    frames 是数组；每个元素里有 filename 和 frame（x,y,w,h），还包含 rotated/trimmed/spriteSourceSize/sourceSize。
  - TexturePacker JSON (Hash)
    frames 是对象（key=文件名），值里是 frame 等信息。数据同上，只是组织方式不同。
  - Pixi JSON
    结构更偏 Pixi 生态，frames 通常是对象，meta 里会写 app: "Pixi"、scale 等。字段名和 TP 接近，但细节/命名可能略不同
    （尤其 meta）。

  简单选型：

  - 你要兼容 Babylon 的 SpritePackedManager，更稳的是 TexturePacker 的 JSON（Array 或 Hash）。
  - Pixi 格式更适合 Pixi.js 生态。

## 3.【是否要裁边、去透明边、强制 2 的幂尺寸、最大纹理尺寸】分别有什么用？

它们都是为了减少纹理浪费、提高兼容性/性能：

  - 裁边（trim）
    去掉每个子图四周的透明像素，只保留有内容的最小矩形。好处是图集更紧凑、尺寸更小。
    注意：会改变子图原始尺寸，需要在 JSON 里保存偏移/原尺寸，渲染时要“还原”。
  - 去透明边（extrude/bleed/alpha bleed）
    在子图边缘向外扩展一圈像素（用边缘颜色填充），避免采样时因为双线性过滤/缩放出现“漏白/黑边”。
    常用于图集防止“边缘渗色”。
  - 强制 2 的幂尺寸（POT）
    把整张图集尺寸调整为 2 的幂（比如 512/1024/2048）。
    好处：兼容一些旧设备/管线，MipMap、压缩纹理更友好。
    代价：可能变大、浪费空间。
  - 最大纹理尺寸
    限制图集最大边长（如 2048/4096）。
    好处：避免超出 GPU 上限（移动端常见限制），也便于性能/内存控制。
    代价：可能拆成多张图集。

### 移动端默认参数

```
 - trim: true
  - extrude/bleed: 2 px
  - padding: 2 px
  - pot (power-of-two): true
  - maxSize: 2048
  - allowRotation: false（避免旋转带来的采样/UV 心智负担）
  - scale: 1
```

### 桌面PC端

桌面/WebGL2，也可以把 maxSize 调到 4096，或把 pot 设为 false 来减少浪费。



## 资源目录和输出目录分别怎么写比较好？

• 常见、清晰的结构是：

  - 资源输入（原始小图）：src/assets/ui/atlas-src/
  - 输出图集：src/assets/atlases/ui/（产出 ui.png + ui.json）

  理由：原始素材和生成物分开，后续脚本清理、重建都更安全；图集集中在 atlases/ 便于引用和缓存。

  如果有多套图集（UI/FX/Icons），可以再分子目录：

  - src/assets/ui/atlas-src/
  - src/assets/fx/atlas-src/
  - src/assets/atlases/ui/
  - src/assets/atlases/fx/

  你希望只做一套 UI 图集，还是要按类型拆多套？