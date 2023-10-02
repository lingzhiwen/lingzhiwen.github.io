---
title: Git
categories: 命令行
---
Git 是一个强大的版本控制工具，用于跟踪和管理代码仓库的版本历史。以下是一些常见的 Git 命令以及它们的用法：

1. **初始化一个新的 Git 仓库**：

   ```bash
   git init
   ```

   这个命令在当前目录创建一个新的 Git 仓库。

2. **克隆一个远程仓库**：

   ```bash
   git clone <远程仓库URL>
   ```

   这个命令用于从远程仓库克隆代码到本地。

3. **查看文件状态**：

   ```bash
   git status
   ```

   这个命令显示工作目录中文件的状态，包括未跟踪、已修改和已暂存的文件。

4. **添加文件到暂存区**：

   ```bash
   git add <文件名>
   ```

   你可以使用这个命令将文件添加到 Git 的暂存区，以便后续提交。

5. **提交更改**：

   ```bash
   git commit -m "提交说明"
   ```

   这个命令将你的暂存区中的更改提交到本地仓库，并附带一个提交说明。

6. **查看提交历史**：

   ```bash
   git log
   ```

   这个命令显示项目的提交历史，包括每个提交的作者、日期和提交说明。

7. **创建分支**：

   ```bash
   git branch <分支名>
   ```

   这个命令创建一个新的分支，你可以在新分支上进行开发，而不影响主分支。

8. **切换分支**：

   ```bash
   git checkout <分支名>
   ```

   这个命令用于切换到指定分支。

9. **合并分支**：

   ```bash
   git merge <分支名>
   ```

   这个命令将指定分支的更改合并到当前分支。

10. **推送到远程仓库**：

    ```bash
    git push origin <分支名>
    ```

    这个命令将本地分支的更改推送到远程仓库。

11. **拉取远程仓库的更改**：

    ```bash
    git pull
    ```

    这个命令用于从远程仓库拉取最新的更改到本地。

12. **查看远程仓库**：

    ```bash
    git remote -v
    ```

    这个命令列出了与你的本地仓库关联的远程仓库。

13. **撤销更改**：

    ```bash
    git reset <文件名>
    ```

    这个命令用于从暂存区中取消对文件的暂存。

14. **创建标签**：

    ```bash
    git tag <标签名>
    ```

    这个命令用于创建一个新的标签，通常用于标记版本发布。

**克隆大项目**

```bash
git config --global http.postBuffer 524288000

$ git clone --depth 1 https://github.com/demo/xxxxxxx.git
$ git remote set-branches origin 'remote_branch_name'
$ git fetch --depth 1 origin remote_branch_name
$ git checkout remote_branch_nameÅ

```

***Gitee在提交大文件时，出现如下错误，异常退出，提示需要付费企业版支持Git LFS功能***

```bash
rm .git/hooks/pre-push
git push -u origin "master"
``` 