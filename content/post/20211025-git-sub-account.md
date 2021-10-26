---
title: "gitのサブアカウント設定方法作成"
date: 2021-10-26T00:56:23+09:00
author: "guranytou"
tags: ["git"]
categories: ["tech"]
---

# 前書き
同じPC上で複数のgitアカウントを利用したくなったので、個人的に良さそうなアカウントの切り替え方法についてメモします。

# やり方
1. サブ用ディレクトリを作成する
2. サブ用ディレクトリの中に.sshを作成し、鍵を生成する
3. サブ用のgit configを作成し、設定を投入する。
```
# サブ用のgit configを作成
touch ~/.gitconfig_sub

# 設定を投入する
[user]
	name = [your_user_name]
	email = [your_email_address]

```

4. `~/.gitconfig`に設定を投入する
```
[includeIf "gitdir:~/your/sub/dir/"]
	path = ~/.gitconfig_sub
```

5. `~/.ssh/config`に設定を投入する
```
Host github github.com
  HostName github.com
  IdentityFile ~/your/sub/dir/.ssh/id_rsa
  User git
```