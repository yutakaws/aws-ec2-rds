# EC2(Docker[Rails+Nginx])+RDSの環境構築

下記の構成で構築を行う

- **3つのAZ内に9つのサブネット**  
本番(prod)、開発(dev)、検証(stg）ごとにCiderBlockを作成。今回はprodのみを使用

- **Public Subnet**  
インターネット公開用、中と外の両方向の通信が可能

- **Protected Subnet**  
EC2インスタンス用、NAT Gatewayによる中から外の通信が可能

- **Private Subnet**  
データベースなどのローカル用、ローカル内のみの通信が可能

- **Route53によるDNS名前解決**  

- **EC2インスタンスへrails、ngixベースのDockerコンテナを展開**  
インスタンスは、Protected Subnetへ設置
NAT gateway及びALBによる通信

- **AZをまたがるELB(ALB)、EC2 Auto Scaling**  
ALBをロードバランサーとしてPublic Subnetに展開
EC2 Auto Scaling　作成したEC2インスタンスのAMIを利用したLaunch Templateを使用し、EC2インスタンスのスケーリングを行う

- **データベースとしてRDSを展開**  
DBクラスターをPrivate Subnetへ展開
EC2からのデータをこちらで管理

<!-- (TOC:collapse=true&collapseText=Click to expand) -->
<details>
<summary>click (詳細な使用リソース一覧はこちら)</summary>

- IAM  
管理者権限を付与したIAMユーザー（MFA有効）

- IAM Role  
リソースへ取付け権限を付与する（Assume Role)  
ManagedPolicyとInlinePolicyを両方使う。

- Route53  
Hosted Zone内のキャッシュDNSリゾルバー及び、権威DNSサーバーにてDNS名前解決を行う。  
ドメイン名はお名前.comから使用。  

- EIP  
固定パブリックIP  
今回はNATゲートウェイへアタッチ。  

- NAT Gateway  
インターネットに出ていく際に、送信元IPアドレスをEIPに変換してくれる装置。  
Public Subnetへ展開し、Protected Subnet内から外への通信を可能にする。  
料金が高いため注意。  
アジアパシフィック（東京）  
NAT ゲートウェイあたりの料金 (USD/時)　0.062USD　処理データ 1 GB あたりの料金 (USD)　0.062USD  

- AWS Certificate Manager (ACM)  
ELBで使用する証明書

- VPC  
IPアドレスの範囲を定義し、インターネットに公開するか・インターネットに接続できるかも制御できる、ネットワークの骨組み。

- RouteTable  
通信経路を示す表  
Public Subnet共通で1つ  
Protected Subnetごとに1つ　
Private Subnet共通で1つ  

- EC2: SecurityGroup  
内向きは許可するものだけルールを追加していく。今回はHTTP（TCP 80番ポート）をフルオープンにする。

- EC2:Instance  
仮想サーバ。AMIのイメージID、インスタンスタイプ、インスタンスプロファイル、サブネットID、セキュリティグループIDなどを指定して作成する。  
料金制は主に従量課金制の「オンデマンド」、1年もしくは3年間の契約を行う「リザーブド」または「Savings Plans」、一時的な利用に向いている「スポット」がある。  
今回はオンデマンドインスタンス　インスタンスタイプはt2.microを使用。（EC2 Auto Scalingのスケーリングにはスポットインスタンスを使用）  
料金  
オンデマンドインスタンス　t2.micro	１時間あたり0.0152 USD  
  
- RDS  
マネージドRDBサービス  
クラウド向けに最適化されたAmazon Aurora MySQLを使用。  
DBインスタンスのうち、1つがPrimaryインスタンス (Writer)、残りがAuroraレプリカ (Reader)  
自動的に3つのAZに6個のデータがCluster Volumeへレプリケーションされる。  
料金  
Aurora Standard db.t3.small	1時間あたり0.063USD

- SecretsManager: Secret  
DBの認証情報「シークレット」の安全な保管が可能。  
DBのマスターユーザー(管理者)と、後で作るRailsアプリ用のDB名・ユーザー名、2つの認証情報を保管する。

- ELB(ALB)  
負荷分散を行う  
HTTP/HTTPSに特化したALB (Application Load Balancer)を使用  
インターネットに公開するため、Public Subnetで作成する。画面上は1つに見えるが、実際は指定した3つのサブネットに作られる。HTTP/2を有効にする。  

- AMI(Amazon Machine Images)  
EC2インスタンスを起動するのに必要なマシンイメージ。

- Launch Template  
インスタンスプロファイル（IAMロール）、サブネット、セキュリティグループ、IPアドレスの自動割当などの設定値をテンプレ化したリソース。

- EC2 Auto Scaling  
EC2インスタンスのスケールアウトやスケールインを自動的に行う。  
CPUやメモリの負荷などをトリガーや曜日と時間をトリガーにした設定を行える。

- Amazon Simple Storage Service（Amazon S3)  
容量無制限のオブジェクトストレージサービス  
RDSのログ保存先として利用。  

- CloudFormation  
AWSのインフラ環境/リソースをコードで管理することができ、それを用いて一括でリソースを起動/削除が可能となる。
  
</details>

##  構築手順  
1.IAM 管理ユーザーを作成  
2.Route53にてDNSの設定、ACMにて証明書の設定    
3.VPCを構築する  
4.SecurityGroup  
5.RDS、Secrets Manager作成  
6.EC2作成＆Dockerコンテナ展開  
7.ELB展開  
8.AMI、Launch Template、EC2 Auto Scaling  
## IAM 管理ユーザーの作成
### User groupsにて管理者権限を付与したユーザーグループを作成  
- グループ名を入力
  
![iam-usergroup-name](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/7240d850-36c5-4f44-b0d9-7702a0a5ca4b)

- Managed policyの[AdministratorAccess](https://docs.aws.amazon.com/ja_jp/IAM/latest/UserGuide/access_policies_job-functions.html#jf_administrator)を選択し作成
  
![iam-usergroup-permission](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/5e52c8e5-1463-4dff-9405-794a8a608322)

### UsersにてUserを作成しUser groupへ追加を行い、MFA認証を有効にする
- Add Usersを選択後、任意の名前を入力し先程作成したUser groupへ追加する  
  
![user add group](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/f16f2ffa-e810-44b7-8bfa-9386db25bd37)

- Security credentialsにてMFAを有効化する  

![MFA](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/649d2df6-53c5-4f9a-93d0-378e2a71d206)
## Route53にてDNSの設定、ACMにて証明書の設定  
### Route 53にて[Hosted zone](https://docs.aws.amazon.com/ja_jp/Route53/latest/DeveloperGuide/hosted-zones-working-with.html)を作成  

- 購入したドメインを入力し、Public hosted zoneを設定
  
![hosted zone](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/4981a816-42db-4a63-9eee-80017bd8af1e)
![hosted zone2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/8896f870-4684-4579-bc48-b82429588c4f)

- タグを設定  
  
![hosted zone3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/289d744a-8cca-454c-bcaa-4f059d3450fd)

### CAAレコードを作成し、ドメイン登録サービスで、Route 53ホストゾーンのネームサーバを変更する  
- Hosted zone画面にてCreate Recordを選択し、CAAを設定し、レコードとしてValueへ'0 issue "amazon.com"'を入力する
  
![CAA1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/3791c3ad-dd5b-4a65-a4cd-469639c0137d)
  
- TTL (Time To Live)を１時間(3600秒)に設定する
  
![CAA2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/733ba058-ee9c-4ba0-8a24-4aea6af60dba)

- ドメイン登録サービスの管理画面にてネームサーバーをHosted zoneにて設定されたネームサーバーへ変更する  

![name server2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/a8e63fd8-0d3f-4d1f-86f2-f53e4c33012e)
![name server](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/41dabb6b-74d2-4664-9a99-bc3a0d841dd6)

### AWS Certificate Manager (ACM)にて証明書をリクエストする  
- Request Certificate画面にてAmazonからのSSL/TLS証明書をリクエストする

![ACM1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/7ecb659f-fe65-437e-9e83-eeb1dd90546c)

- ドメイン名を入力し、Add another name to this certificateを選択し、SANも登録する

![ACM2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/ce028b7f-3a53-42e8-a5c1-043252590694)

### ドメイン認証のためにRoute 53にCNAMEレコードを追加する

- List certificatesにて該当の証明書を選択し、Create records in Route53を選択する

![ACM3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/faec4c2e-3697-4b2b-ae3e-f4010ff90ae0)

- Route 53にてCNAMEレコードが追加されていることを確認

![ACM4](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/e5e9d295-9860-406d-9410-abf0a3faabbd)

## VPCを構築する  
### VPCを作成する

- Your VPCsにて、ネットワーク全体を表すCIDRを10.0.0.0/19と設定し、任意のNameを入力する  

  ![Vpc1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/e490ea39-6d34-4a5b-b01b-ed520d16b384)
  
### サブネットを作成する  
- 3つのAZに3つの目的（Public、Protected、Private）ごとの合計9つのサブネットを作る  
**Public**  インターネット公開用、中と外の両方向の通信が可能  
**Protected**  EC2インスタンス用、NAT Gatewayによる中から外の通信が可能  
**Private**  データベースなどのローカル用、ローカル内のみの通信が可能
  
| Name | AZ | CIDR |
| --- | --- | --- |
| yutakaws-public-subnet-a | ap-northeast-1a | 10.0.0.0/24 |
| yutakaws-protected-subnet-a | ap-northeast-1a | 10.0.4.0/24 |
| yutakaws-private-subnet-a | ap-northeast-1a | 10.0.8.0/24 |
| yutakaws-public-subnet-c | ap-northeast-1c | 10.0.1.0/24 |
| yutakaws-protected-subnet-c | ap-northeast-1c | 10.0.5.0/24 |
| yutakaws-private-subnet-c | ap-northeast-1c | 10.0.9.0/24 |
| yutakaws-public-subnet-d | ap-northeast-1d | 10.0.2.0/24 |
| yutakaws-protected-subnet-d | ap-northeast-1d | 10.0.6.0/24 |
| yutakaws-private-subnet-d | ap-northeast-1d | 10.0.10.0/24 |  
  
- 作成したVPC IDを選択し、上記のサブネットを参考にサブネット名、AZ、CIDR、Name Tagを設定する

![subnet1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/6fde4dfc-7c91-4bf2-84c7-d4c84f610f2f)  
   
![subnet2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/3468fd87-9d6c-43a7-b96a-127d2cef65f8)  

