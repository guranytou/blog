---
title: "Pulumiを利用してAWS上でWebサーバ一式を立ててみた その２"
date: 2022-05-27T00:33:09+09:00
author: "guranytou"
tags: ["Pulumi", "AWS", "Go"]
categories: ["Tech"]
---

前回： [Pulumiを利用してAWS上でWebサーバ一式を立ててみた その１](https://guranytou.github.io/blog/post/20220523-create-aws-resource-use-pulumi-1/)

## はじめに
今回は実際にALB-EC2でWebサーバを立ててみます。  
構成は[以前Terraformで作成したもの](https://zenn.dev/guranytou/articles/7e6680fc93ce16)と同じものとします。  
今回のソースコード：[Github](https://github.com/guranytou/TestPulumi)

## 書いたコードの解説
1. ネットワーク周りのコードを作成する  
VPC、Subnetを作成します。
```go
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

		pubSub1a, err := ec2.NewSubnet(ctx, "pubSub1a", &ec2.SubnetArgs{
			VpcId:            vpc.ID(),
			CidrBlock:        pulumi.String("10.100.0.0/24"),
			AvailabilityZone: pulumi.String("ap-northeast-1a"),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_public_1a"),
			},
		})
		if err != nil {
			return err
		}

		pubSub1c, err := ec2.NewSubnet(ctx, "pubSub1c", &ec2.SubnetArgs{
			VpcId:            vpc.ID(),
			CidrBlock:        pulumi.String("10.100.1.0/24"),
			AvailabilityZone: pulumi.String("ap-northeast-1c"),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_public_1c"),
			},
		})
		if err != nil {
			return err
		}

		priSub1a, err := ec2.NewSubnet(ctx, "priSub1a", &ec2.SubnetArgs{
			VpcId:            vpc.ID(),
			CidrBlock:        pulumi.String("10.100.100.0/24"),
			AvailabilityZone: pulumi.String("ap-northeast-1a"),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_private_1a"),
			},
		})
```

2. GW関連のリソースを作成する  
IGW/NAT GWと、NAT GWにアタッチするEIPを作成します。
```go
		eip, err := ec2.NewEip(ctx, "eip", &ec2.EipArgs{
			Vpc: pulumi.Bool(true),
		})
		if err != nil {
			return err
		}

		igw, err := ec2.NewInternetGateway(ctx, "igw", &ec2.InternetGatewayArgs{
			VpcId: vpc.ID(),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_igw"),
			},
		})
		if err != nil {
			return err
		}

		natgw, err := ec2.NewNatGateway(ctx, "natgw", &ec2.NatGatewayArgs{
			AllocationId: eip.ID(),
			SubnetId:     pubSub1a.ID(),
		}, pulumi.DependsOn([]pulumi.Resource{eip}))
		if err != nil {
			return err
		}

		pubRoutaTable, err := ec2.NewRouteTable(ctx, "pubRouteTable", &ec2.RouteTableArgs{
			VpcId: vpc.ID(),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_route_table_public"),
			},
		})
		if err != nil {
			return err
		}
```

3. GW用のルートテーブルを作成する  
ルートテーブルにて、各サブネットとInternet Gateway及びNAT Gatewayとの経路を用意します。
```go
_, err = ec2.NewRoute(ctx, "pubRoute", &ec2.RouteArgs{
			RouteTableId:         pubRoutaTable.ID(),
			DestinationCidrBlock: pulumi.String("0.0.0.0/0"),
			GatewayId:            igw.ID(),
		})
		if err != nil {
			return err
		}

		_, err = ec2.NewRouteTableAssociation(ctx, "pubRoute1a", &ec2.RouteTableAssociationArgs{
			SubnetId:     pubSub1a.ID(),
			RouteTableId: pubRoutaTable.ID(),
		})
		if err != nil {
			return err
		}

		_, err = ec2.NewRouteTableAssociation(ctx, "pubRoute1c", &ec2.RouteTableAssociationArgs{
			SubnetId:     pubSub1c.ID(),
			RouteTableId: pubRoutaTable.ID(),
		})
		if err != nil {
			return err
		}

		priRouteTable, err := ec2.NewRouteTable(ctx, "priRouteTable", &ec2.RouteTableArgs{
			VpcId: vpc.ID(),
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example_route_table_private"),
			},
		})
		if err != nil {
			return err
		}

		_, err = ec2.NewRoute(ctx, "priRoute", &ec2.RouteArgs{
			RouteTableId:         priRouteTable.ID(),
			DestinationCidrBlock: pulumi.String("0.0.0.0/0"),
			NatGatewayId:         natgw.ID(),
		})
		if err != nil {
			return err
		}

		_, err = ec2.NewRouteTableAssociation(ctx, "priRoute1a", &ec2.RouteTableAssociationArgs{
			SubnetId:     priSub1a.ID(),
			RouteTableId: priRouteTable.ID(),
		})
		if err != nil {
			return err
		}
```

4. ALB関連リソースを作成する  
ALB本体、listener、TG、ALBにアタッチするSGを作成します。
```go
		sgForALB, err := ec2.NewSecurityGroup(ctx, "sgForALB", &ec2.SecurityGroupArgs{
			Name:  pulumi.String("example_sg_for_ALB"),
			VpcId: vpc.ID(),
			Ingress: ec2.SecurityGroupIngressArray{
				&ec2.SecurityGroupIngressArgs{
					FromPort: pulumi.Int(80),
					ToPort:   pulumi.Int(80),
					Protocol: pulumi.String("tcp"),
					CidrBlocks: pulumi.StringArray{
						pulumi.String("0.0.0.0/0"),
					},
				},
			},
			Egress: ec2.SecurityGroupEgressArray{
				&ec2.SecurityGroupEgressArgs{
					FromPort: pulumi.Int(0),
					ToPort:   pulumi.Int(0),
					Protocol: pulumi.String("-1"),
					CidrBlocks: pulumi.StringArray{
						pulumi.String("0.0.0.0/0"),
					},
				},
			},
		})
		if err != nil {
			return err
		}

		alb, err := lb.NewLoadBalancer(ctx, "ALB", &lb.LoadBalancerArgs{
			Name:             pulumi.String("example"),
			LoadBalancerType: pulumi.String("application"),
			SecurityGroups: pulumi.StringArray{
				sgForALB.ID(),
			},
			Subnets: pulumi.StringArray{
				pubSub1a.ID(),
				pubSub1c.ID(),
			},
			Tags: pulumi.StringMap{
				"Name": pulumi.String("example"),
			},
		})
		if err != nil {
			return err
		}

		httpTG, err := lb.NewTargetGroup(ctx, "httpTG", &lb.TargetGroupArgs{
			Name:     pulumi.String("HTTPTG"),
			Port:     pulumi.Int(80),
			Protocol: pulumi.String("HTTP"),
			VpcId:    vpc.ID(),
			HealthCheck: lb.TargetGroupHealthCheckArgs{
				Path:     pulumi.String("/"),
				Matcher:  pulumi.String("403"),
				Port:     pulumi.String("80"),
				Protocol: pulumi.String("HTTP"),
			},
		})
		if err != nil {
			return err
		}

		_, err = lb.NewListener(ctx, "listener", &lb.ListenerArgs{
			LoadBalancerArn: alb.Arn,
			Port:            pulumi.Int(80),
			Protocol:        pulumi.String("HTTP"),
			DefaultActions: lb.ListenerDefaultActionArray{
				&lb.ListenerDefaultActionArgs{
					Type:           pulumi.String("forward"),
					TargetGroupArn: httpTG.Arn,
				},
			},
		}, pulumi.DependsOn([]pulumi.Resource{alb}))
		if err != nil {
			return err
		}
```

5. EC2関連リソースを作成する
EC2本体、EC2にアタッチするSGを作成します。  
また今回はUserDataを利用して、EC2起動時にApacheをインストール&起動させます。  
```go
sgForInstance, err := ec2.NewSecurityGroup(ctx, "sgForInstance", &ec2.SecurityGroupArgs{
			Name:  pulumi.String("example_sg_for_instance"),
			VpcId: vpc.ID(),
			Ingress: ec2.SecurityGroupIngressArray{
				&ec2.SecurityGroupIngressArgs{
					FromPort: pulumi.Int(80),
					ToPort:   pulumi.Int(80),
					Protocol: pulumi.String("tcp"),
					SecurityGroups: pulumi.StringArray{
						sgForALB.ID(),
					},
				},
			},
			Egress: ec2.SecurityGroupEgressArray{
				&ec2.SecurityGroupEgressArgs{
					FromPort: pulumi.Int(0),
					ToPort:   pulumi.Int(0),
					Protocol: pulumi.String("-1"),
					CidrBlocks: pulumi.StringArray{
						pulumi.String("0.0.0.0/0"),
					},
				},
			},
		})
		if err != nil {
			return err
		}

		f, err := os.Open("install_apache.sh")
		if err != nil {
			return err
		}

		b, err := ioutil.ReadAll(f)
		if err != nil {
			return err
		}

		enc := base64.StdEncoding.EncodeToString(b)

		ins, err := ec2.NewInstance(ctx, "instance", &ec2.InstanceArgs{
			Ami:          pulumi.String("ami-0992fc94ca0f1415a"),
			InstanceType: pulumi.String("t2.micro"),
			SubnetId:     priSub1a.ID(),
			VpcSecurityGroupIds: pulumi.StringArray{
				sgForInstance.ID(),
			},
			UserDataBase64: pulumi.String(enc),
		})
		if err != nil {
			return err
		}

		_, err = lb.NewTargetGroupAttachment(ctx, "TGattach", &lb.TargetGroupAttachmentArgs{
			TargetGroupArn: httpTG.Arn,
			TargetId:       ins.ID(),
			Port:           pulumi.Int(80),
		})
		if err != nil {
			return err
		}
```

## Goで書いてみての感想
 Terraformと同じような感覚で書けるので、Terraform書ける人はパッと書けるなと思いました。  
 インフラ構築を汎用言語で選択する場合、フロントエンドやバックエンドで利用している言語と同じ言語を使えるので、システム全体の統一性が上がるのと学習コストの圧縮ができるのかなあと感じました。  
 （個人的にはTerraformのシンプルさが好きなので、汎用言語を選択することで大きなメリットが出ない限りは、Terraformでもいいのでは？と思いますが）  
 今回、main関数に全部ベタで書いたので、次回はTerraformのモジュールと同じ分け方で関数化してみようと思います。