---
title: "Github Actionsで職務経歴書.mdをPDFに変換するようにした話"
date: 2022-01-02T17:27:08+09:00
author: "guranytou"
tags: ["Github Actions"]
categories: ["tech"]
---

## 前書き
職務経歴書がだいぶ古くなっていたので、最新化ついでにGithub Pages化とPDF化すればいいじゃんと思い、.mdで作成しました。  
Github Pages化はこのブログ用のGithub Actionsを利用してできるようにしたので、今回はPDF化するactionを作ることにしました。

## コード
```yml:sample
name: md-to-pdf

on:
  push:
    branches:
      - main
    paths:
      - "content/resume.md"

jobs:
  build:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '14'
      - run: |
          sudo apt install fonts-noto
          npm install
          npm i -g md-to-pdf
          md-to-pdf content/resume.md
      - uses: actions/upload-artifact@v2
        with:
          name: resume
          path: content/resume.pdf
```

## 軽い解説
```yml:sample
on:
  push:
    branches:
      - main
    paths:
      - "content/resume.md"
```
mainブランチにてresume.mdがpushされた時のみ動くようにしました。   

```yml:sample
- uses: actions/setup-node@v2
    with:
        node-version: '14'
```
今回は[md-to-pdf](https://github.com/simonhaenisch/md-to-pdf)を利用してPDF化したかったので、動作に必要なnode.jsを準備します。

```yml:sample
- run: |
    sudo apt install fonts-noto
    npm install
    npm i -g md-to-pdf
    md-to-pdf content/resume.md
```
Ubuntuイメージにはデフォルトで日本語フォントが入っていないので、日本語フォントをインストール。  
その後にmd-to-pdfをインストールし、md -> pdfに変換します。

```yml:sample
- uses: actions/upload-artifact@v2
with:
    name: resume
    path: content/resume.pdf
```
変換したPDFをダウンロードしたいので、artifactとしてアップロードします。

## まとめ
ネットを調べると意外とmd-to-pdfをそのままGithub Actionsに組み込んで実施しているymlがなかったので記事にしました。  
そのうちヘッダーに作成日、フッターにページ数を入れられるようにしたいなあと思います。