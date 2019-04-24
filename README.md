# AWS-sidebyside-vpn-by-vyos



## Account-2: Workspaces構成
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


## Account-1: VPNのみ構成
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

(4)OutboundVPCとClientVPCのVPN接続
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-VPN --template-body "file://${PWD}/Account-1-VPN/VPN.yaml"
```

(5)VPN設定のダウンロード
Vyattaに設定するIPSec情報をマネージメントコンソールからダウンロードします。
Vyattaの場合、ベンダーは”Vyatta”を選択します。
![VPN設定ダウンロード](https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/download_VPN_configuration.png)

(6)設定ファイルの修正(VyOSのインスタンスIP修正)
ダウンロードした設定ファイルのうち、検証では IPSec Tunnel #1のみ利用します。
また設定では、 vyattaのパブリックIPになっているため、この部分をVyOSインスタンスのプライベートIPに修正します。
![VPN設定変更](https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/change_VPN_configuration.png)

(7)VyOS設定
+ ログイン   ※vyattaユーザでログインします
```
ssh –i SSH秘密鍵ファイル   vyatta@VyOSインスタンスのパブリックIP
```
+ 設定(ダウンロードした定義ファイルを流し込み、commit, saveで設定を記録する)
```
Welcome to Vyatta
Linux vyatta-64bit 3.3.8-1-amd64-vyatta #1 SMP Mon Feb 17 14:46:16 PST 2014 x86_64
Welcome to Vyatta.
This system is open-source software. The exact distribution terms for 
each module comprising the full system are described in the individual 
files in /usr/share/doc/*/copyright.
Last login: Mon Feb 17 23:44:05 2014
vyatta@vyatta-64bit:~$ configure

ダウンロードして修正した設定ファイルの
「#1: Internet Key Exchange (IKE) Configuration」〜「 #4: Border Gateway Protocol (BGP) Configuration 」までの setコマンドを全て流す

vyatta@vyatta-64bit# commit
[edit]
# save
Saving configuration to '/config/config.boot'...
Done
[edit]

# exit
exit
vyatta@vyatta-64bit:~$ 

```



## Account-1: VPN+boutoundVPCにProxyを設置する構成
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
(3)以降 Client VPC作成以降

→VPNのみ構成を参照
