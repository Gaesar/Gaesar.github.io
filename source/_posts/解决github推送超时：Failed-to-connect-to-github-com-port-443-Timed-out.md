---
title: '解决github推送超时：Failed to connect to github.com port 443: Timed out.'
date: 2026-10-05 00:19:30
tags: bug复盘
author: Gaesar
---

## 解决办法：换成 SSH 协议

### 1. 先在本地生成 ssh 密钥，添加到 Github

1.打开 git bash，生成密钥（一路回车）。

```bash
ssh-keygen -t ed25519 -C "你github绑定的邮箱"
```

2.打开 `C:\Users\你的用户名\.ssh\id_ed25519.pub`，复制全部公钥文本。

3.Github 添加公钥，头像 → Settings → SSH and GPG keys → New SSH key

### 2. 启用 SSH 443 端口方案

在 `C:\Users\你的用户名\.ssh\` 新建 `config`（无后缀，不是 config.txt）

```
Host github.com
  HostName ssh.github.com
  User git
  Port 443
```

### 3.把 origin 远程地址改成 SSH

执行 `git remote -v`查看当前仓库远程地址

大概率输出类似：

```
origin  https://github.com/aaa/xxx.git (fetch)
origin  https://github.com/aaa/xxx.git (push)
```

把 origin 远程地址改成 SSH

```bash
git remote set-url origin git@github.com:aaa/xxx.git
```

执行 `git remote -v` 确认

```
origin  git@github.com:aaa/xxx.git (fetch)
origin  git@github.com:aaa/xxx.git (push)
```

之后再执行 `git push`，就走 SSH 协议，不再走 443 https

