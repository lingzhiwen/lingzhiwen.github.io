---
title: Git
categories: 命令行
---
Git 是一个强大的版本控制工具，用于跟踪和管理代码仓库的版本历史。以下是一些常见的 Git 命令以及它们的用法：

1. **仓库**：

   ```bash
   //初始化一个新的 Git 仓库
   git init 

   //克隆一个远程仓库
   git clone <远程仓库URL>

   //推送到远程仓库
   git push origin <分支名>

   //拉取远程仓库的更改
   git pull

   //查看远程仓库
   git remote -v
   ```

2. **文件状态**：

   ```bash
   //查看文件状态
   git status

   //添加文件到暂存区
   git add <文件名>

   //提交更改
   git commit -m "提交说明"

   //查看提交历史
   git log

   //撤销更改
   git reset <文件名>
   ```

3. **分支**：

   ```bash
   //创建新的分支
   git branch <分支名>

   //切换分支
   git checkout <分支名>

   //合并分支
   git merge <分支名>

   //查看所有分支
   git branch -a
   ```


4.  **创建标签**：

    ```bash
    git tag <标签名>
    ```

    这个命令用于创建一个新的标签，通常用于标记版本发布。

5. **子模块**
   Git 子模块是一种 Git 仓库中包含其他 Git 仓库的机制，允许您将一个 Git 仓库嵌套在另一个 Git 仓库中。以下是一些常用的 Git 子模块命令：
   ```
   //添加子模块
   git submodule add <repository_url> <path>

   //初始化子模块
   git submodule init

   //更新子模块，这将拉取子模块的最新内容。您还可以使用 `--init` 参数来初始化并更新子模块。
   git submodule update

   //递归更新子模块，这将递归更新所有子模块，包括它们的子模块。
   git submodule update --recursive

   //检出特定提交或分支，这将将子模块切换到其远程仓库的最新提交。
   git submodule update --remote

   //查看子模块状态，这将显示子模块的当前提交、路径和子模块仓库的 URL。
   git submodule status

   //删除子模块

   //删除子模块的记录： 
   git submodule deinit <path>
   
   //删除子模块目录：
   git rm <path>

   //删除子模块配置： 
   rm -rf .git/modules/<path>
   
   //克隆包含子模块的仓库，如果您克隆了一个包含子模块的主仓库，您可以使用 `--recursive` 标志来递归初始化和更新子模块：
   git clone --recursive <repository_url>

   //排除git其他仓库，只执行当前仓库命令
   git submodule foreach 'case $name in "submodule1") ;; *) git checkout dev ;; esac'

   //android-build.gradle写法：
   exec {              
      commandLine 'git', 'submodule', 'foreach', 'case $name in submodule1) ;; *) git checkout dev ;; esac' 
      }
   ```

这些是一些常用的 Git 子模块命令，帮助您管理主仓库中的子模块。子模块是在需要在一个项目中包含另一个项目的情况下非常有用的工具。




**克隆大项目**

```bash
git config --global http.postBuffer 524288000

$ git clone --depth 1 https://github.com/demo/xxxxxxx.git
$ git remote set-branches origin 'remote_branch_name'
$ git fetch --depth 1 origin remote_branch_name
$ git checkout remote_branch_nameÅ

```

**Gitee在提交大文件时，出现如下错误，异常退出，提示需要付费企业版支持Git LFS功能**

```bash
rm .git/hooks/pre-push
git push -u origin "master"
``` 

**Git 仓库中的 .gitignore 文件不生效**
```
git rm -r --cached .
git add .
git commit -m "Fix .gitignore"
```

**查看当前 Git 仓库中的分支名称**
```
git rev-parse --abbrev-ref HEAD

//android-build.gradle写法：
def getGitBranch(String repoPath) {
            Process process = ["git", "-C", file(repoPath), "rev-parse", "--abbrev-ref", "HEAD"].execute();
            return process.text.trim();
        }
```

