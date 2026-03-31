# OpenClaw 容器重建与挂载手记

## 目标

这份笔记记录了我在本机 Windows + Docker Desktop 环境下，如何维护 `openclaw` 容器，包括：

- 保留原有 OpenClaw 配置
- 仅允许本机访问 `127.0.0.1:18789`
- 将本机目录挂载进容器
- 重建容器而不丢失数据

---

## 当前容器设计

当前 `openclaw` 容器的关键点：

- 容器名：`openclaw`
- 镜像：`alpine/openclaw:latest`
- 配置卷：`openclaw_home`
- 宿主机端口：`127.0.0.1:18789 -> 18789`
- 安全参数：`--security-opt no-new-privileges:true`、`--cap-drop ALL`
- 已挂载笔记目录：
  - Windows：`D:\Obsidian\note`
  - 容器内：`/mnt/obsidian-note`（只读）

说明：

- OpenClaw 的个性化配置并不放在 Windows 本地目录里，而是保存在 Docker 卷 `openclaw_home`
- 这个卷里包含模型、API key、skills、sessions 等状态
- 所以平时可以删容器，但不要乱删 `openclaw_home`

---

## 先记住一个原则

Docker **不能给已存在的容器追加挂载**。

如果你想：

- 改端口映射
- 新增本机目录挂载
- 改挂载路径

都必须：

1. 删除旧容器
2. 用新参数重新 `docker run`

只要保留 `openclaw_home` 卷，OpenClaw 数据就不会丢。

---

## 最常用命令

### 1. 查看当前容器

```powershell
docker ps --filter "name=^/openclaw$"
```

### 2. 查看当前挂载

```powershell
docker inspect openclaw --format "{{json .Mounts}}"
```

### 3. 查看健康状态

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:18789/healthz
```

### 4. 查看卷

```powershell
docker volume ls | findstr openclaw_home
```

---

## 当前可直接使用的重建命令

这条命令会：

- 删除旧的 `openclaw`
- 保留 `openclaw_home`
- 只允许本机访问 `127.0.0.1:18789`
- 把 `D:\Obsidian\note` 只读挂到容器内
- 去掉不需要的 Linux capabilities，并阻止提权

```powershell
docker rm -f openclaw

docker run -d --name openclaw `
  --restart unless-stopped `
  -p 127.0.0.1:18789:18789 `
  --security-opt no-new-privileges:true `
  --cap-drop ALL `
  --mount type=volume,src=openclaw_home,dst=/home/node/.openclaw `
  --mount type=bind,src=D:\Obsidian\note,dst=/mnt/obsidian-note,readonly `
  alpine/openclaw:latest `
  node openclaw.mjs gateway --allow-unconfigured --bind lan --port 18789
```

---

## 额外安全收紧

在这套用法里，我当前采用的是：

- 宿主机端口只发布到 `127.0.0.1`
- 笔记目录默认用 `readonly` 挂载
- 加上 `--security-opt no-new-privileges:true`
- 加上 `--cap-drop ALL`

这样做的含义是：

- OpenClaw 仍然能读到笔记库
- OpenClaw 自己的状态仍然写在 `openclaw_home`
- 容器拿不到多余的 Linux capabilities
- 即使进程被利用，也更难在容器内继续提权

如果以后你明确需要让 OpenClaw 直接改本地文件，再把 `readonly` 去掉。

---

## 为什么容器内还是 `--bind lan`

这点容易混淆。

现在的安全边界不在容器内部，而在 Docker 端口发布这一层：

- 容器内部：OpenClaw 监听 `0.0.0.0:18789`
- 宿主机发布：只发布到 `127.0.0.1:18789`

所以最终效果是：

- 本机浏览器可以访问：`http://127.0.0.1:18789/`
- 局域网其他机器不能访问

不要把容器内的 OpenClaw 改成只监听 `127.0.0.1`，那样通常会导致 Docker 端口映射失效。

---

## 如何验证挂载成功

### 验证容器已启动

```powershell
docker ps --filter "name=^/openclaw$"
```

### 验证本机端口只绑定到回环

```powershell
docker ps --filter "name=^/openclaw$" --format "{{.Ports}}"
```

正常应类似：

```text
127.0.0.1:18789->18789/tcp
```

### 验证容器能看到笔记目录

```powershell
docker exec openclaw sh -lc "ls -la /mnt/obsidian-note | sed -n '1,20p'"
```

### 验证 OpenClaw 服务可用

```powershell
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:18789/healthz
```

返回 `200` 即正常。

---

## 新增别的本机目录时怎么做

例如以后想再挂一个目录：

- Windows：`E:\data`
- 容器内：`/mnt/host-e-data`

就在 `docker run` 里多加一条：

```powershell
--mount type=bind,src=E:\data,dst=/mnt/host-e-data
```

完整示例：

```powershell
docker rm -f openclaw

docker run -d --name openclaw `
  --restart unless-stopped `
  -p 127.0.0.1:18789:18789 `
  --security-opt no-new-privileges:true `
  --cap-drop ALL `
  --mount type=volume,src=openclaw_home,dst=/home/node/.openclaw `
  --mount type=bind,src=D:\Obsidian\note,dst=/mnt/obsidian-note,readonly `
  --mount type=bind,src=E:\data,dst=/mnt/host-e-data `
  alpine/openclaw:latest `
  node openclaw.mjs gateway --allow-unconfigured --bind lan --port 18789
```

---

## 只读挂载怎么写

如果只想让容器读取，不允许写入：

```powershell
--mount type=bind,src=D:\Obsidian\note,dst=/mnt/obsidian-note,readonly
```

适合：

- 笔记库
- 文档库
- 代码仓库只读分析

---

## 常见坑

### 1. 不能给已有容器直接加挂载

错误思路：

- 先 `docker start`
- 再想补挂目录

这不行，必须重建容器。

### 2. 不要随便删 `openclaw_home`

删了容器问题不大，删了卷才会丢配置。

错误操作：

```powershell
docker volume rm openclaw_home
```

除非你确定不要当前 OpenClaw 全部配置。

### 3. 某些盘符整盘挂载可能失败

在 Windows + Docker Desktop 下：

- 挂具体目录通常更稳
- 整个盘符挂载有时会被 Docker Desktop 拒绝或跳过

所以优先挂这种路径：

```text
D:\Obsidian\note
E:\some-folder
```

而不是直接挂整个 `D:\` 或 `E:\`。

---

## 我自己的最短操作流程

以后如果只是新增一个挂载目录，我通常按这个顺序做：

1. 先确认本机目录存在
2. 记下当前卷名 `openclaw_home`
3. `docker rm -f openclaw`
4. 用新的 `docker run` 重新创建
5. 用 `docker inspect openclaw` 看挂载
6. 用 `/healthz` 验证服务

---

## 一条检查清单

- [ ] `docker volume ls` 里还能看到 `openclaw_home`
- [ ] `docker ps` 显示容器名是 `openclaw`
- [ ] 端口是 `127.0.0.1:18789->18789/tcp`
- [ ] `docker inspect openclaw` 能看到目标挂载
- [ ] `docker exec openclaw` 能列出挂载目录内容
- [ ] `http://127.0.0.1:18789/healthz` 返回 `200`

---

## 当前环境中的实际挂载路径

截至这份笔记写入时：

- Docker 卷：
  - `openclaw_home -> /home/node/.openclaw`
- Windows 目录挂载：
  - `D:\Obsidian\note -> /mnt/obsidian-note`（只读）

如果后面你又增加新的挂载，记得同步更新这篇笔记。
