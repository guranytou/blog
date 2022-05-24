---
title: "天気予報CLIを作るついでに色々勉強した話"
date: 2022-01-10T17:00:20+09:00
author: "guranytou"
tags: ["Go"]
categories: ["Tech"]
---

## はじめに
外部APIを利用してCLI作りをしたいなあと思い、天気予報を確認できるCLIを作ることにしました。  

```bash:表示例
$ tenki tokyo,jp
時刻: 2022-01-10 17:14:44 +0900 JST
場所: Tokyo
天気: くもり
気温: 8.1
```

## コードの解説
本体のコードはこちら: [Github](https://github.com/guranytou/GoStudy/tree/master/tenki)

### Weather MAP APIを利用する
#### APIのtokenを環境変数に格納する
Githubに上げるにあたり、tokenは環境変数にしたいなと思いました。  
なので、`godotenv`を利用して.envに格納したtokenを呼び出しています。
```go
	err := godotenv.Load(".env")
	if err != nil {
		panic("Error loading .env file")
	}
	token := os.Getenv("API_TOKEN")
```
```env
API_TOKEN=YOUR_TOKEN
```

#### 都市情報の設定とWeather MAP APIへクエリを投げる
今回のCLIでは `tenki tokyo,jp`のように都市名をコマンドで投げるとその都市の天気予報が見れるように、また`tenki`のみで投げた場合には東京の天気を見れるようにしたいと思いました。  
そのためWeather MAP APIに天気予報を確認したい都市を投げるために、以下のようにしました。  
```go
	values := url.Values{}
	values.Set("APPID", token)
	if city == "" {
		values.Set("q", "tokyo,jp")
	} else {
		values.Set("q", city)
	}
```

そして以下のように整形し、Weather MAP APIを投げています。
```go
	endPoint := "https://api.openweathermap.org/data/2.5/weather"

	resp, err := http.Get(endPoint + "?" + values.Encode())
	if err != nil {
		panic(err)
	}
	defer resp.Body.Close()
```

#### 天気情報を受け取って表示する
現在時刻、場所、天気、気温を表示するようにしたかったため、以下の構造体を準備しました。  
```go
type OWMAPIResponce struct {
	Weather []Weather `json:"weather"`
	Main    Main      `json:"main"`
	Dt      int64     `json:"dt`
	Name    string    `json:"name"`
}

type Main struct {
	Temp float64 `json:"temp"`
}

type Weather struct {
	Main        string `json:"main"`
	Description string `json:"description"`
}
```

天気情報は日本語にしたかったのと、気温は摂氏表示にしたかったので、表示するにあたり以下のようにしています。
```go
	fmt.Printf("時刻: %s\n", time.Unix(apiRes.Dt, 0))
	fmt.Printf("場所: %s\n", apiRes.Name)

	if apiRes.Weather[0].Main == "Clear" {
		fmt.Println("天気: 晴れ")
	} else if apiRes.Weather[0].Main == "Clouds" {
		fmt.Println("天気: くもり")
	} else if apiRes.Weather[0].Main == "Rain" {
		fmt.Println("天気: 雨")
	} else if apiRes.Weather[0].Main == "Snow" {
		fmt.Println("天気: 雪")
	} else {
		fmt.Printf("天気: %s\n", apiRes.Weather[0].Main)
	}

	fmt.Printf("気温: %g\n", math.Floor((apiRes.Main.Temp-273.15)*10)/10)
```

### やってみて感想
- JSONを取り出す際には必要な情報だけで良い（いらない情報は定義しなくていい）ことがわかったのでなるほどと思いました
- godotenvとても便利

次はTerraform関連のツールを「車輪の再実装」して勉強してみようと思います。  
もしくはToDoアプリをまた作ってみて、API作りについて理解を深めるか。