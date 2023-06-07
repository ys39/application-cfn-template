## はじめに

これまで、AWSでサービスを構築する際には、AWSのコンソール画面から手動で構築していました。

しかし、それだと、構築したサービスの構成を把握するのが難しいですし、構築したサービスを再利用することもできません。また、手動で構築すると、ミスが発生する可能性もあります。
さらに、手動で構築すると、構築したサービスの構成をドキュメント化するのも大変です。

そこで今回は、AWSのサービスの構成をテンプレート化することで、サービスの構成を把握しやすくし、再利用しやすくすることを目的として、Webサービスを展開する際の冗長的な構成を構築するためのCloudFormationテンプレートを作成しました。

※ IaCだとTerraformも有名ですが、今回は勉強のためにCloudFormationを利用しています。

## 構成図
![構成図](https://storage.googleapis.com/zenn-user-upload/95ac4d4830ba-20230606.png)

* Private SubnetにWebサーバ、DBサーバ、キャッシュサーバを配置。
* WebサーバへのアクセスはApplication Load Balancerを利用して行います。
* Public SubnetにNAT Gatewayを配置し、Private Subnetからインターネットへのアクセスを可能にします。

## Github
ソースコード一式です。
https://github.com/ys39/application-cfn-template

## 実行手順

下記のようにスタックファイルを分割しました。
1から6まで順番に実行することを想定しています。
1. 1-network.yaml
2. 2-securitygroup.yaml
3. 3-ec2.yaml
4. 4-rds.yaml
5. 5-elasticache.yaml
6. 6-applicationloadbalancer.yaml

## 各スタックファイルについて
各スタックファイルで作成しているリソースは下記の通りです。

1. 1-network.yaml
    ```
    * VPC
    * Subnet
    * InternetGateway
    * NatGateway
    * RouteTable
    * Route
    * SubnetとRouteTableのAssociation
    * PublicSubnetのNetworkAcl
    * PrivateSubnetのNetworkAcl
    * PublicSubnetとNetworkAclのAssociation
    * PrivateSubnetとNetworkAclのAssociation
   ```
2. 2-securitygroup.yaml
   ```
    * ALBのSecurityGroup
    * WebサーバのSecurityGroup
    * SSHサーバのSecurityGroup
    * RDSのSecurityGroup
    * ElastiCacheのSecurityGroup
   ```
    セキュリティグループはインバウンドルールとアウトバウンドルールにセキュリティグループを指定することができますが、循環参照を避けるために、セキュリティグループを作成した後に、インバウンドルールとアウトバウンドルールを設定しています。
    こちらのセキュリティグループを後ほどのスタックファイルで参照して紐づけていきます。
    ※ NatGatewayにはセキュリティグループは割り当てることはできません。

3. 3-ec2.yaml
   ```
    * EC2インスタンス
    * EC2インスタンスのキーペア
    * EC2インスタンスのUserData
    * EC2インスタンスのEIP
    * EC2インスタンスのEIPのAssociation
   ```

4. 4-rds.yaml
   ```
    * RDSインスタンス
    * RDSインスタンスのパラメータグループ
    * RDSインスタンスのサブネットグループ
   ```

5. 5-elasticache.yaml
   ```
    * ElastiCacheインスタンス
    * ElastiCacheインスタンスのパラメータグループ
    * ElastiCacheインスタンスのサブネットグループ
   ```

6. 6-applicationloadbalancer.yaml
   ```
    * Application Load Balancer
    * Application Load Balancerのリスナー
    * Application Load Balancerのターゲットグループ
   ```
## ファイアウォール
AWS上では、ファイアウォールとして、セキュリティグループとネットワークACLが利用できます。今回は、セキュリティグループとネットワークACLの両方を利用しています。
簡単に分けるとそれぞれ以下のような特徴があります。

* ネットワークACL
  * サブネット単位で設定する。
  * ステートレスのため、行きの通信と帰りの通信を許可する必要がある。

* セキュリティグループ
  * EC2などのインスタンス単位で設定する。
  * ステートフルのため、行きの通信を許可すれば、帰りの通信も許可される。

実際に構成したいのは、以下のような構成です。

### ネットワークACL
Public SubnetとPrivate SubnetでそれぞれネットワークACLを作成しました。

* Inbound Rule(Public Subnet)

    | Rule Number | Protocol | Port Range | Source     | Action |
    | ---------- | ---------- | ---------- | ---------- | --------------- |
    | 100        | TCP (6)    | 22         | ClientIP   | Allow           |
    | 101        | TCP (6)    | 80         | 0.0.0.0/0  | Allow           |
    | 102        | TCP (6)    | 443        | 0.0.0.0/0  | Allow           |
    | 103        | TCP (6)    | 1024-65535 | 0.0.0.0/0  | Allow           |
    | *          | ALL        | ALL        | 0.0.0.0/0  | Deny            |

* Outbound Rule(Public Subnet)

    | Rule Number | Protocol | Port Range | Destination | Action |
    | ---------- | ---------- | ---------- | ----------------- | --------------- |
    | 100        | TCP (6)    | 22         | 0.0.0.0/0         | Allow           |
    | 101        | TCP (6)    | 80         | 0.0.0.0/0         | Allow           |
    | 102        | TCP (6)    | 443        | 0.0.0.0/0         | Allow           |
    | 103        | TCP (6)    | 1024-65535 | 0.0.0.0/0         | Allow           |
    | *          | ALL        | ALL        | 0.0.0.0/0         | Deny            |

* Inbound Rule(Private Subnet)

    | Rule Number | Protocol | Port Range | Source     | Action |
    | ---------- | ---------- | ---------- | ---------- | --------------- |
    | 100        | TCP (6)    | 22         | 0.0.0.0/0  | Allow           |
    | 101        | TCP (6)    | 80         | 0.0.0.0/0  | Allow           |
    | 102        | TCP (6)    | 3306       | 0.0.0.0/0  | Allow           |
    | 103        | TCP (6)    | 11211      | 0.0.0.0/0  | Allow           |
    | 104        | TCP (6)    | 1024-65535 | 0.0.0.0/0  | Allow           |
    | *          | ALL        | ALL        | 0.0.0.0/0  | Deny            |

* Outbound Rule(Private Subnet)

    | Rule Number | Protocol | Port Range | Destination | Action |
    | ---------- | ---------- | ---------- | ----------------- | --------------- |
    | 100        | TCP (6)    | 80         | 0.0.0.0/0         | Allow           |
    | 101        | TCP (6)    | 443        | 0.0.0.0/0         | Allow           |
    | 102        | TCP (6)    | 1024-65535 | 0.0.0.0/0         | Allow           |
    | *          | ALL        | ALL        | 0.0.0.0/0         | Deny            |

### セキュリティグループ
各リソースに対してセキュリティグループを作成しました。
セキュリティグループのSourceとDestinationにセキュリティグループを指定することで、スケーラビリティとセキュリティが担保しています。

* ALBのSecurityGroup

    *Inbound Rule*

    | Type | Protocol | Port | Source |
    | ------ | --------- | ----- | ------ |
    | HTTP   | TCP       | 80    | 0.0.0.0/0 |
    | HTTPS  | TCP       | 443   | 0.0.0.0/0 |

    *Outbound Rule*

    | Type | Protocol | Port | Destination |
    | ------ | --------- | ----- | ------ |
    | HTTP   | TCP       | 80    | WebサーバのSecurityGroup |

* WebサーバのSecurityGroup

    *Inbound Rule*

    | Type | Protocol | Port | Source |
    | ------ | --------- | ----- | ------ |
    | HTTP   | TCP       | 80    | ALBのSecurityGroup  |
    | SSH    | TCP       | 22    | SSHサーバのSecurityGroup |

    *Outbound Rule*

    | Type | Protocol | Port | Destination |
    | ------ | --------- | ----- | ------ |
    | MYSQL  | TCP       | 3306  | RDSのSecurityGroup |
    | Custom TCP | TCP  | 11211 | ElastiCacheのSecurityGroup |
    | HTTP   | TCP       | 80    | 0.0.0.0/0 |
    | HTTPS  | TCP       | 443   | 0.0.0.0/0 |

* SSHサーバのSecurityGroup

    *Inbound Rule*

    | Type | Protocol | Port | Source |
    | ------ | --------- | ----- | ------ |
    | SSH    | TCP       | 22    | Client IP |

    *Outbound Rule*

    | Type | Protocol | Port | Destination |
    | ------ | --------- | ----- | ------ |
    | SSH    | TCP       | 22    | WebサーバのSecurityGroup |
    | HTTP   | TCP       | 80    | 0.0.0.0/0 |
    | HTTPS  | TCP       | 443   | 0.0.0.0/0 |

* RDSのSecurityGroup

    *Inbound Rule*

    | Type | Protocol | Port | Source |
    | ------ | --------- | ----- | ------ |
    | MYSQL  | TCP       | 3306  | WebサーバのSecurityGroup |

    *Outbound Rule*

    | Type | Protocol | Port | Destination |
    | ------ | --------- | ----- | ------ |
    | All Traffic | All | All | WebサーバのSecurityGroup |

* ElastiCacheのSecurityGroup

    *Inbound Rule*

    | Type | Protocol | Port | Source |
    | ------ | --------- | ----- | ------ |
    | Custom TCP | TCP | 11211 | WebサーバのSecurityGroup |

    *Outbound Rule*

    | Type | Protocol | Port | Destination |
    | ------ | --------- | ----- | ------ |
    | All Traffic | All | All | WebサーバのSecurityGroup |


## CloudFormationの備忘
* !Ref でリソースの参照ができる。
    ```
    PublicSubnetInboundHTTPRule:
        Type: AWS::EC2::NetworkAclEntry
        Properties:
        NetworkAclId: !Ref PublicSubnetNetworkAcl
        RuleNumber: 101
        Protocol: 6  # TCP
        RuleAction: allow
        Egress: false
        CidrBlock: 0.0.0.0/0
        PortRange:
            From: 80
            To: 80
    ```
* !Sub で文字列の置換ができる。
    ```
    InternetGateway:
        Type: AWS::EC2::InternetGateway
        Properties:
        Tags:
            - Key: Name
        　    Value: !Sub "${Prefix}InternetGateway"
    ```
* Exportで変数をエクスポートし、!ImportValue で他のスタックファイルで定義した値を参照できる。
    ```
    EC2WebSecurityGroup:
        Type: AWS::EC2::SecurityGroup
        Properties:
        GroupName: !Sub "${Prefix}EC2WebSecurityGroup"
        GroupDescription: Security Group for EC2(Web)
        VpcId: !ImportValue VPC # Import VPC ID from network stack
        Tags:
            - Key: Name
              Value: !Sub "${Prefix}EC2WebSecurityGroup"
    ```

## 所感
* 初めてCloudFormationを利用してみましたが、記述自体のハードルは低めに感じました。
ParametersやMappingsを利用することで、環境によって変更する必要がある値を変数化することができるようになるので、そこで汎用性を高めることができるのかと思います。
* 「yamlを作成 → 実行 → コンソールで確認 → 動作確認 → yamlを修正」のようなサイクルを回す作業を行いました。実行に時間がかかる（体感午前帯だと早く夜だと遅い？）ので、サイクルは15～20分程度かかりましたが、それでも手動で構築するよりは安定していると感じました。
* 参考[1]と参考[2]にある通り、AWSが提供しているサンプルテンプレートがあるので、大変参考になりました。
* 構成に複数のリソースがあるため、どのようにCloudFormationのスタックファイルを分割しようかと悩みましたが、今回は土台となる部分(network)、リソースのグループの作成(security group)、リソースの作成(ec2, rds, elasticache, alb)というように分類してみました。
* S3、Cloudfront、Route53、Autoscaling、IAMあたりのスタックファイルも作成してみたいです。

## 参考
[1] サンプルテンプレート
https://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/cfn-sample-templates.html

[2] aws-cloudformation-templates
https://github.com/awslabs/aws-cloudformation-templates