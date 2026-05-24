# Git 本地与远程同时修改时的冲突处理

## 1. 这类问题是什么

典型场景：

1. 你在电脑 A 上修改并推送了仓库。
2. 你在电脑 B 上也修改了同一个仓库，但还没有拉取电脑 A 的更新。
3. 电脑 B 执行 `git pull` 或准备 `git push` 时，Git 发现本地和远程历史不一致，可能出现冲突。

这种情况本质上不是“仓库坏了”，而是 Git 不知道应该如何同时保留两边的修改，需要你明确告诉它：

- 哪些本地修改要保留；
- 哪些远程修改要保留；
- 如果同一个文件两边都改了，应该如何合并；
- 如果一边改名、一边修改原文件，应该把修改迁移到新文件，还是恢复旧文件。

---

## 2. 本次我是如何解决的

本次仓库状态大致是：

- 本地分支：`main`
- 远程分支：`origin/main`
- 本地落后远程 1 个提交；
- 本地也有一批未提交修改；
- 主要风险点是：
  - `.obsidian/plugins/manual-sorting/data.json` 本地和远程都改了；
  - 本地把数据结构章节重新编号了，例如：
    - `第四章 树与二叉树.md` 改成了 `第五章 树与二叉树.md`
    - `第五章 图.md` 改成了 `第六章 图.md`
    - `第六章 查找.md` 改成了 `第七章 查找.md`
    - `第七章 排序.md` 改成了 `第八章 排序.md`
  - 远程提交还在旧路径 `第四章 树与二叉树.md` 上新增了内容和图片。

### 2.1 先查看状态

我先执行了：

```bash
git status --short --branch
git diff --name-only --diff-filter=U
git log --oneline --graph --decorate --all -10
git diff --name-status HEAD
git diff --name-status HEAD..origin/main
```

其中：

- `git status --short --branch` 用来判断当前分支是否领先、落后、是否有未提交文件；
- `git diff --name-only --diff-filter=U` 用来查看是否已经处在冲突状态；
- `git log --oneline --graph --decorate --all -10` 用来看本地和远程历史分叉情况；
- `git diff --name-status HEAD` 用来看本地未提交改了哪些文件；
- `git diff --name-status HEAD..origin/main` 用来看远程比本地多了哪些修改。

本次发现并没有已经进入冲突状态，只是“本地有未提交修改 + 远程有新提交”。

### 2.2 判断冲突真正在哪里

远程改动包括：

```text
.obsidian/plugins/manual-sorting/data.json
408/数据结构/assets/第四章 树与二叉树/file-20260520.png
408/数据结构/assets/第四章 树与二叉树/file-20260520 1.png
408/数据结构/第四章 树与二叉树.md
考研数学/线性代数/第二章 矩阵/第三节 矩阵的逆矩阵.md
考研数学/高等数学/第九章 无穷级数/第九章 无穷级数.md
```

本地改动包括：

```text
.obsidian/plugins/manual-sorting/data.json
408/数据结构/第三章 栈、队列和数组.md
408/数据结构/第四章 串.md
408/数据结构/第五章 树与二叉树.md
408/数据结构/第六章 图.md
408/数据结构/第七章 查找.md
408/数据结构/第八章 排序.md
考研数学/高等数学/补充知识/级数敛散性判别方法.md
```

关键判断：

- 本地并不是想删除“树与二叉树”内容，而是把它从第四章移动到了第五章；
- 远程是在旧文件 `第四章 树与二叉树.md` 上继续写了内容；
- 所以正确处理方式不是简单保留本地删除，也不是恢复旧文件，而是把远程新增内容合并到本地新文件 `第五章 树与二叉树.md`。

### 2.3 先提交本地修改

因为本地修改是一批完整的笔记调整，所以我没有直接 `pull`。我先把本地修改提交成一个独立提交：

```bash
git add -A
git commit -m "vault backup: local notes before merge"
```

这样做的好处：

- 本地修改有了一个明确的历史节点；
- 合并远程时，Git 更容易识别“文件重命名”；
- 即使合并失败，也更容易回退到合并前状态。

本次生成的本地提交是：

```text
6292225 vault backup: local notes before merge
```

### 2.4 合并远程提交

然后执行：

```bash
git merge origin/main
```

Git 自动完成了合并，并生成合并提交：

```text
b9e911d Merge remote-tracking branch 'origin/main'
```

这次 Git 的 `ort` 合并策略识别出了本地的重命名，所以远程对旧文件 `第四章 树与二叉树.md` 的新增内容被合并进了本地新文件：

```text
408/数据结构/第五章 树与二叉树.md
```

同时远程新增的图片也被保留下来：

```text
408/数据结构/assets/第四章 树与二叉树/file-20260520.png
408/数据结构/assets/第四章 树与二叉树/file-20260520 1.png
```

### 2.5 验证合并结果

合并后我检查了：

```bash
git status --short --branch
git log --oneline --graph --decorate --all -5
```

并确认 `第五章 树与二叉树.md` 中同时保留了远程新增内容，例如：

- `file-20260520.png`
- `file-20260520 1.png`
- `先序线索二叉树的寻找规律`
- `转换成二叉树后没有左孩子的结点数=树/森林中的叶子结点的数量`
- `加权平均长度`
- `前缀编码`

### 2.6 最后推送

确认工作区干净后执行：

```bash
git push origin main
```

最终状态：

```text
main 和 origin/main 已同步
工作区干净
```

---

## 3. 以后遇到类似问题的标准流程

### 第一步：不要急着 pull

先看状态：

```bash
git status --short --branch
```

常见结果：

```text
## main...origin/main [behind 1]
 M some-file.md
?? new-file.md
```

含义：

- `[behind 1]`：远程比你本地多 1 个提交；
- `M`：本地修改过的文件；
- `??`：本地新增但还没有被 Git 跟踪的文件。

如果你这时直接 `git pull`，Git 可能会因为本地修改和远程修改冲突而中断。

### 第二步：获取远程信息

建议先执行：

```bash
git fetch origin
```

`git fetch` 只下载远程最新历史，不会改你的工作区，比 `git pull` 更安全。

然后查看本地和远程差异：

```bash
git log --oneline --graph --decorate --all -10
git diff --name-status HEAD
git diff --name-status HEAD..origin/main
```

重点看有没有同一个文件两边都改了，或者一边改名、一边修改旧文件。

### 第三步：判断本地修改是否值得先提交

如果本地修改是一批有意义的完整改动，优先提交：

```bash
git add -A
git commit -m "说明这次本地修改"
```

如果本地修改只是临时改动，不想提交，可以先暂存：

```bash
git stash push -u -m "临时保存本地修改"
```

说明：

- `git commit` 适合已经确定要保留的修改；
- `git stash` 适合临时改动，还不想形成提交；
- `-u` 表示连未跟踪的新文件也一起暂存。

### 第四步：合并远程

如果已经提交本地修改：

```bash
git merge origin/main
```

如果使用了 stash：

```bash
git merge origin/main
git stash pop
```

如果没有冲突，Git 会自动完成。

如果有冲突，继续下一步。

---

## 4. 真的出现冲突时怎么处理

### 4.1 查看冲突文件

```bash
git status --short
git diff --name-only --diff-filter=U
```

冲突文件通常会显示为：

```text
UU path/to/file.md
```

如果是删除/修改冲突，可能看到：

```text
DU path/to/file.md
UD path/to/file.md
```

大致含义：

- `UU`：两边都修改了同一个文件；
- `DU`：一边删除，另一边修改；
- `UD`：一边修改，另一边删除。

### 4.2 打开冲突文件

冲突文件里通常会出现：

```text
<<<<<<< HEAD
本地版本
=======
远程版本
>>>>>>> origin/main
```

处理原则：

- `<<<<<<< HEAD` 到 `=======` 之间是当前本地版本；
- `=======` 到 `>>>>>>> origin/main` 之间是远程版本；
- 不要把这些冲突标记留在最终文件里；
- 最终文件应该是你人工整理后的正确内容。

### 4.3 常见冲突处理策略

#### 情况一：两边改了不同段落

保留两边内容，整理成一份完整文件。

#### 情况二：两边改了同一句话

人工判断哪边更正确，或者改写成同时吸收两边信息的新表述。

#### 情况三：本地改名，远程修改旧文件

这是本次遇到的核心情况。

处理思路：

1. 先确认本地改名是否是有意的；
2. 如果改名是正确的，不要恢复旧文件；
3. 把远程旧文件里的新增内容迁移到本地新文件；
4. 保留远程新增的图片、附件等资源；
5. 删除不应该继续存在的旧路径。

举例：

```text
远程修改：408/数据结构/第四章 树与二叉树.md
本地改名：408/数据结构/第五章 树与二叉树.md
```

正确结果通常是：

```text
保留：408/数据结构/第五章 树与二叉树.md
合并：远程在旧文件中新增的内容
删除：408/数据结构/第四章 树与二叉树.md
```

#### 情况四：本地删除，远程修改

先问自己：

- 我本地删除这个文件是误删，还是有意删除？
- 远程修改的内容还有没有价值？
- 文件是否已经被移动到别的位置？

如果是误删：

```bash
git checkout --theirs -- path/to/file.md
git add path/to/file.md
```

如果是有意删除：

```bash
git rm path/to/file.md
```

如果是移动文件：

手动把远程新增内容复制/合并到新路径，然后：

```bash
git add 新路径
git rm 旧路径
```

### 4.4 标记冲突已解决

修改完冲突文件后：

```bash
git add 冲突文件路径
```

然后继续完成合并：

```bash
git commit
```

如果 Git 已经自动准备好了合并提交信息，直接保存即可。

---

## 5. 冲突处理后必须验证

至少执行：

```bash
git status --short --branch
git log --oneline --graph --decorate --all -5
```

确认：

- 没有 `UU`、`DU`、`UD` 等未解决冲突；
- 工作区没有意外修改；
- 本地提交历史符合预期；
- 需要保留的本地内容还在；
- 需要保留的远程内容也在。

如果是 Obsidian 笔记，还要额外检查：

- 图片链接是否还有效；
- 文件路径是否符合现在的章节编号；
- 重命名后的文件是否仍然被其他笔记引用；
- `.obsidian/plugins/manual-sorting/data.json` 是否和实际目录结构一致。

---

## 6. 最后推送

确认没问题后：

```bash
git push origin main
```

如果推送失败，提示远程又更新了：

```bash
git fetch origin
git status --short --branch
git log --oneline --graph --decorate --all -10
```

然后重复上面的合并流程。

---

## 7. 常用命令速查

### 查看状态

```bash
git status --short --branch
```

### 只下载远程更新，不改工作区

```bash
git fetch origin
```

### 查看本地修改了哪些文件

```bash
git diff --name-status HEAD
```

### 查看远程比本地多了哪些修改

```bash
git diff --name-status HEAD..origin/main
```

### 查看冲突文件

```bash
git diff --name-only --diff-filter=U
```

### 提交本地修改

```bash
git add -A
git commit -m "说明这次修改"
```

### 合并远程

```bash
git merge origin/main
```

### 推送

```bash
git push origin main
```

### 临时保存本地修改

```bash
git stash push -u -m "临时保存本地修改"
```

### 恢复临时保存的修改

```bash
git stash pop
```

---

## 8. 推荐习惯

1. 每次开始写笔记前，先执行：

```bash
git pull --ff-only
```

如果本地没有修改，这样可以快速同步远程。

2. 如果已经有本地修改，不要直接 `pull`，先执行：

```bash
git status --short --branch
git fetch origin
```

3. 章节重命名、文件移动这类操作，最好单独提交一次。

例如：

```bash
git add -A
git commit -m "调整数据结构章节编号"
```

4. 多台设备写同一个 Obsidian 仓库时，尽量养成小步提交、小步推送的习惯。

推荐节奏：

```bash
git pull --ff-only
# 写笔记
git add -A
git commit -m "更新某某笔记"
git push origin main
```

这样最不容易产生复杂冲突。

