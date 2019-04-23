# AWS-sidebyside-vpn-by-vyos



## Workspaces構成
(1)事前準備
```shell
cd ＜ソースコードのディレクトリ＞
export KeyName=＜利用するキーペア名称＞
export Profile=＜aws cliに設定したプロファイル名＞
export ADPassword=＜ADに設定するパスワード＞
export ADname="wkstest.mycorporation.local"


```
- ADのパスワードは、8～64 文字で指定し、「admin」という語は含めず、英小文字、英大文字、数字、特殊文字の 4 つのカテゴリのうちの 3 つを含める必要があります。

(2)Workspaces VDI用　VPC作成(commonVPC)
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-CommonVPC --template-body "file://${PWD}/Account-2/CommonVPC.yaml" --capabilities CAPABILITY_NAMED_IAM 
```
(3)AD用　VPC作成(operationVPC)
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-OperationVPC --template-body "file://${PWD}/Account-2/OperationVPC.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```
(4)CommonVPCとOperationVPCのVPCPeering接続
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-VpcPeering --template-body "file://${PWD}/Account-2/vpcpeer.yaml"
```
(5)Managed Microsoft ADデプロイ
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-AD --template-body "file://${PWD}/Account-2/ad.yaml" --parameters "ParameterKey=ADPassword,ParameterValue=${ADPassword}" "ParameterKey=ADname,ParameterValue=${ADname}" --timeout-in-minutes 60
```
(6)Bastion(Windows)へのAD管理ツールセットアップとAD参加
+ OperationVPCのBasiotnサーバ(Windows)にRDPでログインする
+ AD管理に必要なツールをPowerShellでインストールする
```
Install-WindowsFeature -Name GPMC,RSAT-AD-PowerShell,RSAT-AD-AdminCenter,RSAT-ADDS-Tools,RSAT-DNS-Server

詳細はこちらを参照：https://docs.aws.amazon.com/ja_jp/directoryservice/latest/admin-guide/microsoftadbasestep3.html
```
+ `コントロールパネル`-`ネットワーク設定`から、ネットワーク設定で優先MS `ADのDNSアドレス`を入力する
+ `コントロールパネル`-`システム`から`コンピュータ名/ドメイン名の変更`を開き、所属グループで、`ドメイン`を選択しADの`Directory DNS name`を指定する
+ リブートすると、WIndowsがドメインに所属される
+ ADのAdminユーザでRPDからログインして状態を確認する

(7)Workspacesユーザの追加
+ `ServerManager`を起動し、右上のメニューバーから`tool`->`ActiveDirectory Users and Computers`を選択する
+ ADの`ショート名`の下の`Users`の下に、Workspaces用のユーザを作成する。作成する際に、メールアドレスも合わせて登録する。

(8)AD connectorデプロイ
```shell
AdId=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-AD --query "Stacks[].Outputs[?OutputKey==\`ADId\`].OutputValue")
DNSIPS="$(aws --profile ${Profile} --output text ds describe-domain-controllers --directory-id ${AdId} --query 'DomainControllers[0].DnsIpAddr'),$(aws --profile ${Profile} --output text ds describe-domain-controllers --directory-id ${AdId} --query 'DomainControllers[1].DnsIpAddr')"
CommonVpcId=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-CommonVPC --query "Stacks[].Outputs[?OutputKey==\`VpcId\`].OutputValue")
Subnet1Id=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-CommonVPC --query "Stacks[].Outputs[?OutputKey==\`Subnet1Id\`].OutputValue")
Subnet2Id=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-CommonVPC --query "Stacks[].Outputs[?OutputKey==\`Subnet2Id\`].OutputValue")
aws --profile ${Profile} ds connect-directory --name "${ADname}" --password "${ADPassword}" --size Small --connect-settings "VpcId=${CommonVpcId},SubnetIds=${Subnet1Id},${Subnet2Id},CustomerDnsIps=${DNSIPS},CustomerUserName=Admin"
```

(7)Workspaces作成
+ マネージメントコンソールで、Workspacesを開く
+ ADは、作成したAD connectorを選択する
+ 好きなイメージを選び起動する




## VPNのみ構成
(1)事前準備
```shell
cd ＜ソースコードのディレクトリ＞
export Profile=＜aws cliに設定したプロファイル名＞
export KeyName=＜利用するキーペア名称＞
```
(2)Outbound VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-OutboundVPC --template-body "file://${PWD}/Account-1-VPN/OutboundVPC.yaml" --capabilities CAPABILITY_NAMED_IAM 
```

(3)Client VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-ClientVPC --template-body "file://${PWD}/Account-1-VPN/ClientVPC.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```

(4)Vyattaインスタンスの Source/Dest. Checkの無効化
```shell
VyOS=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-ClientVPC --query "Stacks[].Outputs[?OutputKey==\`CustomerGwId\`].OutputValue")
echo ${VyOS}
aws --profile ${Profile} ec2 modify-instance-attribute --instance-id ${VyOS} --source-dest-check "{\"Value\": false}"
```

(5)OutboundVPCとClientVPCのVPN接続
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-VPN --template-body "file://${PWD}/Account-1-VPN/VPN.yaml"
```

## VPN+boutoundVPCにProxyを設置する構成
(1)事前準備
```shell
cd ＜ソースコードのディレクトリ＞
export Profile=＜aws cliに設定したプロファイル名＞
export KeyName=＜利用するキーペア名称＞
```
(2)Outbound VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-OutboundVPC --template-body "file://${PWD}/Account-1-Proxy/OutboundVPC-Proxy.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```