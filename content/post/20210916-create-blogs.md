---
title: "Hugo + Github Pages + Github Actionsでブログを作った"
date: 2021-09-16T22:49:02+09:00
draft: false
author: "guranytou"
tags: ["Hugo", "Github Actions"]
categories: ["tech"]
---

## まえがき
Zenn.devに書くほどではないけど、自分では備忘録的なものが欲しくてブログを作った。  
そして作ってからn週間経ってようやく記事を書くのであった…。  

## 環境
- Windows10
- WSL2（Ubuntu20.04）

## やったこと
ざっくりとやったことをメモ。

### Hugoの導入
公式から最新版をインストールする。

```
wget https://github.com/gohugoio/hugo/releases/download/v0.88.0/hugo_extended_0.88.0_Linux-64bit.deb
sudo apt-get install ./hugo_extended_0.88.0_Linux-64bit.deb
```

### Hugoの設定
まずはサイトを新規作成する。
```
hugo new site blog
```

次に好きなthemesを探してダウンロードする。  
シンプルなデザインがよかったので、hugo-paperを選択した。  
themesリポジトリ：https://github.com/nanxiaobei/hugo-paper
```
git submodule add https://github.com/nanxiaobei/hugo-paper themes/paper
```

あとはconfig.tomlを設定する（設定内容はthemesリポジトリを参考）。  

### Github Actionsの設定
記事をpushしたら自動的にGithub Pagesに上げてほしかったので、Github Actionsを設定。
[Hugo公式](https://gohugo.io/hosting-and-deployment/hosting-on-github/#build-hugo-with-github-action)にyamlの書き方があるのでこれを参考にした。

## 感想
途中何故かHugoのthemesがローカルでもGithub上でも反映されなくて困った。  
いろいろいじってたら反映されたけど、理由は不明のまま。。。  

