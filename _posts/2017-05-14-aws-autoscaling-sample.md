---
layout: post
title: AWS AutoScaling Sample
tags: "cloudwatch, cloudformation, aws"
comments: true
---

AWS AutoScaling のサンプルを AWS CloudFormation で作成します。

1. [全体概要図](#introduction)
2. [CloudFormation 概要](#cfn-intro)
3. [CloudFormation テンプレート](#cfn-template)
4. [AutoScaling の定義](#def-autoscaling)

---

<div id="introduction" />

### 1. 全体概要図

AWS のサービスの一つとしてある条件に応じて EC2 インスタンスの scale in, scale out が自動で行われる AutoScaling というものがあります。

ここでは AutoScaling のサンプルとして以下のようなインフラ、サービスを CloudFormation で構築します。

<img
  src="/images/aws-cloudwatch-sample/aws-cloudwatch-sample-summary.png"
  title="summary of cloudwatch sample"
  alt="summary of cloudwatch sample"
  style="display: block; margin: 0 auto; border: 1px solid #eee"
/>

- CloudWatch で EC2 インスタンスを監視する
  - ここでは CPUUtilization の監視
- 指定した閾値を超えた場合、Alarm が発生する
- Alarm の action として ScalingPolicy が実行される
  - scale in 用
  - scale out 用
- ScalingPolicy に基づいて AutoScalingGroup 内の EC2 インスタンスが増減される

よくある形として EC2 インスタンスを Web サーバとしてその前に ELB (Elastic Load Balancing) を置くものがありますが、ここでは素の EC2 インスタンスを起動させるだけとしています。

<div id="cfn-intro" />

### 2. CloudFormation 概要

[AWS CloudFormation][4] は AWS リソースをコードとして管理する (Infrastructre as Code) ためのサービスです。
詳細はリファレンスがわかりやすいですが、簡単に要点、利点をまとめると以下のようになります。

- AWS のリソースを JSON または YAML ファイルでまとめて管理できる
- 一度テンプレートを作ればさくっとインフラのセットアップができる
- リソース一覧の把握が容易
  - 異なる region でのセットアップ、移行のようなこともやりやすい
- テンプレートファイルのバージョン管理をすればインフラの変更履歴も追いやすい

CloudFormation ではテンプレートと呼ばれるものにリソースの定義を書き連ねていきます。
テンプレートは [リファレンス][5] で示されている通り、いくつかの要素から構成されます。

```yaml
AWSTemplateFormatVersion: ...
Description: ...
Metadata:
  ...
Parameters:
  ...
Mappings:
  ...
Conditions:
  ...
Transform:
  ...
Resources:
  ...
Outputs:
  ...
```

このうち必須なのは Resources のみです。
Resources とその他ここで使用するものに関しての概要は以下の通りです。

- [AWSTemplateVersion][6]
  - テンプレートフォーマットのバージョンを定義する
  - 現時点では `2010-09-09` が唯一で最新
- [Resources][3]
  - 各種 AWS のリソースやサービスを定義する
  - リファレンスが丁寧なのでそれを見ながらがんばる
  - はじめは web console から作成し、それを参考にしてテンプレートにするのが良いかも
- [Parameters][2]
  - Resources 定義中で共通に使用可能な値を宣言する
  - スタック作成、更新時に動的に設定できる
  - default 値を設定できる
    - 単純に複数のリソースで使用したい値を Parameter にするのも良さそう
- [Mappings][1]
  - Parameters 同様 Resources の定義中に参照できる値
  - 例えば region に応じて違う値が必要になるといった場合に便利

テンプレートから作成された一連のリソースはスタックと呼ばれ、aws-cli からは以下のように作成できます。

```
$ aws cloudformation create-stack --stack-name CloudWatchSampleStack --template-body file://cfn-template.yml
```

スタック更新時は `create-stack` のかわりに `update-stack` を使用します。

```
$ aws cloudformation update-stack --stack-name <stack_name> --template-body <template_file>
```

<div id="cfn-template" />

### 3. CloudFormation テンプレート

今回は AutoScaling のサンプルということでまずは EC2 インスタンスを動かすために必要なインフラまわりを定義します。

テンプレート全体をここに載せると長くなってしまうので省略しますが、作成するリソースの一覧は以下の通りです (全体は [こちら][7] を参照)。

- VPC
- Subnet x 2 (SampleCloudWatchPublicSubnet1a, SampleCloudWatchPublicSubnet1c)
- Internet Gateway
- Route Table
- Security Group (InstanceSecrityGroup)

またテンプレート内では Parameters と Mappings として `ImageId`, `KeyName`, `TargetAZ` というものを定義しています。
それぞれ AutoScaling 設定時に EC2 インスタンスイメージ、鍵ペアの名前、availability zone としてのちほど使用します。

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Parameters:
    ImageId:
        # String, Number, etc.
        Type: "String"
        Default: "ami-923d12f5"
    KeyName:
        Type: "String"
        Default: "aws-sample-key"
Mappings:
    TargetAZ:
        "ap-northeast-1":
            default: ["ap-northeast-1a", "ap-northeast-1c"]
Resources:
    ...
```

<div id="def-autoscaling" />

### 4. AutoScaling の定義

AutoScaling を定義するために以下の 3 種類のリソースを定義します。

- LaunchConfiguration
- AutoScalingGroup
- ScalingPolicy

#### LaunchConfiguration

[LaunchConfiguration (起動設定)][8] は AutoScaling 時に起動される EC2 インスタンスに関する設定です。
例えばどのイメージを使用するか、どのインスタンスタイプを指定するかといったことを定義します。

テンプレートの詳細は [リファレンス][9] を参照です。

```yaml
# --- Laungh Configurations ---
LaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
        ImageId: !Ref ImageId
        InstanceType: "t2.micro"
        KeyName: !Ref KeyName
        SecurityGroups:
            - !Ref InstanceSecurityGroup
        AssociatePublicIpAddress: "true"
        InstanceMonitoring: "false"
        BlockDeviceMappings:
            - DeviceName: "/dev/xvda"
              Ebs:
                  VolumeSize: "8"
                  VolumeType: "gp2"
                  DeleteOnTermination: "true"
```

ここでは Parameters で宣言した KeyName を定義に使用しています。 Parameters の値を使用するためには `!Ref KeyName` というような表記で対象 parameter を指定します。

#### AutoScalingGroup

[AutoScalingGroup][10] は AutoScaling の対象として扱われる EC2 インスタンスのグループです。

AutoScalingGroup は EC2 インスタンスのヘルスチェックを行い、異常なインスタンスを排除、正常なインスタンスを追加して一定のインスタンス数を保つように働きます。
今回は更に ScalingPolicy と連携させ、条件に合わせてインスタンスの増減を行わせますが、どちらにせよ AutoScalingGroup 側で意識することは変わらないと思います。

テンプレートの詳細は [リファレンス][11] を参照です。

```yaml
# --- Auto Scaling Groups ---
AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
        AvailabilityZones: !FindInMap [TargetAZ, !Ref "AWS::Region", "default"]
        VPCZoneIdentifier:
            - !Ref SampleCloudWatchPublicSubnet1a
            - !Ref SampleCloudWatchPublicSubnet1c
        LaunchConfigurationName: !Ref LaunchConfiguration
        DesiredCapacity: "1"
        MinSize: "1"
        MaxSize: "2"
        HealthCheckType: "EC2"
        HealthCheckGracePeriod: 300
        Cooldown: 300
        # LoadBalancerNames:
        #     - !Ref ElasticLoadBalancer
        # MetricsCollection:
        #     - Granularity: ""
        #       Metrics:
        #           - ""
```

Mappings で定義した値を指定するためには `!FindInMap [...]` という表記を使用します。

#### ScalingPolicy

[ScalingPolicy][12] では EC2 インスタンス増減の挙動を定義します。
例えば以下のテンプレート定義は 「policy 実行時に EC2 インスタンスを 1 つ増やす。この policy を実行後、次の policy がトリガーされるまでは最低 60 秒必要となる」 という policy になります。

のちの設定でこの policy を CloudWatch Alarm からトリガーしてもらうようにします。

テンプレートの詳細は [リファレンス][13] を参照です。

```yaml
# --- Scaling Policy ---
ScaleOutPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
        AdjustmentType: "ChangeInCapacity"
        PolicyType: "SimpleScaling"
        Cooldown: "60"
        AutoScalingGroupName: !Ref AutoScalingGroup
        ScalingAdjustment: 1
```

最後に CloudWatch Alarm を定義して自動で EC2 インスタンスの増減が行われるようにします。

#### Alarm

[CloudWatch Alarm][14] で EC2 インスタンスのメトリクスの監視を行い、条件に応じて Alarm を発生させます。

ここでは 「CPU 使用率が 平均 50 % 以上である状態が 300 秒 1 回続いた場合に、Alarm」という設定にしています。
また Alarm 時のアクションとして `!Ref ScaleOutPolicy` を指定し上で定義した policy を実行させます。

テンプレートの詳細は [リファレンス][15] を参照です。

```yaml
# --- Cloud Watch ---
CPUAlarmHigh:
    Type: AWS::CloudWatch::Alarm
    Properties:
        AlarmDescription: "Alarm if CPU too high or metric disappears indicating instance is down"
        Namespace: "AWS/EC2"
        MetricName: CPUUtilization
        Statistic: Average
        ComparisonOperator: GreaterThanThreshold
        Threshold: "50"
        Period: "300"
        EvaluationPeriods: "1"
        AlarmActions:
            - !Ref ScaleOutPolicy
        Dimensions:
            - Name: "AutoScalingGroupName"
              Value: !Ref AutoScalingGroup
```

これらを定義した CloudFormation テンプレートからスタックを作成すると一連のリソースが生成されます。適当な EC2 インスタンスに入り、`yes >> /dev/null` なんかを実行すると CPU 使用率が上昇し scale out が行われることが確認できるはずです。
`yes` を止めれば しばらくのちに scale in することも確認できます。

[1]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/mappings-section-structure.html
[2]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/parameters-section-structure.html
[3]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-reference.html
[4]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/Welcome.html
[5]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/template-anatomy.html
[6]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/format-version-structure.html
[7]: https://github.com/tiqwab/example/tree/master/sample-cloudwatch
[8]: http://docs.aws.amazon.com/ja_jp/autoscaling/latest/userguide/LaunchConfiguration.html
[9]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-launchconfig.html
[10]: http://docs.aws.amazon.com/ja_jp/autoscaling/latest/userguide/AutoScalingGroup.html
[11]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-group.html
[12]: http://docs.aws.amazon.com/ja_jp/autoscaling/latest/userguide/policy_creating.html
[13]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-as-policy.html
[14]: http://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html
[15]: http://docs.aws.amazon.com/ja_jp/AWSCloudFormation/latest/UserGuide/aws-properties-cw-alarm.html
