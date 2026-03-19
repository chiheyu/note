
## 1. 先理解两个概念

- `Git`：本地版本控制工具，用来管理代码历史
- `GitHub`：远程代码托管平台，用来保存仓库、协作开发、发起 Pull Request

可以简单理解为：

- `Git` 负责“本地管代码”
- `GitHub` 负责“远程存代码 + 团队协作”

---

## 2. 前期准备

### 2.1 安装 Git

先[安装 Git](https://github.com/git-for-windows/git/releases/download/v2.53.0.windows.2/Git-2.53.0.2-64-bit.exe)，安装完成后在终端执行：

```bash
git --version
```

如果能看到版本号，说明 Git 安装成功。

---

### 2.2 配置 Git 身份信息

首次使用 Git，建议先配置用户名和邮箱：

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

查看是否配置成功：

```bash
git config --global --list
```

Windows 下还可以顺手加上这些常用配置（可选）：

```bash
git config --global init.defaultBranch main
git config --global core.autocrlf true
```

说明：

- `init.defaultBranch main`：以后新仓库默认主分支叫 `main`
- `core.autocrlf true`：Windows 下自动处理换行符，减少文本文件换行问题

---

## 3. 注册并绑定 GitHub 账号

### 3.1 注册 GitHub 账号

先去 GitHub 官网注册账号，记住自己的：

- 用户名
- 邮箱
- 密码

注册完成后建议完成邮箱验证。

---

### 3.2 推荐使用 SSH 方式绑定 GitHub （可选，嫌麻烦的话直接看4.1）

相比 HTTPS，`SSH` 更适合长期开发，省去重复输入账号密码。

#### 第一步：生成 SSH 密钥

在终端执行：

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
```

一路回车即可。默认会生成两个文件：

- 私钥：`~/.ssh/id_ed25519`
- 公钥：`~/.ssh/id_ed25519.pub`

如果你的系统不支持 `ed25519`，也可以使用：

```bash
ssh-keygen -t rsa -b 4096 -C "你的GitHub邮箱"
```

---

#### 第二步：启动 ssh-agent 并添加私钥

PowerShell 中执行：

```powershell
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
```

---

#### 第三步：复制公钥

PowerShell 中执行：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub
```

复制输出的整段内容。

---

#### 第四步：把公钥添加到 GitHub

进入 GitHub：

`Settings -> SSH and GPG keys -> New SSH key`

然后：

- `Title` 随便写，例如：`My Windows PC`
- `Key type` 选 `Authentication Key`
- `Key` 粘贴刚刚复制的公钥

保存即可。

---

#### 第五步：测试 SSH 是否绑定成功

```bash
ssh -T git@github.com
```

如果看到类似欢迎信息，说明绑定成功。

---

### 3.3 查看和修改远程地址

查看当前仓库远程地址：

```bash
git remote -v
```

如果你要把仓库地址改成 SSH：

```bash
git remote set-url origin git@github.com:用户名/仓库名.git
```

---

## 4. 拉取远程仓库到本地

### 4.1 直接克隆远程仓库

如果你本地还没有项目，直接执行：

```bash
git clone git@github.com:用户名/仓库名.git
```

进入项目目录：

```bash
cd 仓库名
```

---

### 4.2 已有本地目录时绑定远程仓库

如果你已经有一个本地项目目录：

```bash
cd 你的项目目录
git init
git remote add origin git@github.com:用户名/仓库名.git
```

查看是否绑定成功：

```bash
git remote -v
```

首次拉取远程主分支：

```bash
git pull origin main
```

如果远程默认分支叫 `master`，就改成：

```bash
git pull origin master
```

本仓库的默认分支就是main，不用改。

---

## 5. 拉取远程最新代码

日常开发前，建议先同步远程：

```bash
git fetch origin
git pull origin main
```

更推荐保持提交历史整洁的方式：

```bash
git pull --rebase origin main
```

说明：

- `git fetch origin`：先把远程更新取下来，但不立刻合并
- `git pull origin main`：拉取并合并远程 `main` 分支
- `git pull --rebase origin main`：拉取后用变基方式整理提交历史

---

## 6. 创建并切换到自己的开发分支

不要直接在 `main` 或 `master` 上开发，正确做法是新建分支。

创建并切换分支：

```bash
git switch -c feat/login-page
```

如果你的 Git 版本较老，也可以用：

```bash
git checkout -b feat/login-page
```

查看当前分支：

```bash
git branch
```

查看本地和远程所有分支：

```bash
git branch -a
```

常见分支命名示例，推荐将分支名设为自己的名字，例如**heyu**：

- `feat/user-center`
- `fix/order-bug`
- `refactor/auth-module`
- `docs/github-note`

---

## 7. 开发过程中最常用的本地命令

查看文件改动状态：

```bash
git status
```

查看未暂存的差异：

```bash
git diff
```

查看已经暂存的差异：

```bash
git diff --cached
```

把某个文件加入暂存区：

```bash
git add 文件名
```

把当前所有改动加入暂存区：

```bash
git add .
```

取消某个文件的暂存：

```bash
git restore --staged 文件名
```

撤销工作区未提交的改动：

```bash
git restore 文件名
```

注意：`git restore 文件名` 会丢弃该文件的本地未提交修改，使用前要确认。

---

## 8. 提交本地改动

先查看状态：

```bash
git status
```

添加改动：

```bash
git add .
```

提交：

```bash
git commit -m "feat: 完成登录页面开发"
```

提交信息建议写清楚本次修改内容，例如：

- `feat: 新增用户管理页面`
- `fix: 修复文件上传失败问题`
- `refactor: 重构权限校验逻辑`
- `docs: 补充 GitHub 使用笔记`
**注意**：提交消息应简洁明了。
查看提交历史：

```bash
git log --oneline
```

---

## 9. 把本地分支推送到 GitHub

第一次推送当前分支：

```bash
git push -u origin feat/login-page
```

说明：

- `origin`：远程仓库默认名字
- `feat/login-page`：当前本地分支名（替换为自己的分支名）
- `-u`：把本地分支和远程分支建立追踪关系，后续可以直接 `git push`

后续再次推送：

```bash
git push
```

---

## 10. 提交前同步远程最新代码

如果你开发过程中远程主分支已经更新，推送前建议先同步：

```bash
git fetch origin
git rebase origin/main
```

如果你更习惯合并方式，也可以：

```bash
git pull origin main
```

如果推送被拒绝，通常是因为远程分支比本地更新，此时先同步再推送：

```bash
git pull --rebase origin 当前分支名
git push
```

---

## 11. 一个完整的常见开发流程

假设你要从 GitHub 拉项目下来，新建分支开发，再提交到远程：

```bash
git clone git@github.com:用户名/仓库名.git
cd 仓库名
git pull --rebase origin main
git switch -c feat/my-feature
git status
git add .
git commit -m "feat: 完成某个功能"
git push -u origin feat/my-feature
```

后续继续开发：

```bash
git status
git add .
git commit -m "fix: 修复某个问题"
git push
```

---

## 12. 提交到 GitHub 之后一般还会做什么

代码推送到 GitHub 后，通常还会：

1. 在 GitHub 页面发起 `Pull Request`
2. 让同事进行代码评审
3. 评审通过后再合并到 `main`

所以一个标准协作流程通常是：

`拉代码 -> 建分支 -> 开发 -> 提交 -> 推送 -> Pull Request -> 合并`

---

## 13. 常用命令速查

### 13.1 Git 基础配置

```bash
git --version
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
git config --global --list
```

### 13.2 SSH 相关

```bash
ssh-keygen -t ed25519 -C "你的GitHub邮箱"
ssh-add $env:USERPROFILE\.ssh\id_ed25519
ssh -T git@github.com
```

### 13.3 仓库操作

```bash
git clone git@github.com:用户名/仓库名.git
git init
git remote add origin git@github.com:用户名/仓库名.git
git remote -v
git remote set-url origin git@github.com:用户名/仓库名.git
```

### 13.4 分支操作

```bash
git branch
git branch -a
git switch -c 分支名
git checkout -b 分支名
git switch main
```

### 13.5 拉取与同步

```bash
git fetch origin
git pull origin main
git pull --rebase origin main
git rebase origin/main
```

### 13.6 提交与推送

```bash
git status
git diff
git add .
git commit -m "提交说明"
git log --oneline
git push
git push -u origin 分支名
```

---

## 14. 新手最容易混淆的几点

### 14.1 `git fetch` 和 `git pull` 的区别

- `git fetch`：只获取远程更新，不自动合并
- `git pull`：获取远程更新并自动合并到当前分支

---

### 14.2 `git clone` 和 `git pull` 的区别

- `git clone`：第一次把整个远程仓库下载到本地
- `git pull`：本地已经有仓库后，再拉最新更新

---

### 14.3 为什么不要直接在 `main` 上开发

因为：

- 容易污染主分支
- 不方便多人协作
- 不利于代码评审和回滚

正确做法是：

- 从 `main` 拉最新代码
- 新建自己的功能分支
- 开发完成后推送分支并发起 Pull Request

---

## 15. 多人开发项目时应注意的事项

多人协作时，真正容易出问题的往往不是命令不会写，而是流程不规范。

### 15.1 不要直接在 `main` 或 `master` 上改代码

多人项目里，主分支通常要求保持：

- 随时可运行
- 随时可发布
- 历史清晰稳定

所以开发时应该：

- 先拉最新主分支代码
- 再创建自己的功能分支
- 在功能分支开发完成后发起 `Pull Request`

常见流程：

```bash
git switch main
git pull --rebase origin main
git switch -c feat/xxx
```

---

### 15.2 开发前先同步远程最新代码

如果多人同时开发，远程代码变化会非常频繁。开始写代码前，最好先同步一次：

```bash
git fetch origin
git pull --rebase origin main
```

这样可以减少：

- 代码冲突
- 重复开发
- 基于旧代码开发导致的问题

---

### 15.3 一个分支尽量只做一件事

不要把多个不相关需求混在一个分支里，否则会导致：

- 提交记录混乱
- 代码评审困难
- 合并风险升高

更推荐：

- 一个功能一个分支
- 一个 bug 一个分支
- 一个重构任务一个分支
我们不需要这么麻烦，在提交时使用自己的分支并配好提交信息即可。
例如：

- `feat/user-export`
- `fix/login-timeout`
- `refactor/menu-permission`

---

### 15.4 提交要小而清晰

多人协作时，提交记录不仅是给自己看的，也是给同事看的。

建议：

- 一次提交只包含一类改动
- 提交说明写清楚“做了什么”
- 不要把格式化、重构、功能修改全部混在同一次提交里

例如：

```bash
git commit -m "feat: 新增用户导出功能"
git commit -m "fix: 修复角色权限校验错误"
git commit -m "docs: 补充部署说明"
```

---

### 15.5 推送前再次同步主分支

你本地开发期间，别人可能已经把新代码合并进主分支了，所以推送前最好同步一下：

```bash
git fetch origin
git rebase origin/main
```

如果执行 `rebase` 时发生冲突：

1. 打开冲突文件并手动处理
2. 处理完后执行：

```bash
git add .
git rebase --continue
```

如果你想放弃这次变基：

```bash
git rebase --abort
```

---

### 15.6 遇到冲突不要慌，先看谁改了什么

代码冲突不代表出错，只代表两个人改到了相近位置。

处理冲突时建议：

- 先看冲突文件属于哪个模块
- 确认自己改动和同事改动是否都需要保留
- 不确定时先沟通，不要凭感觉删代码

常用辅助命令：

```bash
git status
git diff
```

如果冲突较复杂，最好和相关同事一起确认再解决。

---

### 15.7 不要把本地配置、密钥、临时文件提交上去

多人项目里尤其要注意以下内容不要随便提交：

- 数据库账号密码
- 接口密钥、Token、JWT 密钥
- 本地环境专用配置
- 日志文件
- 构建产物
- IDE 配置和临时缓存文件

提交前可以先看：

```bash
git status
```

必要时通过 `.gitignore` 忽略不该提交的文件。

---

### 15.8 合并前尽量走 Pull Request 和代码评审

不要习惯性地“写完就直接合并”，更稳妥的做法是：

1. 推送自己的功能分支
2. 发起 `Pull Request`
3. 让同事 review
4. 修改 review 意见
5. 评审通过后再合并

这样做的好处是：

- 能提前发现 bug
- 能统一代码风格
- 能减少误删逻辑
- 能让团队成员了解彼此改动

---

### 15.9 涉及数据库和接口变更时要提前沟通

多人开发里最容易互相影响的，通常不是页面代码，而是：

- 数据库表结构
- 接口字段
- 枚举和状态值
- 公共工具类
- 权限与菜单配置

如果你要改这些内容，建议提前同步给团队，避免出现：

- 你改了字段名，别人页面全挂
- 你删了接口参数，别人服务报错
- 你改了公共方法，多个模块一起受影响

---

### 15.10 养成先看状态再操作的习惯

多人协作中，很多问题都能通过“先看一眼状态”避免。

常用检查命令：

```bash
git status
git branch
git remote -v
git log --oneline --decorate -5
```

建议在这些时机先检查一下：

- 提交前
- 推送前
- 切换分支前
- 拉远程代码前
- 解决冲突后

---

## 16. 一句话总结

从零开始使用 GitHub，核心流程就是：

`安装 Git -> 配置身份 -> 绑定 GitHub SSH -> clone/pull 项目 -> 新建分支 -> 开发并 git add/git commit -> git push 到远程`

只要把这条流程跑顺，日常开发基本就能独立完成。
