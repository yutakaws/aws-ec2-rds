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
4.RDS、Secrets Manager作成  
5.Security Groups設定、ELB(ALB)展開  
6.EC2作成＆Dockerコンテナ展開  
7.AMI、Launch Template、EC2 Auto Scaling  
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

### インターネットゲートウェイを作成しVPCへアタッチする  

- Internet gateways画面にて名前を設定し作成する
  
![igw](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/4fb8a2c8-c09a-4c2b-9729-b9c049510328)

- 作成したインターネットゲートウェイを選択し、Actions>Attach to VPCを選択しVPCへ紐づけする  
  
![igw2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/3ad7beb9-0b2f-414f-b056-2c01f2a44365)  
  
### NATゲートウェイをPublic Subnetに作成する  

- NAT gateways画面にて、任意の名前を入力し、Public Subnetを選択し、Allocate Elastic IP(EIP)を作成しNATゲートウェイへ割り当てる

![NAT](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/207f6231-ce84-4761-b97e-48716f1cb2c8)  

### ルートテーブルを作成する

- 下記を参照に各ルートテーブルを作成する

yutakaws-public-subnet(a,c,d)
| 送信先(Destination) | ターゲット(Target) | 目的 |
| --- | --- | --- |
|0.0.0.0/0|igw|インターネットと通信|
|10.0.0.0/19|local|VPC内でのローカル通信|
|Prefix list ID|S3エンドポイント|S3との通信|  

yutakaws-protected-subnet(a,c,d)
| 送信先(Destination) | ターゲット(Target) | 目的 |
| --- | --- | --- |
|0.0.0.0/0|nat|インターネットと通信(中から外のみ)|
|10.0.0.0/19|local|VPC内でのローカル通信|
|Prefix list ID|S3エンドポイント|S3との通信|  
  
yutakaws-private-subnet(a,c,d)  
| 送信先(Destination) | ターゲット(Target) | 目的 |
| --- | --- | --- |
|10.0.0.0/19|local|VPC内でのローカル通信|

- Route tables画面にて任意の名前を入力し、作成したVPCを選択する  
  
![route table1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/f4cfd07b-fed7-4269-9187-1ae8f4a2dfe9)  

- 作成したRoute tableを開き、Routeタブ[Edit Routes]にて各サブネットのルートを設定する  

![route table2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/de66c8de-6723-47fe-9a15-24ccf9869d10)

- EndpointsにてS3 endpoint gatewayタイプを作成しサブネットへ割り当てる

![s3ep](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/ce43fc68-40bf-4e4c-b879-77a6fe294354)

## RDSを構築及び、Secrets ManagerにてSecretsを作成する  
### RDS Aurora MySQLを作成する  
- RDS Databasesにて[Create database]を選択し、Aurora MySQL 5.7.mysql_aurora.2.11.3を選択する
  
![RDS1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/3291fe69-451a-4220-a245-42111093e159)

![RDS2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/1eda8976-75f1-4023-8567-441cb889f6a7)

- 任意の名前を選択し、Masterユーザ、パスワードを設定する

![RDS8](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/c87acb60-0f43-4e79-b43f-ee5bce9158bc)  

-  DBインスタンスクラスをdb.t3.smallに設定する

![RDS4](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/364ff0b6-37aa-4510-83e0-7a2ff53dfa42)  

- 今回はマルチAZ設定をOFFにする
  
![RDS5](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/87c83520-0e35-4b13-8128-b539594b8acd)

- 作成したVPCへアタッチする

![RDS6](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/9e42baf9-8c8a-49b8-8924-5a5ac94432e1)

- ローカルのみアクセス許可をするためPublic accessをNOに設定する
  
![RDS7](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/6baba49e-baed-4564-8362-942967e03e3c)

### AWS Secrets ManagerにてDBアクセス用のSecretを保管する

-  AWS Secrets Manager画面にて[Store a new secret]を選択しSecret typeに[Credential for Amazon RDS]を選択し、任意のユーザーとパスワードを設定する
  
![secrets1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/415f6063-13e9-4cd4-8375-e8f20fd22622)  

-  Databaseに作成したDB instanceを選択する

![secrets2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/2664b6f0-2a0d-4abd-b672-838457b088a0)  

- Secret名と説明を入力

![secrets3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/d73ed1d1-a7a1-49d5-b4cb-26f30e229c47)
  
## Security Groups設定、ELB(ALB)を展開する  
### Security Groupsを設定する  

- 下記を参照しインスタンスへアクセスできるInboundの通信を許可するルールを追加する。Outboundは全てデフォルトのまま  

- インターネット→ALB

| プロトコル(Port) | アクセス元(Source) |
| --- | --- |
|HTTP(TCP80)|0.0.0.0/0|
|HTTPS(TCP443)|0.0.0.0/0|
  
- ALBのSGからEC2

| プロトコル(Port) | アクセス元(Source) |
| --- | --- |
|HTTP(TCP80)|ALBのSG|
  
- EC2のSGからRDS

| プロトコル(Port) | アクセス元(Source) |
| --- | --- |
|MYSQL/Aurora(TCP3306)|EC2のSG|
  
- Security Groups画面にて[Create security group]を選択し、上記を参照し値を入力する  
※Security group nameはどのリソースへ取り付けたか、わかりやすくするため正確に付ける  
  
![sgalb](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/8fc65b39-ddff-4e14-b9a6-47c69e237762)  
  
![sgalb2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/19647469-3875-4b8a-bccc-1dd5b5a1f4f4)

### ELB(ALB)を展開する  
- Load Balancers画面にて[Create load balancer]を選択し、任意の名前を入力、Internet-facing、IPv4を選択する

![ALB1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/1e33a9d7-8155-42ca-9adf-852867524b12)

- 作成したVPCを選択肢、さらに3つのAZのPublic subnetを選択する

![ALB2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/411e5a06-e9a2-4967-ab62-61f935ccf0df)  

- 作成したSecurity Groupを選択する

![ALB3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/6550895e-f3ee-4d2e-963f-9c97d865b94f)

### HTTPSへリダイレクトを行うためにListenersにてListener rulesを設定し、Target groupも併せて作成

- HTTP Port:80にRedirectを選択し、443ポートを指定する
  
![ALBlistener](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/606a3c1a-e93e-4404-b76e-ac50014ee8ac)  
  
![ALBlistener3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/1644a72c-0411-42a6-8d28-b0813a056f66)  
  
- Target Groups画面にて[Create target group]を選択し、対象としてInstanceを選択する

![tg](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/bfac0910-3e56-4d7f-9f9c-62719a856a8f)

- 任意の名前を入力し、VPCを選択する
  
![tg2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/8d041dde-1111-4d88-a7b4-d50a244cadb2)
  
### HTTPSへ特定のリクエストのみ転送するListnener ruleを設定する　　

- 403を表示させるため、Response code,Content type,Response bodyの設定を行う
  
![ALBlistenerhttps](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/a0a276bc-9f24-4177-b807-9a4819bcd5d5)

- セキュリティポリシー及びSSL証明書をACMに設定する

![ALBlistenerhttps3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/3eb26db9-8dc2-4521-9bdf-1cee66bf9722)

- HTTPS:443 Listenerを選択し、Rulesタブ[Manage rules]にて更に細かく設定する

![ALBlistenerhttps4](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/1483c208-3c90-455b-beec-1d66b949bd4b)


## EC2インスタンスを作成 

### IAM roleにてEC2インスタンスへ取り付ける「Session Managerを許可する」「Secretsの値を参照を許可する」Roleを作成する  

- IAM Roles画面にてcreate roleを選択しAWS service,EC2を選択する  

![iam](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/533ff952-ce3c-4df1-b184-3ce55e803482)  

- [Create policy]を選択し、Secrets Managerを検索し、Access levelのRead,GetSecretValueにチェック
  
![policy2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/4b600b1e-9d19-48ed-899b-353ea8cf1477)

- 任意のPolicyネームを付ける
  
![policy3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/b89a3ba5-0870-4baf-9779-2e1a7a0d4209)

- Session Managerを許可する[AmazonSSMManagedInstanceCore]と、先程作成したPoliciyを選択する  
  
![amrrole;](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/b0f51782-3f3c-4a72-8e6a-c4a6b1a6d9fb)  
![amrrole2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/4e94fa60-aea5-4353-9b6f-40bd9b23e1d0)  
  
- 任意のRoleネームを付ける

![amrrole3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/a1cbc316-b772-4149-9027-8e78831e5919)

### EC2インスタンスを作成する  

- Instances画面にて[Launch instances]を選択し、任意の名前を入力しAMIを選択する（今回はAmazon Linux2のAMIを使用)

![ec1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/8738a255-ce5b-47f2-8d7c-0a2be6dc17fe)
![ec2](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/d3e31276-d18f-420c-b2ad-3255603b04a6)

- Instance typeを選択(今回はt2.micro)

![ec3](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/33c235ba-c43c-4546-b9f5-596b40ff1738)

- Session Managerにて接続をするため、Key pairなしを選択
  
![ec4](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/89f639d9-ad72-41b5-926f-e6e10d0e8968)  
  
- 作成したVPC,Protected Subnet,Security groupを選択  
  
![ec5](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/d1f1e5ee-3b4a-473c-a966-c98f1801be08)

- EBSをgp3へ変更する  
  
![ec6 5](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/592b2987-c115-49c0-8381-f35740e0ab19)
  
- Advanced detailsを開き、IAM instance profileにて作成したIAM roleを選択する

![ec6](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/b4d9efd0-9b40-46ff-a1a9-22adde594246)


- インスタンス起動時にDockerおよびDocker Composeをインストールするため下記をUser dataを入力する  
```
#!/bin/bash
yum -y update
aws configure set default.region ${AWS::Region}
## Install Docker Engine
amazon-linux-extras install docker -y
systemctl enable docker.service
systemctl start docker.service
## Install Docker Compose
CLI_DIR=/usr/local/lib/docker/cli-plugins
LATEST_RELEASE=$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep browser_download_url | grep -i $(uname -s)-$(uname -m) | grep -v sha256 | cut -d : -f 2,3 | tr -d \")
mkdir -p ${!CLI_DIR}
curl -sL ${!LATEST_RELEASE} -o ${!CLI_DIR}/docker-compose
chmod +x ${!CLI_DIR}/docker-compose
ln -s ${!CLI_DIR}/docker-compose /usr/bin/docker-compose
## Run Docker Container
docker container run --name nginx --restart=always -d -p 80:80 nginx
```

- EC2インスタンスを作成したら、ELBのTarget Groupにて[Register targets]を選択し、EC2インスタンスを追加する
  
  ![ec8](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/ef75ecde-66f2-418f-b4df-68a6b8f099b3)
  
## DockerコンテナにNginxとRubyのコンテナを作成する  
### Dockerビルド前の準備を行う  

- EC2Instance画面にて作成したEC2インスタンスにチェックを入れ[Connect]を選択し、Session Managerで接続する
  
![docker1](https://github.com/yutakaws/aws-ec2-rds/assets/138670733/d5e6eb6d-1f35-4d15-b160-30f4ff27d111)  

- 作業ディレクトリに移動する
```
###　任意の名前=***
bash
mkdir -p ~/*** && cd ~/***
```
- dockerコマンドをssm-userで使えるようにする
```
sudo usermod -a -G docker ssm-user
```
- グループの変更がすぐに反映されないので下記を実行する
```
exec sudo -i -u $(whoami)
cd ~/***
```
- ディレクトリの作成
```
mkdir -p app
mkdir -p containers/app
mkdir -p containers/web
```
- docker-compose.ymlを作成する
