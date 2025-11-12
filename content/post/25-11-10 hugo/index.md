---
title: Hugo：个人博客建站指南
description: 通过Hugo构建的静态博客，以stack主题为例。
date: 2025-11-09 10:12:30+0000
image: cover.jpg
categories:
    - 工具指南
tags:
    - Hugo
weight: 2043       # You can add weight to some posts to override the default sorting (date descending)

---

作为Hugo建站新手，使用作者提供的 **Stack starter template**，并用 **GitHub Actions** 自动构建并部署到 GitHub Pages，最为方便。

> 建站之初参考了很多博主的指南博客，都是手工方式在本地安装Hugo，把stack主题拉取到本地再部署到github上。然而实际操作过程中总有和博主相悖的情况发生，遂找G老师从头复盘一遍。最后一遍过。
>
> Todo：补充自备案域名的建站方式。

## 一、先决条件

1. 一个 GitHub 账号（有仓库权限）。
2. 安装 Git（本地开发时需要）。

> 小提示：如果只是用 starter 模板并通过 GitHub Actions 构建部署，**本地安装 Hugo 可以暂时跳过** — 但建议安装以便本地预览与调试。

3.从Hugo资源页[Releases · gohugoio/hugo](https://github.com/gohugoio/hugo/releases)下载amd版本（查看自己的电脑为**`基于 x64 的处理器`** -> **下载 `amd64`版本。**）

![image-20251107154753124](image-20251107154753124.png)

由于部分主题需要 Hugo 的 Extended 版本才可以正常使用，因此建议**一步到位直接安装 Extended 版本**的 Hugo，在将压缩包解压后一定不要忘记的是将 `hugo.exe` 所在的文件夹添加至用户的**[环境变量](https://zhida.zhihu.com/search?content_id=165841755&content_type=Article&match_order=1&q=环境变量&zhida_source=entity)**。

若是以上步骤都正常完成，那么可以在输入 `hugo version` 命令后得到正常的版本号显示。

![image-20251107154836195](image-20251107154836195.png)

4.初始化

```
hugo new site XXX
```

5.快速预览

> 浏览 http://localhost:1313 查看效果。注意：-D的意思是同时预览草稿。

```
hugo server -D
```

在浏览器中访问http://localhost:1313/ ，如果正常就会显示出页面。



## 二、用 Hugo-Theme-Stack Starter（适合新手）

Stack 作者维护了一个 starter 模板并自带 GitHub Actions 来部署，直接用模板能省很多配置工作。示例仓库与 workflow 可参考作者的 starter。[Hugo-Theme-Stack](https://github.com/CaiJimmy/hugo-theme-stack-starter)

### **步骤：**

#### 1.**在 GitHub 上用模板创建仓库**

- 打开 `CaiJimmy/hugo-theme-stack-starter` 仓库（或访问主题主页找到 starter 链接），点击 **Use this template → Create a new repository from template**。选择仓库名：
  - 如果你想把博客作为个人主页（username.github.io），仓库名应为 `your-github-username.github.io`（这样 GitHub Pages 会直接在根域名托管）。
  - 如果想作为项目页（例如 `your-username/myblog`），可以用任意仓库名（但发布 URL 会是 `https://username.github.io/repo`）。
- （参考：starter 仓库自带 Actions 和样例配置。）[GitHub](https://github.com/CaiJimmy/hugo-theme-stack-starter?utm_source=chatgpt.com)

#### 2. **把仓库 clone 到本地（可选但推荐）**

```
git clone https://github.com/your-username/your-repo.git
cd your-repo
```

#### 3. **修改站点配置（最重要的两项）**

- 打开仓库中的 `config.toml`（或 `config/_default/config.toml`，starter 可能把配置拆分），修改：
  - `baseURL`：部署到 GitHub Pages 时必须设置为你站点的真实 URL，例如：
    - 个人主页（username.github.io）: `https://your-username.github.io/`
    - 比如我：[Selaphina](https://selaphina.github.io/)
    - 项目页：`https://your-username.github.io/repo-name/`
  - `title`、`author` 等按需修改。
- Starter 通常已经把 theme 配好为 Hugo module（不用把主题源码放到 `themes/`），直接修改 baseURL 即可。参考 Stack 文档或 starter README。[GitHub](https://github.com/CaiJimmy/hugo-theme-stack?utm_source=chatgpt.com)

#### 4.**启动本地预览**

```
hugo server -D
# 浏览 http://localhost:1313 查看效果
```

#### 5.**确认 GitHub Actions workflow（自动部署）**

- starter 仓库中自带 `.github/workflows/deploy.yml`（或类似文件），该 workflow 会在你 push 到主分支后构建 Hugo 并把 `public/` 内容发布到 GitHub Pages（通常是 `gh-pages` 分支或通过 `pages` 路径）。你可以打开该文件预览构建步骤并根据需要修改 Hugo 版本或 build 参数。示例和历史记录见作者 repo。[GitHub](https://github.com/CaiJimmy/hugo-theme-stack-starter/actions/workflows/deploy.yml?utm_source=chatgpt.com)

![image-20251107220605272](image-20251107220605272.png)

![image-20251109181141764](image-20251109181141764.png)

#### 6.**推送修改并等待自动部署**

```
git add .
git commit -m "Customize site config"
git push -u origin master
```

- 推送后，GitHub Actions 会触发构建并将结果发布到 GitHub Pages。你可以在仓库的 **Actions** 标签页查看构建日志与历史。成功后，访问你设置的 `baseURL` 即可看到站点。

比如：https://selaphina.github.io/

**7.新建博客帖子：**

根目录\content\post

![image-20251109181312523](image-20251109181312523.png)

## 附录

### create a new repository on the command line

```
git remote add origin git@github.com:Selaphina/Selaphina.github.io.git

git add .
git commit -m "first commit"
git push -u origin master
```

### push an existing repository from the command line

```
git remote add origin git@github.com:huoshaoweiba/Sera.github.io-.git
git branch -M main
git push -u origin main
```

### 怎么删除帖子？

> 本想设置草稿帖子，即在网络上访问时不显示的帖子。但是无论设置draft: true还是private：true都失败了，草稿照常显示，非常不美观。对构建config.yml等进行修改也无效果。

删除不需要的帖子。单纯从content中删除之后，本地hugo server正常删除，而公网上的帖子仍然存在。经检查，应在`\..\【站点根目录】\public\p`目录下的对应帖子也删除才可以。

不知是否是构建工作流没有顾及到位。总之目前只能用此方法删除。

### 怎么设置草稿？

上述所示，没有常规标记draft : true的草稿方式。

但是！由于工作流中规定的【不允许上传未来日期/过时的帖子】。因此，如果想要上传一个公网上不可见（相当于私密的）帖子，可以把日期改为未来的日期。比如`2100年6月6日`，这样的话起码75年内都不会有人能在公网上访问你的私密帖子了。