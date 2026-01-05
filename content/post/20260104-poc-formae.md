---
title: "formaeを試してみた"
date: 2026-01-04T22:17:21+09:00
draft: false
author: "guranytou"
tags: ["formae", "IaC"]
categories: ["Tech"]
---

## はじめに
会社の同僚から「formaeという新しいIaCツールがあるよ」というのを教えてもらったので、年末年始休暇で遊んでみた。

## formaeとは
大体は多分以下二つを読めばわかるはず
- [formae公式サイト](https://platform.engineering/formae)
- [ローンチ時のニュース](https://www.infoq.com/jp/news/2025/12/iac-formae/)

簡単にまとめると、2025/10/22にPlatform Engineering Labs社がローンチした新しいIaCツール。
既存のIaCツールが抱える課題を解決するために開発した、とのこと。

## Quick Startを試してみる
まずは[Quick Start](https://docs.formae.io/en/latest/)を試してみることにした。あとは気になったコマンドもいくつか試してみる。
S3コードは以下のような感じ（Quick Startとほぼ変わらないが）

```
amends "@formae/forma.pkl"
import "@formae/formae.pkl"
import "@aws/aws.pkl"
import "@aws/s3/bucket.pkl"
import "@formae/ext/random.pkl"

local randomId = random.id(6).toString()

properties {
  name = new formae.Prop {
    flag = "name"
    default = "my-name"
  }
}

forma {
  new formae.Stack {
    label = "my-stack"
    description = "Stack for my resources"
  }

  new formae.Target {
    label = "my-aws-apne1"
    config = new aws.Config {
      region = "ap-northeast-1"
    }
  }

  // My Resources
  new bucket.Bucket {
    label = "my-s3-bucket"
    bucketName = "my-first-bucket-\(randomId)"
  }
}
```

コードを記述した後に、`formae apply --mode reconcile --watch main.pkl`を実行すると、S3が作成される。
ちなみにmodeオプションはreconcile/patchのどちらかを指定する必要があり、
- reconcile: 実際のインフラリソースとコードを比較し、それらを一致させるモード。コードにないAWSリソースは削除され、Driftが発生した場合にはリソース側がコードに合わせて変更される。
- patch: リソースの作成・更新のみ行う。既存インフラに対して影響を与えないように振る舞う。

詳しくは[公式ドキュメントのApply modes explained](https://docs.formae.io/en/latest/formae-101/workflows/#apply-modes-explained)参照。

### つまづいたポイント
`formae project init`をした時にできる`PklProject`で依存関係のあるpluginを管理している。
この時、formaeは自動でdependenciesに追加されていたがawsは追加されていなかったので、以下のように記述することで依存関係に関するエラーが解消できた。
```
dependencies {
  ["formae"] {
    uri = "package://hub.platform.engineering/plugins/pkl/schema/pkl/formae/formae@0.76.0"
  }

  // 追記した記述
  ["aws"] {
    uri = "package://hub.platform.engineering/plugins/aws/schema/pkl/aws/aws@0.76.0"
  }
}
```

## Driftを発生させて眺めてみる
事前にどういう変更が発生するのか`--simulate`オプションを付与すると確認できる。
先ほど作成したS3バケットにEnvタグを付与した上で、`--simulate`オプションを付与してapplyしてみると以下のように出力された。
```
// S3バケットに付与されているタグの確認
❯ aws s3api get-bucket-tagging --bucket my-first-bucket-647306 --query 'TagSet[*].[Key,Value]' --output table
-----------------------------------------
|           GetBucketTagging            |
+----------------------+----------------+
|  FormaeResourceLabel |  my-s3-bucket  |
|  Env                 |  dev           |
|  FormaeStackLabel    |  my-stack      |
+----------------------+----------------+

// driftが発生しているか確認
❯ formae apply --force --mode reconcile --simulate main.pkl

Command will
└── update resource my-s3-bucket
    ├── of type AWS::S3::Bucket
    ├── in stack my-stack
    └── by doing the following:
        └── remove Tag "Env"
Command will not continue - simulation only
```

applyしてみると以下のようになった。
```
❯ formae apply --force --mode reconcile --watch main.pkl

Command will
└── update resource my-s3-bucket
    ├── of type AWS::S3::Bucket
    ├── in stack my-stack
    └── by doing the following:
        └── remove Tag "Env"
This operation will update 1 resource(s).

Do you want to continue? (Y): Y

The asynchronous command has started on the formae agent.

Watching commands status (refreshing every 2s)...

┌────────────── ┬──── ┬──── ┬────┬───┬─────┬──── ┬─── ┬───┬─────┬───────── ┬───┐
│             ID              │ Command │ Status  │ Change │ Wait │ Progress │ Success │ Retry │ Fail │ Canceled │    Started At     │ Time │
├────────────── ┼──── ┼──── ┼────┼───┼─────┼──── ┼─── ┼───┼─────┼───────── ┼───┤
│ 37nRFBoKnBq649tXLtfkrDfRsDM │ apply   │ Success │ 1      │ 0    │ 0        │ 1       │ 0     │ 0    │ 0        │ 01/04/2026 2:25PM │ 43s  │
└────────────── ┴──── ┴──── ┴────┴───┴─────┴──── ┴─── ┴───┴─────┴───────── ┴───┘

// S3タグの確認
❯ aws s3api get-bucket-tagging --bucket my-first-bucket-647306 --query 'TagSet[*].[Key,Value]' --output table
-----------------------------------------
|           GetBucketTagging            |
+----------------------+----------------+
|  FormaeResourceLabel |  my-s3-bucket  |
|  FormaeStackLabel    |  my-stack      |
+----------------------+----------------+
```

## Discoveryを眺めてみる
formae管理されていないリソースがどのように出力されるのか興味があったので、試しに眺めてみる。
`formae inventory resources`コマンドを利用すると確認できるとのことなので実行してみると以下のように出力された。
```
❯ formae inventory resources --query="managed:false" --max-results 3

┌─────────── ┬───── ┬───────────── ┬─────────── ┐
│       NativeID        │   Stack   │           Type            │         Label         │
├─────────── ┼───── ┼───────────── ┼─────────── ┤
├─────────── ┼───── ┼───────────── ┼─────────── ┤
│ dopt-ceaec4a5         │ unmanaged │ AWS::EC2::DHCPOptions     │ dopt-ceaec4a5         │
├─────────── ┼───── ┼───────────── ┼─────────── ┤
│ igw-0a497b62          │ unmanaged │ AWS::EC2::InternetGateway │ igw-0a497b62          │
├─────────── ┼───── ┼───────────── ┼─────────── ┤
│ igw-0dd4e59e869dd3a62 │ unmanaged │ AWS::EC2::InternetGateway │ igw-0dd4e59e869dd3a62 │
└─────────── ┴───── ┴───────────── ┴─────────── ┘

Summary: Showing 3 of 93 total resources (use --max-results 93 to see all)
```

Terraformだと気合いで探すか、default tagsを利用してtagを付与した上でtagがないものを探す感じになるので、コマンドでサクッと探せるのは相当体験として良い。

## 所感
ツールとしては（当然ではあるんだけど）まだまだ未成熟だなあと感じるところがいくつかあったけど、「実リソースが現実なので常にIaCツールがそれに追随する機能を持っている」部分とかなぜそういう設計なのかの思想とかはだいぶ共感できるし、理解できるなあと思った。特に、CIで定期的にDiscovery/Synchronizationを回して、管理されてないリソースやコードと実リソースの差分があれば自動PR作成させるとかできそうなのはとても良い。管理されていないリソースを探してimportする作業はとてもつらいので。

実際に現場で利用する場合、formae agentは管理したいクラウドインフラのあるアカウントの中で管理する必要があるけどそこのアーキテクチャがまだピンときてないなあと思ったのと、管理したいリソースがたくさんある場合にどういう挙動になるのか（例：Terraformだとstateが大きすぎるとPlanに時間がかかる）とかは思ったりした。
あと今回ゴリゴリにコードを書いたわけではないので、PKLの書き味やコードをどう管理するのが良さそうか（ディレクトリやファイル構造など）とかもなんとも言えないなと思った。気が向いたらコードを実際に書いてみて考えたいかも。
