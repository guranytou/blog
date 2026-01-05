---
title: "Pulumiを利用してAWS上でWebサーバ一式を立ててみた その１"
date: 2022-05-23T09:55:46+09:00
author: "guranytou"
tags: ["Pulumi", "AWS", "Go"]
categories: ["Tech"]
---

## はじめに
普段はTerraformを利用していますが、PulumiだったりSDKだったり、汎用言語を使ってAWSリソースを作ってみたかったのでやってみました。  
言語は現在勉強しているGoを使いました。

## Pulumiを導入してみる
Macで開発しているのでbrew installしました。
```
brew install pulumi
```

## チュートリアル
https://www.pulumi.com/docs/get-started/aws/  
このページにチュートリアルがあるのでこのガイドに従ってやっていきました（内容は割愛します）  

## 試し書き
チュートリアルでS3バケットを作ることができました。  
ということで、試しに公式Docを読みながらVPCを1つ作成してみます。 

1. コードを準備する
```go
package main

import (
	"github.com/pulumi/pulumi-aws/sdk/v5/go/aws/ec2"
	"github.com/pulumi/pulumi/sdk/v3/go/pulumi"
)

func main() {
	pulumi.Run(func(ctx *pulumi.Context) error {
		vpc, err := ec2.NewVpc(ctx, "example_vpc", &ec2.VpcArgs{
			CidrBlock: pulumi.String("10.100.0.0/16"),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_vpc"),
			},
		})
		if err != nil {
			return err
		}

        ctx.Export("vpcid: ", vpc.ID())
        return nil
	})
}
```

2. `pulumi preview`で見てみる  
`pulumi preview`を実行し、作成できるか確認してみます。
```bash
> pulumi preview
Previewing update (dev)

View Live: https://app.pulumi.com/xxx

     Type                 Name            Plan       
 +   pulumi:pulumi:Stack  TestPulumi-dev  create     
 +   └─ aws:ec2:Vpc       example_vpc     create     
 
Resources:
    + 2 to create
```

3. `pulumi up`で実際に作成する  
```bash
> pulumi up
Previewing update (dev)

View Live: https://app.pulumi.com/xxx

     Type                 Name            Plan       
 +   pulumi:pulumi:Stack  TestPulumi-dev  create     
 +   └─ aws:ec2:Vpc       example_vpc     create     
 
Resources:
    + 2 to create

Do you want to perform this update? yes
Updating (dev)

View Live: https://app.pulumi.com/xxx

     Type                 Name            Status      
 +   pulumi:pulumi:Stack  TestPulumi-dev  created     
 +   └─ aws:ec2:Vpc       example_vpc     created     
 
Outputs:
    vpcid: : "vpc-0b8a7faf1fb502f03"

Resources:
    + 2 created

Duration: 6s
```

4. コンソール上で確認してみる  
無事にできていることがわかりました。
![AWSコンソールでの確認結果](https://raw.githubusercontent.com/guranytou/blog/main/content/post/image/image_202205241031.png)

## まとめ
Terraformと似たような感じで書けるので、そこまで苦労しませんでした。  
次の記事では実際にEC2 - ALBでのWebサーバを構築していきます。