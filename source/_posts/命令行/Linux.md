---
title: Linux
categories: 命令行
---
以下是一些常见的 Linux 基本命令，用于在终端中执行各种任务：

1. **文件和目录操作**：
   - `ls`: 列出当前目录的文件和子目录。
   - `pwd`: 显示当前工作目录的路径。
   - `cd`: 切换目录，使用 `cd` 后跟目录路径。
   - `mkdir`: 创建新目录，使用 `mkdir` 后跟目录名。
   - `rmdir`: 删除空目录。
   - `touch`: 创建空文件或更新文件的访问和修改时间戳。

2. **文件查看和编辑**：
   - `cat`: 查看文件内容。
   - `more` 或 `less`: 逐页查看文件内容。
   - `nano` 或 `vim` 或 `vi`: 启动文本编辑器。
   - `head` 或 `tail`: 显示文件的前几行或后几行。
   - `grep`: 在文件中搜索文本模式。

3. **文件和目录管理**：
   - `cp`: 复制文件或目录。
   - `mv`: 移动文件或目录，也可用于重命名文件。
   - `rm`: 删除文件或目录。
   - `chmod`: 修改文件或目录的权限。
   - `chown`: 更改文件或目录的所有者。
   - `chgrp`: 更改文件或目录的组。
   
4. **文件压缩和解压**：
   - `tar`: 创建和提取 tar 归档文件。
   - `gzip` 或 `gunzip`: 压缩或解压文件。
   - `zip` 或 `unzip`: 创建和提取 zip 压缩文件。

5. **系统信息查看**：
   - `uname`: 显示系统信息。
   - `who`: 显示当前登录用户。
   - `ps`: 显示当前进程信息。
   - `top`: 动态显示系统资源使用情况。
   - `df`: 显示磁盘空间使用情况。
   - `free`: 显示内存使用情况。

6. **用户和权限管理**：
   - `passwd`: 更改用户密码。
   - `useradd` 和 `userdel`: 创建和删除用户。
   - `groupadd` 和 `groupdel`: 创建和删除用户组。
   - `sudo`: 以超级用户权限执行命令。
   - `su`: 切换用户。

7. **网络操作**：
   - `ping`: 测试与主机的网络连接。
   - `ifconfig` 或 `ip`: 显示和配置网络接口。
   - `netstat` 或 `ss`: 显示网络连接和路由表。
   - `ssh`: 远程登录到其他计算机。
   - `scp`: 安全复制文件到远程计算机。

8. **包管理（适用于不同的 Linux 发行版）**：
   - `apt` 或 `apt-get`: Debian/Ubuntu 包管理器。
   - `yum` 或 `dnf`: CentOS/RHEL 包管理器。
   - `zypper`: openSUSE 包管理器。

这些是一些常见的 Linux 基本命令，可以帮助你在终端中执行常见的任务。要了解更多关于这些命令的详细信息和选项，请查阅命令的帮助文档，通常可以使用 `man` 命令查看，例如 `man ls` 或 `man grep`。

##### 搜索apt软件包
 ```
 sudo apt-cache search <keyword>
 ```

##### 安装speex
```
下载地址：
https://ftp.osuosl.org/pub/xiph/releases/speex/speex-1.2.0.tar.gz
cd speex-1.2.0/
./configure
sudo make && sudo make install
```
##### 安装Yasm
```
下载地址：
http://www.tortall.net/projects/yasm/releases/
tar -xvf yasm-1.3.0.tar.gz
cd yasm-1.3.0/
./configure
sudo make && sudo make install
```
##### 安装X264开发库
```
下载地址：
http://www.videolan.org/developers/x264.html
tar -xvf x264-master.tar.bz2
cd x264-master/
./configure --enable-shared
sudo make && sudo make install
```
##### 安装X265开发库
```
git克隆地址：
https://github.com/videolan/x265.git
cd x265/build
cmake ../source
sudo make && sudo make install
```