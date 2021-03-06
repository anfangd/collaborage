AWS
================================
ここではAWSにチーム開発環境を構築します。
はじめに「アーキテクチャ」に軽く目を通して作成内容を確認してから、「事前準備」「インストール」に進みます。

- [アーキテクチャ](#アーキテクチャ)
- [事前準備](#事前準備)
- [インストール](#インストール)
- [オペレーション](#オペレーション)


## アーキテクチャ


![アーキテクチャ](images/aws-architecture.png)


### アプリの配置


- 3つのインスタンスにアプリを配置します。
  - CQ(Communication/Quality)サーバ
  - CI(Continuous Integration)サーバ
  - Demoサーバ
- CQ/CIサーバのEC2はm4.large、ルートボリューム20GB、データボリューム40GBでデフォルト提供します。
- Demoサーバはt2.small、ルートボリューム20GBでデフォルト提供します。
- Concourse/GitLabを使う場合は、Concourse/GitLabがリソースを消費するため、CIサーバのEC2のみ、m4.xlarge、ルートボリューム40GB、データボリューム40GBでデフォルト提供します。
- **上記のマシンスペックは推奨する最低限のスペックになります。PJの規模に合わせて変更してください。**


### HTTPSによる通信


- 開発拠点からインターネットを経由してAWSにアクセスするため、ALBを設けてHTTPSを使用します。
- HTTPSに必要なドメイン、SSL証明書は事前に準備します。


### バックアップ


- CQ/CIサーバのアプリデータのみバックアップを取得します。（CIにより復元できるためDemoサーバのアプリデータは取得しません）
- EBSのスナップショットをバックアップとします。
- CQ/CIサーバの/dataをEBSにマウントし、アプリのデータを/data/<アプリ名\>に格納します。
- CQ/CIサーバのcronでシェルスクリプトを定期実行し、EBSのスナップショットを取得します。
- デフォルトは1日1回（JST 23時0分）取得します。
- シェルスクリプトの処理は「アプリ停止」＞「スナップショット取得」＞「古いスナップショット削除」＞「アプリ開始」の順に処理します。
  - アプリ停止
    - 「docker-compose stop」でアプリを停止します。
  - スナップショット取得
    - AWS CLIを使ってスナップショットを取得します。
    - **スナップショットの取得対象は、Nameタグに「nop-ebs-data-(cq|ci)」が設定されているEBSとなります。**
    - 作成したスナップショットには、以下のタグを設定します。
      - Nameタグ: nop-ebs-data-(cq|ci)-snapshot
      - NopPurgeAfterタグ: スナップショットが有効な日付（この日付以降に削除されます）
  - 古いスナップショット削除
    - AWS CLIを使ってスナップショットを削除します。
    - スナップショットの取得対象は、Nameタグに「nop-ebs-data-(cq|ci)-snapshot」が設定されているEBSとなります。
    - NopPurgeAfterタグの日付＜現在の日付となった場合に対象のスナップショットを削除します。
  - どこかの処理で失敗すると、AWS CLIを使ってSNSのトピックにメッセージを発行します。
    - SNSのトピックにサブスクリプションとしてemailを設定しておくことで、メール通知を実現します。
- デフォルトで7日前のスナップショットまで残します。


### 監視


- CQ/CIサーバのディスク使用率のみ監視します。（Demoサーバは監視しません）
- CQ/CIサーバのcronでシェルスクリプトを定期実行し、メトリクス(ディスク使用率)をCloudWatchに送信します。
- デフォルトは5分間隔で送信します。
- シェルスクリプトでは、[Amazon CloudWatch モニタリングスクリプト](http://aws.amazon.com/code/8720044071969977)を使って、メトリクスをCloudWatchに送信します。
  - メトリクス取得の詳細は、[Amazon EC2 Linux インスタンスのメモリとディスクのメトリクスのモニタリング](http://docs.aws.amazon.com/ja_jp/AWSEC2/latest/UserGuide/mon-scripts.html)を参照してください。
  - どこかの処理で失敗すると、AWS CLIを使ってSNSのトピックにメッセージを発行します。
    - SNSのトピックにサブスクリプションとしてemailを設定しておくことで、メール通知を実現します。
- CloudWatchでアラームを手動で設定します。
  - ディスク使用率が80%を超えたらメール送信（SNS経由）するように設定します。


## 事前準備


### AWS

- AWSマネジメントコンソールにアクセスし、図にあるリソースを作成します。
  - ![事前準備](images/aws-prepare-resources.png)
- 既にある環境にインストールする場合は足りないリソースのみ作成します。
- ネットワーク構成は[パブリックサブネットとプライベートサブネットを持つ VPC (NAT)](http://docs.aws.amazon.com/ja_jp/AmazonVPC/latest/UserGuide/VPC_Scenario2.html)を参照してください。
- 各リソースの作成方法はAWSのドキュメントを参照するか、AWS有識者に相談してください。30分～1時間で作成できると思います。
- トピックにはemailのサブスクリプションも作成します。
- ポリシーの作成は以下のjsonを使用してください。
  ```
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "Stmt1504604285000",
        "Effect": "Allow",
        "Action": [
          "cloudwatch:PutMetricData",
          "ec2:CreateSnapshot",
          "ec2:CreateTags",
          "ec2:DeleteSnapshot",
          "ec2:DescribeTags",
          "ec2:DescribeVolumes",
          "sns:Publish"
        ],
        "Resource": [
          "*"
        ]
      }
    ]
  }
  ```
- インストールにはCollaborageが提供するAMIを使用します。AMIはパブリックイメージとして公開しています。
  - CQサーバ： nop-dev-cq-0.1.6
  - CIサーバ(GitBucket/Jenkins)： nop-dev-ci-jenkins-0.1.6
  - CIサーバ(GitLab/Concourse)： nop-dev-ci-concourse-0.1.6
  - Demoサーバ： nop-inst-demo-0.1.4

### 作業PC

- gitコマンド/sshコマンドを使えるようにします。
  - Windowsであれば[Git for Windows](https://git-for-windows.github.io/)をインストールすれば、Git BASHを使用してgitコマンド/sshコマンドを使えます。
  - インストール方法は[このあたり](https://www.google.co.jp/search?q=git+for+windows+%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB&oq=Git+for+Windows%E3%80%80&gs_l=psy-ab.1.0.0l8.3848.3848.0.5856.1.1.0.0.0.0.139.139.0j1.1.0....0...1.4.64.psy-ab..0.1.138.7Ht2O-5L3YM)を参考にしてください。
- sshコマンドで踏み台サーバにアクセスします。
  - アクセス方法は[このあたり](https://www.google.co.jp/search?q=aws+ec2+ssh+%E8%B8%8F%E3%81%BF%E5%8F%B0&oq=aws+ec2+ssh&gs_l=psy-ab.1.2.0l8.19285.21173.0.23904.7.7.0.0.0.0.140.668.0j5.5.0....0...1.1.64.psy-ab..2.5.667...0i131k1.HfnC6xbuzC4)を参考にしてください。


## インストール


インストールは次の手順で行います。

- [作業場所を準備します](#作業場所を準備します)
- [CloudFormationでAWSリソース（EC2など）を作成します](#cloudformationでawsリソースec2などを作成します)
- [SSHを準備します](#sshを準備します)
- [Route53でサブドメインを追加して、各アプリにアクセスできるようにします](#route53でサブドメインを追加して各アプリにアクセスできるようにします)
- [AMIから作成したEC2インスタンスの初期設定を行います](#amiから作成したec2インスタンスの初期設定を行います)



### 作業場所を準備します


![](images/aws-workplace.png)

- Collaborageをクローンします。
  ```
  $ git clone https://github.com/Fintan-contents/collaborage.git
  $ cd collaborage
  ```
- 作業場所を作成します。作業場所のディレクトリ、CIツール(GitBucket/Jenkins or GitLab/Concourse)を決めてください。特に希望がなければ、情報が多く、リソース消費が少ない(つまり低コスト)、Jenkinsを使ってください。
  - ユーザのnopディレクトリにJenkinsで作る場合
    ```
    $ ./init-workplace.sh ~/nop aws jenkins
    $ cd ~/nop
    ```
  - ユーザのnopディレクトリにConcourseで作る場合
    ```
    $ ./init-workplace.sh ~/nop aws concourse
    $ cd ~/nop
    ```


### CloudFormationでAWSリソース（EC2など）を作成します

![](images/aws-cf-image.png)

- AWSマネジメントコンソールにアクセスします。
- S3にアクセスし、テンプレートファイルをアップロードします。バケットが無ければ、新しくバケットを作成します。
  - Jenkinsを使う場合
    ```
    nop/template/nop-with-ssl-jenkins.yaml
    ```
  - Concourseを使う場合
    ```
    nop/template/nop-with-ssl-concourse.yaml
    ```
  - アップロードの後、テンプレートファイルを選択し、リンクをコピーします。
    - ![S3のファイルリンク](images/aws-s3-link.png)
- CloudFormationにアクセスし、スタックを作成します。
  - 「新しいスタックの作成」を選択します。
  - アップロードしたテンプレートファイルを選択します。
    - ![CFのテンプレート](images/aws-cf-template.png)
  - パラメータを設定します。リソース関連は事前準備で作成したものを指定します。
    - スタックの名前
      - 「nop」と入力します。
    - BastionSecurityGroup
      - 踏み台サーバのセキュリティグループを選びます。
    - Ec2KeyPair
      - EC2に割り当てるキーペアを選びます。
    - Ec2Role
      - EC2に割り当てるロールの名前を指定します。
      - IAMにアクセスし、ロールの名前を確認します。
        - ![IAMのロール](images/aws-iam-role.png)
    - Ec2TypeForCi
      - CIサーバのインスタンスタイプを指定します。
      - Jenkinsを使う場合は「m4.large」、Concourseを使う場合は「m4.xlarge」ぐらいあれば大丈夫だと思います。
    - Ec2TypeForCq
      - Communication/Qualityサーバのインスタンスタイプを指定します。
      - 「m4.large」ぐらいあれば大丈夫だと思います。
    - Ec2TypeForDemo
      - Demoサーバのインスタンスタイプを指定します。
      - PJの要件(APサーバやDBサーバ等)に合わせて指定してください。
    - PrivateSubnet
      - プライベートサブネットを選びます。
    - PublicSubnets
      - パブリックサブネットを選びます。
      - **ALBを使用するため、パブリックサブネットを2つ選んでください。**
    - SourceIp
      - 開発環境へのアクセス元となるソースIPのCIDRを指定します。
    - SslCertificate
      - SSL証明書のARNを指定します。
      - Certificate Managerにアクセスし、証明書を選択し、証明書のARNを確認します。
        - ![Certificate Managerの証明書ARN](images/aws-cm-sslarn.png)
    - Vpc
      - VPCを選択します。
  - 以降は「次へ」を選択し、作成します。大体3分ぐらいで完了します。
    - 状況がPROGRESSからCOMPLETEになれば作成完了です。右上の更新ボタンで表示を更新できます。
      - PROGRESS
        - ![CFの作業中](images/aws-cf-progress.png)
      - COMPLETE
        - ![CFの作業完了](images/aws-cf-complete.png)
    - EC2インスタンスはすぐに作成中の状態になり、プライベートIPが割り当てられるので、待っている間に次の手順にあるSSHの準備をします。
      - EC2にアクセスし、各EC2インスタンスを選択して、プライベートIPを確認します。
        - ![EC2のIP](images/aws-ec2-privateip.png)


### SSHを準備します


- 多段SSHを使って、踏み台サーバ経由で各EC2インスタンスにアクセスします。
- 多段SSHは[このあたり](https://www.google.co.jp/search?q=%E5%A4%9A%E6%AE%B5ssh&oq=%E5%A4%9A%E6%AE%B5ssh&gs_l=psy-ab.3..0i71k1l4.0.0.0.3362.0.0.0.0.0.0.0.0..0.0....0...1..64.psy-ab..0.0.0.vBzx5nON7hY)を参照ください。
- SSHの接続情報はconfigファイルに定義しておき、configファイルとホスト名を指定するだけでSSH接続できるようにします。
- SSHの接続設定を修正します。
  ```
  nop/.ssh/ssh.config
  ```
  - 踏み台サーバとEC2インスタンスのキーペアファイルを.sshにコピーします。
  - プライベートIP、キーペアファイルへのパス、踏み台サーバのユーザ名を指定します。
- 作業場所で各EC2インスタンスにアクセスできることを確認します。
  ```
  $ ssh -F .ssh/ssh.config nop-bastion
  $ ssh -F .ssh/ssh.config nop-cq
  $ ssh -F .ssh/ssh.config nop-ci
  $ ssh -F .ssh/ssh.config nop-demo
  ```


### Route53でサブドメインを追加して、各アプリにアクセスできるようにします


- Route 53にアクセスし、レコードセットを追加します。
  - ここでは、例として「adc-tis.com」というドメインへ追加しています。
  - 「Hosted zones」を選択します。
    - ![Route53のHostedZones](images/aws-r53-hostedzones.png)
  - 「Domain Name」のリンクを選択します。
    - ![Route53のドメイン](images/aws-r53-domain.png)
  - レコードセットを追加します。
    - ![Route53のレコードセット](images/aws-r53-recordset.png)
    - Name:
      - サブドメインの名前を指定します。
      - ここで指定した名前が各EC2インスタンスへアクセスする際のホストになります。
      - アクセス先となるEC2インスタンスの用途（CQ、CI、Demo）に合わせた名前を指定します。
    - Type:
      - IPv4のままです。
    - Alias:
      - Yesを選択します。
    - Alias Target:
      - EC2インスタンスの用途（CQ、CI、Demo）に合わせたALBを選びます。
      - ALBの名前は「nop-alb-(cq|ci|demo)-xxxxx」で作成しています。
      - フォーカスを合わせて入力欄を空にすると、存在しているALBがプルダウンにロードされます。
    - 以降の項目はデフォルトのままでCreateします。
    - EC2インスタンスごとにレコードセットを追加します。
- ブラウザを開いて各EC2インスタンスのアプリにアクセスできることを確認します。
  ```
  CQサーバ
  https://<CQサーバのホスト>/redmine

  CIサーバ
  https://<CIサーバのホスト>/nexus
  
  Demoサーバ
  アプリをデプロイしていないので、この時点ではアクセスできません。
  ```


### AMIから作成したEC2インスタンスの初期設定を行います


- [こちら](ami.md)を参照して作業します。

これでインストールは終わりです！
お疲れさまでした。

プロジェクト向けの開発準備は[プロジェクトの開発準備](dev.md)を参照して作業してください。


## オペレーション


- [アプリの操作方法（起動/停止/ログ確認）](#アプリの操作方法起動/停止/ログ確認)
- [アプリの復元方法（アプリ/アプリデータ）](#アプリの復元方法アプリ/アプリデータ)
- [アプリデータの復元方法](#アプリデータの復元方法)
- [バックアップとメトリクス取得が失敗した場合のログの確認方法](#バックアップとメトリクス取得が失敗した場合のログの確認方法)


### アプリの操作方法（起動/停止/ログ確認）


CQサーバ/CIサーバのアプリはdocker-composeを使って起動しています。
アプリの操作はSSHでアクセスして、docker-composeのコマンドで行います。

docker-composeのコマンドは[コマンドリファレンス](http://docs.docker.jp/compose/reference/toc.html)を見てください。

CQサーバ/CIサーバのアプリを操作する場所は次の通りです。

- CQサーバ
  ```
  /home/centos/nop/docker/cq
  ```
- CIサーバ
  ```
  /home/centos/nop/docker/ci
  ```

#### アプリの起動

```
$ docker-compose start
```

#### アプリの停止

```
$ docker-compose stop
```

#### アプリの一覧表示

```
$ docker-compose ps
```
- Name列にサービス名が表示されます。

#### アプリのログ確認

```
$ docker-compose logs <サービス名>
```
- 「docker-compose ps」コマンドでサービス名を確認します。


### アプリの復元方法(アプリ/アプリデータ)


#### アプリの復元

アプリの復元は、インストール時に作成しておいたAMIを使って行います。

- CQサーバ/CIサーバの場合は、アプリデータのバックアップを取得します。
  - SSHでサーバにアクセスします。
  - 次のコマンドでデータボリュームのスナップショットを取得します。cronで実行しているシェルスクリプトを直接実行します。
    ```
    $ cd nop/script/cron/
    $ ./create-snapshot-of-ebs-data-volume.sh
    ```
  - AWSマネジメントコンソールで「EC2」＞「スナップショット」にアクセスし、スナップショットが作成されていることを確認します。
  - SSHを切断します。
    ```
    $ exit
    ```
- 不要となったEC2インスタンスを削除します
  - AWSマネジメントコンソールで「EC2」にアクセスし、インスタンスを選択＞「アクション」＞「インスタンスの状態」＞「削除」を選択します。
- AMIからEC2インスタンスを復元します。
  - EC2インスタンスのプライベートIPを確認します。
    - 作業PCの作業場所にあるSSHの接続情報(.sshディレクトリのconfigファイル)を見てプライベートIPを確認します。
  - 「EC2」＞「AMI」にアクセスし、AMIを選択＞「作成」を選択します。
    - ステップ 2: インスタンスタイプの選択
      - [アーキテクチャ](#アーキテクチャ)を参考にして選択します。
    - ステップ 3: インスタンスの詳細の設定
      - ネットワーク、サブネット、IAMロールを指定します。
      - ネットワークインタフェースの「プライマリIP」に先ほど確認したプライベートIPを指定します。
    - ステップ 4: ストレージの追加
      - 特に変更することはありません。
    - ステップ 5: タグの追加
      - 「クリックして Name タグを追加します 」を選択します。
        - 値に「nop-ec2-xx」を指定します。xxにはサーバの用途に合わせて「cq」「ci」「demo」のいずれかを指定します。
    - ステップ 6: セキュリティグループの設定
      - 「既存のセキュリティグループを選択する」を選択します。
        - 名前「nop-sg-ec2」を選択します。
    - 作成します。
- [アプリデータの復元](#アプリデータの復元方法)を行います。
- ALBから復元したEC2インスタンスにアクセスできるようにターゲットグループの設定を変更します。
  - AWSマネジメントコンソールで「EC2」＞「ターゲットグループ」にアクセスします。
  - サーバの用途に合わせて「nop-tg-cq」「nop-tg-ci」「nop-tg-demo」のいずれかを選択します。
  - 「ターゲット」タブ＞「編集」を選択します。
  - インスタンス選択＞「登録済みに追加」を選択し、保存します。
  - ブラウザからアプリにアクセスできることを確認します。
- CQサーバ/CIサーバの場合は、cron関連の設定作業を行います。
  - [バックアップ対象の目印となる名前をデータボリュームに設定します](ami.md#バックアップ対象の目印となる名前をデータボリュームに設定します)を参照して作業します。
  - SSHでサーバにアクセスします。
  - cronを削除します。
    ```
    $ crontab -r
    ```
  - AWS CLIのキャッシュ(インスタンスID)を削除します。
    ```
    $ sudo rm /var/tmp/aws-mon/instance-id
    ```
  - cronを設定します。
    ```
    $ cd ~/nop/script/
    $ ./set-cron-after-try-command.sh
    ```
    - AWSマネジメントコンソールで「EC2」＞「スナップショット」にアクセスし、スナップショットが作成されていることを確認します。
    - AWSマネジメントコンソールで「CloudWatch」＞アラームにアクセスし、ディスク使用率のアラーム設定を変更します。
      - メトリクスを選択しなおします。


### アプリデータの復元方法


アプリデータの復元は、EBSのスナップショットを使って行います。

以下の手順で行います。
- スナップショットからボリュームを作成します。
- EC2インスタンスを停止します。
- EC2インスタンスの「/dev/sdb」デバイスにアタッチされているボリュームをデタッチします。
- EC2インスタンスの「/dev/sdb」デバイスにスナップショットから作成したボリュームをアタッチします。
- デタッチしたボリュームを削除します。



### バックアップとメトリクス取得が失敗した場合のログの確認方法


cronで実行しているバックアップとメトリクス取得が失敗した場合は、メールで通知されます。

SSHでサーバにアクセスし、エラーログの内容を確認してください。

CQサーバ/CIサーバともに、以下の場所にエラーログが出力されます。

```
/home/centos/nop/log/
```
