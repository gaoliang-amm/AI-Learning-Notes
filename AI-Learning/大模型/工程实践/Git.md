---
aliases: [Git, 版本控制]
tags: [AI/Learning, 工具, Git]
created: 2026-06-13
---

# Git 工具参考

> **背诵版**：Git 是分布式版本控制系统，管理工作区、暂存区、本地库三区域。核心流程 `add → commit → push`，分支隔离开发，合并解决冲突。远程协作用 clone/pull/push，分支策略推荐 Git Flow。

## 一、版本控制概述

**版本控制**：记录文件内容变化，方便查阅历史版本、多人协作、分支管理。

| 类型 | 代表工具 | 特点 |
|------|---------|------|
| 集中式 | SVN, CVS | 单一中央服务器，离线不可用，有单点故障 |
| **分布式** | **Git**, Mercurial | 每个客户端保存完整仓库，断网可开发，安全可靠 |

**代码托管平台**：GitHub（全球开源）、Gitee 码云（国内快）、GitLab（可自建私有）。

---

## 二、Git 基础

### 安装与配置

- 下载：https://git-scm.com/downloads | 路径要求：**无中文、无空格** | 终端推荐 **Git Bash**

```bash
git config --global user.name "用户名"     # 全局签名（首次必做）
git config --global user.email "邮箱"      # 签名与 GitHub/Gitee 账号无关
```

### 三个区域与状态

```
工作区 (Working Directory)  --git add-->  暂存区 (Stage)  --git commit-->  本地库 (Repository)
```

| 区域 | 说明 | git status 颜色 |
|------|------|----------------|
| 工作区 | 项目目录，编辑代码的地方 | 红色（未暂存） |
| 暂存区 | `.git/index`，临时存放修改 | 绿色（已暂存未提交） |
| 本地库 | `.git/` 目录，存储版本历史 | 无颜色（已提交） |

---

## 三、常用命令

### 仓库初始化与状态

```bash
git init                          # 初始化本地仓库
git status                        # 查看工作区状态
```

### 暂存、提交与删除

```bash
git add .                         # 暂存所有文件
git add 文件名                     # 暂存指定文件
git commit -m "提交说明"           # 提交到本地库
git commit -m "说明" 文件名        # 仅提交指定文件（需先 add）
git rm 文件名                     # 删除已追踪文件（工作区 + 暂存区），需再 commit
```

### 日志与版本

```bash
git log                           # 查看详细提交历史（q 退出）
git reflog                        # 查看精简版本记录（含被 reset 的）
git reflog -n 5                   # 只看最近 5 条
```

### 版本穿梭

```bash
git reset --hard 版本号            # 回退/切换到指定版本（移动 HEAD 指针）
```

| reset 模式 | 作用 |
|-----------|------|
| `--soft` | 仅移动 HEAD，暂存区和工作区不变 |
| `--mixed`（默认） | 移动 HEAD + 清空暂存区，工作区不变 |
| `--hard` | 全部重置（慎用，会丢失未提交的修改） |

### 文件比较

```bash
git diff <file>                  # 工作区 vs 暂存区
git diff HEAD <file>             # 工作区 vs 本地库最新提交
git diff --cached <file>         # 暂存区 vs 本地库最新提交
```

### 分支操作

```bash
git branch 分支名                 # 创建分支
git branch -v                    # 查看所有分支（含最新提交）
git checkout 分支名               # 切换分支
git checkout -b 分支名            # 创建并切换分支（快捷方式）
git merge 分支名                  # 合并指定分支到当前分支
git branch -d 分支名              # 删除已合并的分支
git branch -D 分支名              # 强制删除分支
```

### 暂存工作

```bash
git stash                        # 暂存当前工作区修改（切分支前用）
git stash pop                    # 恢复最近一次暂存
git stash list                   # 查看所有暂存记录
```

### 标签管理

```bash
git tag v1.0                     # 创建标签
git tag -a v1.0 -m "说明"         # 创建带注释的标签
git tag                          # 查看所有标签
git push origin v1.0             # 推送标签到远程
```

---

## 四、远程仓库操作

### 关联远程

```bash
git remote -v                    # 查看远程地址别名
git remote add origin 远程地址     # 添加远程仓库别名
```

### 推送、拉取与克隆

```bash
git push origin 分支名            # 推送本地分支到远程
git pull origin 分支名            # 拉取远程并自动合并（= fetch + merge）
git fetch origin                 # 仅下载远程更新（不合并，更安全）
git clone 远程地址 [项目名]        # 克隆远程仓库（自动初始化 + 关联）
```

---

## 五、分支策略

### 分支本质

分支是指向提交的**可移动指针**，HEAD 指针指向当前分支。切换分支就是移动 HEAD。

### 常见分支模型

| 分支 | 用途 | 生命周期 |
|------|------|---------|
| `main`/`master` | 生产环境代码 | 永久 |
| `develop` | 开发主线 | 永久 |
| `feature/*` | 功能开发 | 临时 |
| `hotfix/*` | 紧急修复 | 临时 |
| `release/*` | 发布准备 | 临时 |

### 减少冲突的习惯

- 先 pull，再修改，改完立即 commit + push
- 各自维护自己的分支，避免修改同一文件
- 公共文件由专人维护
- 下班前提交代码，上班先拉最新

---

## 六、常见问题

### 冲突解决

合并时两个分支修改了**同一文件同一位置**，Git 无法自动决策。

```text
<<<<<<< HEAD
当前分支的代码
=======
合并过来的代码
>>>>>>> 分支名
```

**解决步骤**：
1. 手动编辑冲突文件，删除特殊标记，保留需要的代码
2. `git add 冲突文件`（标记冲突已解决）
3. `git commit`（**不带文件名**）

### 回退与误操作恢复

```bash
# 场景：commit 了但还没 push
git reset --soft HEAD~1           # 撤销最近一次 commit，保留修改在暂存区
git reset --mixed HEAD~1          # 撤销 commit，修改回到工作区

# 场景：误删文件
git checkout -- 文件名             # 从本地库恢复被删除/修改的文件（旧语法）
git restore 文件名                 # Git 2.23+ 推荐写法

# 场景：误 reset --hard（代码丢失）
git reflog                        # 找到被丢弃的提交哈希
git reset --hard 哈希              # 恢复到那次提交

# 场景：push 了错误代码
git revert <commit-hash>          # 生成反向提交（安全回滚，保留历史）
```

### .gitignore

忽略不需要版本管理的文件（IDE 配置、编译产物、密钥等）：

```text
# .gitignore 示例
.idea/
*.iml
__pycache__/
.env
*.pyc
```

配置全局忽略：
```bash
git config --global core.excludesfile ~/.gitignore_global
```

---

## 七、Git Flow 简介

Git Flow 是一种标准化的分支管理流程，适合有明确发布周期的项目。

```
main ─────●───────────────●─────── (生产环境)
            \            / \        \
             ●──●──●──●    \   hotfix
              \          \  ●──●
               develop ──●────●──── (开发主线)
                   \        /
                    ●──●──●
                   feature/xxx
```

**核心分支**：`main`（生产）+ `develop`（开发）

**辅助分支**：
- `feature/*`：从 develop 拉出，完成后合回 develop
- `release/*`：发布准备，从 develop 拉出，完成后同时合入 main 和 develop
- `hotfix/*`：紧急修复，从 main 拉出，完成后同时合入 main 和 develop

**日常开发流程**：
```bash
git checkout -b feature/新功能 develop     # 开始新功能
# ... 开发、提交 ...
git checkout develop
git merge --no-ff feature/新功能            # 合并到 develop（保留分支信息）
git branch -d feature/新功能               # 删除已合并的分支
```

> 对于个人项目或小团队，简单的 `main` + 功能分支 + PR/MR 流程已够用，不必严格遵循 Git Flow。

---

## 附：提交流程速查

```
修改代码 → git add → git commit -m "说明" → git push origin 分支名
```

| 操作 | 命令 | 说明 |
|------|------|------|
| 初始化 | `git init` | 仅首次 |
| 配置 | `git config --global user.name/email` | 仅首次 |
| 暂存 | `git add` | 每次修改后 |
| 提交 | `git commit -m "说明"` | 每次修改后 |
| 推送 | `git push` | 需要同步到远程时 |
| 拉取 | `git pull` | 开始工作前 |
| 分支 | `git branch / checkout / merge` | 功能开发时 |
| 冲突 | 手动编辑 → add → commit（不带文件名） | 合并冲突时 |
