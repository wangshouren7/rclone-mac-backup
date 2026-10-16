# 恢复手册

这台 Mac 用 restic + rclone 把整个数据卷加密备份到 Google Drive。本页说明怎么把数据取回来。

本页是公开页面，不含任何密码和密钥。仓库密码和 Client Secret 存在密码管理器里。

## 备份在哪里

| 项目 | 值 |
|---|---|
| Google 账号 | wangshouren116@gmail.com |
| 网盘文件夹 | 「我的云端硬盘」根目录下的 `mac-backup-restic` |
| restic 仓库地址 | `rclone:gdrive:mac-backup-restic` |
| 备份范围 | 整个数据卷 `/System/Volumes/Data`，不含 `Docker.raw` 和废纸篓 |
| 备份频率 | 每天凌晨 2:00 增量备份 |
| 快照保留 | 最近 7 天每天一份，最近 4 周每周一份，最近 12 个月每月一份 |
| Google Cloud 项目 | `starry-being-511811-i6`（显示名「My Project 7833」） |
| OAuth 客户端 ID | `896631869206-sh6t0kgspqfjcvtge15r9ltf8ol2vs7u.apps.googleusercontent.com` |

网盘里的文件是加密数据块，看不到原文件名，也打不开。取回文件必须用 restic 和仓库密码。

## 情况一：电脑还在，只取回几个文件

1. 设置环境变量：
   ```bash
   export RESTIC_REPOSITORY=rclone:gdrive:mac-backup-restic
   export RESTIC_PASSWORD_COMMAND="security find-generic-password -s restic-gdrive-repo-password -w"
   R=~/.local/opt/mac-backup/restic
   ```
2. 列出快照，选一个日期：
   ```bash
   $R snapshots
   ```
3. 按文件名搜索：
   ```bash
   $R find "*报告*.docx"
   ```
4. 把文件恢复到临时目录。`latest` 表示最新快照，也可以换成第 2 步看到的快照 ID：
   ```bash
   $R restore latest \
     --include "/System/Volumes/Data/Users/wangshouren/Documents/报告.docx" \
     --target ~/Desktop/restored
   ```
5. 到 `~/Desktop/restored/System/Volumes/Data/Users/wangshouren/...` 找到文件，拷回原位置。

备份里的路径都以 `/System/Volumes/Data` 开头，这是正常的。

## 情况二：电脑坏了，在新电脑上恢复

警告：第 3 步必须用上表里的同一个 OAuth 客户端 ID。rclone 只有 `drive.file` 权限，只能看到这个客户端自己创建的文件。用新建的客户端连接，会看不到备份文件夹。

1. 装好 macOS，安装 Homebrew，再安装工具：
   ```bash
   brew install restic rclone
   ```
2. 从密码管理器取出 Client Secret。如果 Client Secret 丢了：
   1. 打开 Google Cloud 控制台，进入项目 `starry-being-511811-i6`。
   2. 进入「Google Auth Platform」→「客户端」，打开 `rclone-mac-backup`。
   3. 点「添加密钥」，生成新的 Client Secret。Client ID 不变。
   4. 不要新建客户端。
3. 连接 Google Drive。命令会打开浏览器，用 wangshouren116@gmail.com 登录并授权：
   ```bash
   rclone config create gdrive drive \
     client_id=896631869206-sh6t0kgspqfjcvtge15r9ltf8ol2vs7u.apps.googleusercontent.com \
     client_secret=<Client Secret> \
     scope=drive.file
   chmod 600 ~/.config/rclone/rclone.conf
   ```
4. 确认能看到备份文件夹：
   ```bash
   rclone lsd gdrive:mac-backup-restic
   ```
   输出里应该有 `data`、`index`、`keys`、`locks`、`snapshots`。
5. 列出快照。restic 会要求输入仓库密码：
   ```bash
   restic -r rclone:gdrive:mac-backup-restic snapshots
   ```
6. 恢复用户目录：
   ```bash
   restic -r rclone:gdrive:mac-backup-restic restore latest \
     --include /System/Volumes/Data/Users/wangshouren \
     --target ~/restore
   ```
7. 恢复本机 PostgreSQL 数据库（brew 安装的）：
   ```bash
   restic -r rclone:gdrive:mac-backup-restic restore latest \
     --include /System/Volumes/Data/opt/homebrew/var \
     --target ~/restore
   ```
8. 把 `~/restore/System/Volumes/Data/Users/wangshouren/` 下需要的内容拷到新家目录，例如 `Documents`、代码仓库、`.ssh`、各种配置。
9. 从 App Store 或官网重新安装 App。

警告：不要把旧的 `/Library`、`/System` 直接覆盖到新系统上，否则新系统可能无法启动。

## 恢复后重新开始备份

在新电脑上重新配置备份脚本、launchd 定时任务和「完全磁盘访问权限」，仓库地址继续用 `rclone:gdrive:mac-backup-restic`。仓库密码不变，旧快照全部保留。

## 必须另存的信息

| 信息 | 丢了会怎样 | 在原电脑上查看 |
|---|---|---|
| 仓库密码 | 备份永远无法解密，无法补救 | `security find-generic-password -s restic-gdrive-repo-password -w` |
| Client Secret | 可以在 Google Cloud 里重新生成 | `security find-generic-password -s rclone-gdrive-client-secret -w` |
