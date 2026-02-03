---
title: Unity MCP 全流程
description:
date: 2026-02-04 10:20:12+0000
image: cover1.png
categories:
    - 技术笔记
tags:
    - Unity
	   - AI Agent
weight: 2000       # You can add weight to some posts to override the default sorting (date descending)
---

## 1. Blender导出带贴图模型

Unity MCP（Model Context Protocol）正在重构开发者的工作流。它像一座桥梁，让 AI 助手能 "读懂"Unity 项目的上下文 —— 从场景结构到脚本逻辑，从资源属性到运行时数据，从而实现智能代码生成、场景优化建议、角色行为设计等高效协作。

想象一下：当你在 Unity 中选中一个卡顿的场景，AI 能通过 MCP 直接分析 Draw Call 数据并给出光照烘焙优化方案；当你设计 NPC 行为时，AI 能基于当前动画控制器结构生成状态机过渡代码。这些并非科幻，而是通过 Unity MCP 配置就能实现的开发体验。
