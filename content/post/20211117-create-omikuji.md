---
title: "簡易なおみくじができるCLIを作った"
date: 2021-11-17T00:35:15+09:00
author: "guranytou"
tags: ["Go"]
categories: ["tech"]
---

## 前書き
Goの勉強をするのにスナック感覚でCLIを作ろうその１として、よくあるおみくじCLIを作りました。  
`math/rand`のお勉強も兼ねてます。

## 書いたコード
### 置き場所
https://github.com/guranytou/GoStudy/tree/master/omikuji

### コード
```golang:main.go
package main

import (
	"fmt"
	"log"
	"math/rand"
	"os"
	"time"

	"github.com/urfave/cli/v2"
)

func main() {
	app := cli.NewApp()
	app.Name = "omikuji"
	app.Usage = "Play omikuji"
	app.Version = "0.0.1"
	app.Action = func(c *cli.Context) error {
		omikuji()
		return nil
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}

func omikuji() {
	rand.Seed(time.Now().UnixNano())
	luck := rand.Intn(12)
	switch luck {
	case 0:
		fmt.Println("大吉")
	case 1, 2:
		fmt.Println("中吉")
	case 3, 4, 5:
		fmt.Println("小吉")
	case 6, 7, 8:
		fmt.Println("末吉")
	case 9, 10:
		fmt.Println("吉")
	case 11:
		fmt.Println("凶")
	}
}
```

難しいことをせずにをモットーに作ったので、色々ベタ書きです。  
`omikuji()`はそのうちリファクタリングしたいなあ（願望）。

### 表示結果
```
$ omikuji
大吉
```
普通に実行したら大吉が出ました。やったね。

## `math/rand`について
### `math/rand`を使った理由
Goで乱数生成に関する標準パッケージには`math/rand`と`crypto/rand`の2つがあります。  
今回はともかくサクッと軽いCLIが作りたかったので、`math/rand`を選択しました。  
セキュアなものを作りたければ`crypto/rand`を利用した方が良さそうです。

### `math/rand`の簡単な解説
乱数生成のためにはシード値を生成する必要があるので、時間を利用して作成します。  
※シード値生成に時間を利用するのはよくある話なので割愛します。   
```
rand.Seed(time.Now().UnixNano())
```
今回はswitchを利用しておみくじを振り分けていきたいので、`rand.Intn()`で、 0 <= n < 12の数をランダムに生成します。  
`rand.Intn(x)`と書くと、0以上xまでのInt型の値を生成します。

### 参考にしたページ
https://xn--go-hh0g6u.com/pkg/math/rand/  
https://qiita.com/crifff/items/b116e6235fedcd18e0de

## まとめ
今回は乱数生成方法について勉強&整理しました。  
実際によく使うのは`crypto/rand`なのかなあという気もするので、次回乱数生成することがあったらそちらを使おうかなと思います。