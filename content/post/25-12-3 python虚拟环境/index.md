---
title: python虚拟环境
description: 在服务器上如何安装python虚拟环境
date: 2025-11-14 10:12:30+0000
image: CUC兑换点2.png
categories:
    - 工具指南
weight: 2030       # You can add weight to some posts to override the default sorting (date descending)

---

## 方法1：使用conda（推荐）

### 1) 创建一个新环境并安装Python 3.10.16
```
conda create -n py310 python=3.10.16
```

### 2）激活新环境
```
conda activate py310
```

### 3）验证版本
```
python --version
```

