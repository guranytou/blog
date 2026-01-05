---
title: "urfave/cliのチュートリアルをやってみた"
date: 2021-11-16T00:05:27+09:00
author: "guranytou"
tags: ["Go"]
categories: ["tech"]
---

# 前書き
Goの勉強をするのにCLIをいくつか作りたくて、[urfave/cli](https://github.com/urfave/cli)のチュートリアルをサクッとやりました。  
リッチなCLIを作りたいのであれば[cobra](https://github.com/spf13/cobra)がいいんでしょうが、今回を含めていくつかのCLIはスナック感覚で作りたかったのでurfave/cliを選定しました。

# 書いたコード
https://github.com/guranytou/GoStudy/tree/master/greet
雑に勉強する用リポジトリにテキトーにサクッと投げた図。  

```golang:main.go
package main

import (
	"fmt"
	"log"
	"os"

	"github.com/urfave/cli/v2"
)

func main() {
	app := cli.NewApp()
	app.Name = "greet"
	app.Usage = "Give a goo greeting."
	app.Version = "0.0.1"
	app.Action = func(c *cli.Context) error {
		fmt.Println("Hello")
		return nil
	}

	err := app.Run(os.Args)
	if err != nil {
		log.Fatal(err)
	}
}
```

# 動作確認
- まずはそもそも問題なく動くのか確認。
```
$ go run main.go
Hello
```

- 作成したコマンドをインストールする
```
$ go install
```

- 実行してみる
```
$ greet
Hello
```

# まとめとか
サクッと作れました。以下のコマンドをサクッと作ってみてGoに慣れていこうかなあと思ってます。
- おみくじ
    - よくあるネタ
- 天気予報
    - 最初はコマンドを打つと東京だけ教えてくれる
    - ゆくゆくは地名を指定するとそこの天気を教えてくれるようになる、といいなあ
- terraform関連ツール
    - terraform&Goの勉強をかねて


