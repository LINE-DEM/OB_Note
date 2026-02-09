# Git 命令拆解

## 核心摘要

Git 的所有命令都围绕**三个区域**流转：工作区 → 暂存区 → 仓库，理解这个流转规律，所有命令都能推出来。

---

## 详细分析

### 底层原理 / 核心规律

Git 的核心公式：

```
工作区（Working Directory）
    ↓  git add
暂存区（Staging Area / Index）
    ↓  git commit
本地仓库（Local Repository）
    ↓  git push
远程仓库（Remote Repository）
```

**一句话规律**：所有 Git 命令都是在这四个区域之间**移动文件状态**，或者**查看/比较**这些状态。

---

### 具体用法（按难度从低到高排列）

---

#### 1. 基础配置命令

**看代码**

```bash
git config --global user.name "Your Name"
git config --global user.email "your@email.com"
git config --list
```

**拆解**

```bash
git config --global user.name "Your Name"
//  ↑↑↑↑↑↑  ↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑  ↑↑↑↑↑↑↑↑↑↑↑
//  命令     作用域    配置项         值
```

| 作用域 | 含义 |
|--------|------|
| `--global` | 当前用户的所有仓库 |
| `--local` | 仅当前仓库（默认） |
| `--system` | 整个系统所有用户 |

---

#### 2. 仓库初始化

**看代码**

```bash
git init                    # 在当前目录创建新仓库
git clone <url>             # 克隆远程仓库
git clone <url> <dirname>   # 克隆到指定目录名
```

**拆解**

```bash
git clone https://github.com/user/repo.git myproject
//  ↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑
//  命令              远程仓库地址            本地目录名
```

---

#### 3. 状态查看命令

**看代码**

```bash
git status              # 查看工作区和暂存区状态
git status -s           # 简洁模式
git log                 # 查看提交历史
git log --oneline       # 单行显示
git log --graph         # 图形化显示分支
git diff                # 工作区 vs 暂存区
git diff --staged       # 暂存区 vs 最新提交
```

**拆解**

```bash
git log --oneline --graph --all -n 10
//  ↑↑↑ ↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑ ↑↑↑↑↑ ↑↑↑↑
//  命令  单行显示  图形化  所有分支 最近10条
```

**状态标记速查**

| 标记 | 含义 |
|------|------|
| `??` | 未跟踪（新文件） |
| `A` | 已添加到暂存区 |
| `M` | 已修改 |
| `D` | 已删除 |
| `R` | 重命名 |

---

#### 4. 添加与提交（核心流程）

**看代码**

```bash
git add <file>          # 添加单个文件到暂存区
git add .               # 添加所有变更
git add -A              # 添加所有（包括删除）
git add -p              # 交互式添加（按块选择）

git commit -m "message" # 提交暂存区到仓库
git commit -am "msg"    # add + commit 合并（仅已跟踪文件）
git commit --amend      # 修改最后一次提交
```

**拆解**

```bash
git commit -m "feat: add login page"
//  ↑↑↑↑↑↑ ↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//  命令   消息标志     提交信息
```

**提交信息规范（Conventional Commits）**

```
<type>(<scope>): <description>

type:
  feat     新功能
  fix      修复bug
  docs     文档变更
  style    代码格式（不影响逻辑）
  refactor 重构
  test     测试
  chore    构建/工具变更
```

---

#### 5. 分支操作

**看代码**

```bash
git branch                  # 列出本地分支
git branch -a               # 列出所有分支（含远程）
git branch <name>           # 创建新分支
git branch -d <name>        # 删除分支（已合并）
git branch -D <name>        # 强制删除分支

git checkout <branch>       # 切换分支（旧语法）
git checkout -b <branch>    # 创建并切换

git switch <branch>         # 切换分支（新语法，推荐）
git switch -c <branch>      # 创建并切换
```

**拆解**

```bash
git checkout -b feature/login origin/main
//  ↑↑↑↑↑↑↑↑ ↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑
//  命令    创建  新分支名       基于哪个分支
```

**对比**

```bash
# 旧语法（checkout 身兼多职，容易混淆）
git checkout <branch>       # 切换分支
git checkout -- <file>      # 撤销文件修改

# 新语法（职责分离，更清晰）
git switch <branch>         # 切换分支
git restore <file>          # 撤销文件修改
```

---

#### 6. 合并与变基

**看代码**

```bash
git merge <branch>          # 合并分支到当前分支
git merge --no-ff <branch>  # 强制创建合并提交
git merge --squash <branch> # 压缩为单次提交

git rebase <branch>         # 变基到目标分支
git rebase -i HEAD~3        # 交互式变基最近3次提交
```

**拆解**

```bash
git rebase -i HEAD~3
//  ↑↑↑↑↑↑ ↑↑ ↑↑↑↑↑↑↑
//  命令  交互  从HEAD往回3个提交
```

**merge vs rebase 对比**

```
merge：保留完整历史，产生合并节点
  A---B---C (main)
       \   \
        D---E---M (feature, M是合并节点)

rebase：线性历史，更整洁
  A---B---C (main)
           \
            D'---E' (feature, 提交被"移动"了)
```

---

#### 7. 远程操作

**看代码**

```bash
git remote -v                       # 查看远程仓库
git remote add <name> <url>         # 添加远程仓库
git remote remove <name>            # 移除远程仓库

git fetch                           # 拉取远程更新（不合并）
git fetch --all                     # 拉取所有远程

git pull                            # fetch + merge
git pull --rebase                   # fetch + rebase

git push                            # 推送到远程
git push -u origin <branch>         # 首次推送并设置上游
git push --force                    # 强制推送（危险！）
git push --force-with-lease         # 安全的强制推送
```

**拆解**

```bash
git push -u origin feature/login
//  ↑↑↑↑ ↑↑ ↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑
//  命令 设上游 远程名   分支名
```

**对比**

```bash
# 不设上游（每次都要写全）
git push origin feature/login
git pull origin feature/login

# 设置上游后（-u 只需第一次）
git push
git pull
```

---

#### 8. 撤销与回退

**看代码**

```bash
# 撤销工作区修改
git restore <file>              # 丢弃工作区改动
git restore .                   # 丢弃所有改动

# 撤销暂存
git restore --staged <file>     # 从暂存区移出
git reset HEAD <file>           # 同上（旧语法）

# 回退提交
git reset --soft HEAD~1         # 回退提交，保留改动在暂存区
git reset --mixed HEAD~1        # 回退提交，保留改动在工作区（默认）
git reset --hard HEAD~1         # 回退提交，丢弃所有改动（危险！）

git revert <commit>             # 创建新提交来撤销指定提交（安全）
```

**拆解**

```bash
git reset --soft HEAD~1
//  ↑↑↑↑↑ ↑↑↑↑↑↑ ↑↑↑↑↑↑↑
//  命令   模式   回退到哪（HEAD的前1个）
```

**reset 三种模式对比**

| 模式 | 仓库 | 暂存区 | 工作区 |
|------|------|--------|--------|
| `--soft` | ✓ 回退 | 保留 | 保留 |
| `--mixed` | ✓ 回退 | ✓ 清空 | 保留 |
| `--hard` | ✓ 回退 | ✓ 清空 | ✓ 清空 |

---

#### 9. 暂存工作（stash）

**看代码**

```bash
git stash                   # 暂存当前工作
git stash save "message"    # 带描述的暂存
git stash list              # 查看暂存列表
git stash pop               # 恢复并删除最近的暂存
git stash apply             # 恢复但不删除
git stash drop              # 删除最近的暂存
git stash clear             # 清空所有暂存
```

**拆解**

```bash
git stash apply stash@{2}
//  ↑↑↑↑↑ ↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑
//  命令  恢复   指定第几个暂存（0是最新）
```

---

#### 10. 标签管理

**看代码**

```bash
git tag                         # 列出所有标签
git tag <name>                  # 创建轻量标签
git tag -a <name> -m "msg"      # 创建附注标签
git tag -a <name> <commit>      # 给指定提交打标签
git push origin <tag>           # 推送标签
git push origin --tags          # 推送所有标签
git tag -d <name>               # 删除本地标签
git push origin :refs/tags/<n>  # 删除远程标签
```

**拆解**

```bash
git tag -a v1.0.0 -m "Release version 1.0.0"
//  ↑↑↑ ↑↑ ↑↑↑↑↑↑ ↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑
//  命令 附注 标签名 消息      标签描述
```

---

#### 11. 查看与比较

**看代码**

```bash
git show <commit>           # 查看某次提交详情
git show <tag>              # 查看标签详情
git blame <file>            # 查看每行是谁改的
git diff <branch1>..<branch2>   # 比较两个分支
git diff <commit1>..<commit2>   # 比较两次提交
```

**拆解**

```bash
git diff main..feature/login -- src/
//  ↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑ ↑↑↑↑
//  命令   分支1..分支2       分隔符 限定目录
```

---

#### 12. 高级操作

**看代码**

```bash
git cherry-pick <commit>    # 挑选某个提交应用到当前分支
git bisect start            # 二分查找问题提交
git reflog                  # 查看所有操作记录（救命用）
git clean -fd               # 删除未跟踪的文件和目录
```

**拆解**

```bash
git cherry-pick abc123 def456
//  ↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑ ↑↑↑↑↑↑
//  命令         提交1   提交2（可以挑多个）
```

---

### 快速读法

遇到不认识的 Git 命令时：

1. **看命令主体**：`git <主命令>` 确定在操作什么（branch/commit/remote...）
2. **看标志位**：`-` 开头的是选项，`--` 开头的是完整选项名
3. **想想三个区**：这个命令是在哪两个区域之间操作？
4. **用 `--help`**：`git <command> --help` 查看完整帮助

---

### 常用组合速查

```bash
# 日常工作流
git pull --rebase && git add . && git commit -m "msg" && git push

# 查看漂亮的日志
git log --oneline --graph --all --decorate

# 回退到某个提交但保留改动
git reset --soft <commit>

# 撤销已推送的提交（安全方式）
git revert <commit> && git push

# 临时切换分支做事
git stash && git switch other-branch && ... && git switch - && git stash pop

# 合并时压缩所有提交
git merge --squash feature && git commit -m "feat: add feature"
```

---

## 关联知识

- [[编译与解释代码]] - 理解代码如何从源文件变成可执行程序
- [[c和c++]] - C/C++ 项目的 Git 工作流
