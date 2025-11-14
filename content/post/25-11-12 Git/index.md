---
title: Git新手使用规范
description: 新手向分布式仓库使用规范指南。
date: 2025-11-12 10:12:30+0000
image: cover.jpg
categories:
    - 工具指南
tags:
    - Git
weight: 2043       # You can add weight to some posts to override the default sorting (date descending)

---

因为Git是分布式的，你的本地仓库和远程仓库（如GitHub, GitLab）可能不同步。直接`git push`可能会因为远程有比你本地更新的提交而被拒绝，或者更糟的是，造成复杂的冲突。

## 简化流程（项目允许直接推送到主分支）

对于非常小的团队或个人项目，有时会直接在主分支上工作。即便如此，**“先拉取，再推送”**的原则依然不变。

```
# 1. 拉取远程最新代码，自动合并
git pull origin master

# 2. （解决可能的冲突，如果有）

# 3. 进行你的开发工作，添加、提交
git add .
git commit -m "你的提交信息"

# 4. 再次拉取，确保在推送前是最新的（期间可能有人提交了代码）
git pull origin master

# 5. 解决可能出现的冲突（第二次拉取时）

# 6. 推送
git push origin master
```

------

## 规范的日常协作流程（推荐工作流）

这个流程结合了分支策略，是最安全、最高效的方法。

### 第零步：准备工作（一次性）

1. **克隆仓库**：如果是第一次参与项目，先将远程仓库克隆到本地。

   ```
   git clone <远程仓库地址>
   cd <项目文件夹名>
   ```

2. **设置上游分支（可选）**：克隆后，你的本地`main`（或`master`）分支通常已经自动跟踪了远程的`origin/main`分支。

### 第一步：开始新功能或修复前——基于最新代码创建分支

这是关键的最佳实践！不要在本地的主分支上直接修改。

1. **切换到主分支**：确保你从一个干净的起点开始。

   ```
   git checkout master
   ```

2. **拉取远程最新变更**：这保证了你的主分支是最新的。

   ```
   git pull origin master
   # 或者简单写 git pull （如果当前分支已跟踪远程分支）
   ```

3. **创建并切换到一个新分支**：分支名要有描述性，例如`feat/user-login`或`fix/header-alignment`。

   ```
   git checkout -b feat/remote-one
   ```

### 第二步：日常开发工作

在你的特性分支上进行开发。

1. **进行修改**：添加、删除、修改代码文件。

2. **暂存更改**：将需要提交的文件加入暂存区。

   ```
   git add <文件名>
   # 或添加所有更改
   git add .
   ```

3. **提交更改**：在本地提交，写清晰的提交信息。

   ```
   git commit -m "描述你完成了什么工作"
   ```

4. **重复**：不断重复`add`和`commit`，直到功能完成。

### 第三步：准备推送——整合最新代码

在将你的分支推送到远程之前，很可能主分支已经有了新的提交。你需要将这些新变化整合到你的特性分支中，以确保你的代码是基于最新代码的。

1. **切换到主分支并拉取最新代码**：

   ```
   git checkout master
   git pull
   ```

2. **切回你特性分支**：

   ```
   git checkout feat/remote-one
   ```

3. **将主分支的更新合并到你的分支**（有两种主流方式）：

   - **方式A：合并Merge）**（更简单，推荐新手）

     ```
     git merge master
     ```

     这会在你的特性分支上创建一个“合并提交”，记录整合点。

   - **方式B：变基（Rebase）**（历史更整洁，但需谨慎）

     ```
     git rebase master
     ```

     这会将你的所有提交“重新播放”在最新的主分支之上，使历史呈线性。**注意：不要在公共分支上使用rebase**。

4. **解决冲突（如果有）**：如果合并或变基过程中发生代码冲突，Git会提示你。你需要手动打开冲突文件，解决冲突（选择保留谁的代码，或进行修改）。解决后，使用`git add`标记冲突已解决。如果使用`merge`，然后执行`git commit`（合并提交）。如果使用`rebase`，执行`git rebase --continue`。

### 第四步：推送到远程并发起合并请求（Pull Request）

1. **推送特性分支到远程**：

   ```
   git push -u origin feat/your-feature-name
   # -u 参数设置上游分支，之后可以直接用 git push
   ```

2. **在GitHub/GitLab等平台上创建Pull Request（PR）**：前往你的仓库页面，通常会有提示让你创建PR。选择你的特性分支作为源分支，选择`main`作为目标分支。在PR描述中清晰说明你的修改内容。

3. **代码审查**：团队成员在PR中进行讨论和代码审查。你可能需要根据反馈继续在本地分支上提交修改，然后再次`push`，PR会自动更新。

4. **合并PR**：审查通过后，由有权限的人（可能就是你）将PR合并到主分支。通常平台会提供“Squash and Merge”（压缩合并）等选项，保持历史整洁。

### 第五步：清理

合并成功后，你可以删除本地和远程的特性分支，保持整洁。

1. **删除远程分支**（在PR页面或通过命令）：

   ```
   git push origin --delete feat/your-feature-name
   ```

2. **切换回主分支**：

   ```
   git checkout main
   ```

3. **拉取最新的合并结果**：

   ```
   git pull
   ```

4. **删除本地分支**：

   ```
   git branch -d feat/your-feature-name
   ```

------

## git 在新设备上配置新的SSH密钥

### 1. **检查现有的 SSH 密钥**

打开 Git Bash 并运行以下命令，查看是否已存在 SSH 密钥：

```
ls -al ~/.ssh
```

如果看到 `id_ed25519`和 `id_ed25519.pub`（或 `id_rsa`和 `id_rsa.pub`）等文件，说明你已有密钥。

------

### 2. **生成新的 SSH 密钥（如果没有）**

运行以下命令（替换为你的 GitHub 注册邮箱）：

```
ssh-keygen -t ed25519 -C "1806527871@qq.com"
```

按 Enter 接受默认保存路径，并设置一个安全的密码（可选）。

------

### 3. **将 SSH 密钥添加到 ssh-agent**

启动 ssh-agent 并添加密钥：

```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

------

### 4. **将公钥复制到 GitHub**

- 显示公钥内容：

  ```
  cat ~/.ssh/id_ed25519.pub
  ```

- 复制输出的全部内容（以 `ssh-ed25519`开头）。

- 登录 GitHub → Settings → SSH and GPG keys → New SSH key → 粘贴并保存。

![image-20251114103822658](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20251114103822658.png)