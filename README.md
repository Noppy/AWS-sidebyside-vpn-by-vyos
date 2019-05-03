# AWS-sidebyside-vpn-by-vyos
オンプレからインターネットに接続するためのOubtound専用DMZをVPCで実装しVPNでオンプレと接続。オンプレのクライアントからOutoboundVPC経由でWorkSpaces接続する構成の検証。
検証でオンプレ環境は、VPCで代用。
OutboundVPCとして下記４パターンを作成し動作確認を実施
- パターン1: 単純VPC接続(ForwardProxyなし)
- パターン2: VPC接続+ForwardProxy接続構成
- パターン3: VPN on VPN接続＋ForwardProxy接続構成
- パターン4: TVPN + ransitGateway + ForwardProxy接続構成

# 検証環境作成手順
## 事前準備
(1) Marketplaceでのvyattaのsubscribe  
Marketplaceで、”Vyatta (Community Edition) (HVM)”を検索し、予めSubscribeする。  
(先にこの手順を行わないと、CloudFormationでインスタンスのデプロイができないため)  
<img src="https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/subscribe_vyatt.png" width="600">

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


## Account-1: (パターン１)VPNのみ構成
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

(5)VyOSスタティックルート設定
+ ログイン   ※vyosユーザでログインします
```shell
ssh –i SSH秘密鍵ファイル   vyos@VyOSインスタンスのパブリックIP
```
+ VyOS Static route設定
    + メンテナンス用通信、VPN接続先のAWS VPNの「サイト間のVPN接続」(Site-to-Site VPN Connection)のPublicIPは、SubnetのGWを利用するよう設定を行う。(デフォルトGWをVPNのtunnel接続先に変更するため、tunnelを介してはいけない通信をstatic routeで設定する)
    + 前提(実態に合わせ適時修正)
        + メンテ用CIDR: `XX.XX.XX.0/24`
        + VPN接続先のAWS側のPublicIP: `13.114.183.29`, `52.199.46.211`
        + SubnetのデフォルトGW: `172.16.1.1`

    + メンテナンス通信用のstatic route追加
    ```shell
    Welcome to VyOS.
    This system is open-source software. The exact distribution terms for 
    each module comprising the full system are described in the individual 
    files in /usr/share/doc/*/copyright.
    Last login: Fri Aug 11 15:26:32 2017
    vyos@vyos:~$ configure
    [edit]
    vyos@vyos# set protocols static route 27.0.3.0/24 next-hop 172.16.1.1
    [edit]
    vyos@vyos# commit
    [edit]
    vyos@vyos# save
    Saving configuration to '/config/config.boot'...
    Done
    [edit]
    vyos@vyos# exit
    exit
    vyos@vyos:~$ show ip route
    Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
           I - ISIS, B - BGP, > - selected route, * - FIB route

    S>* 0.0.0.0/0 [210/0] via 172.16.1.1, eth0
    S>* 27.0.3.0/24 [1/0] via 172.16.1.1, eth0
    C>* 127.0.0.0/8 is directly connected, lon
    C>* 172.16.1.0/24 is directly connected, eth0
    ```
    + VPNの対向のAWSのPublic IPのStatic route設定
    ```
    $ configure
    # set protocols static route 13.114.183.29/32  next-hop 172.16.1.1
    # set protocols static route 52.199.46.211/32  next-hop 172.16.1.1
    # commit
    # save
    # exit
    $ show ip route
    ```

    + デフォルトGWをIPSecのinterface(vti0)に変更する
    ```
    $ configure
    # set protocols static route 172.16.0.0/16 next-hop 172.16.1.1
    # set protocols static interface-route 0.0.0.0/0 next-hop-interface vti0
    # commit
    # save
    # exit
    $ show ip route
    ```    

(6)VPN設定のダウンロード  
VyOSに設定するIPSec情報をマネージメントコンソールからダウンロードする。  
VyOSの場合、ベンダーは”Vyatta”を選択します。  
![VPN設定ダウンロード](https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/download_VPN_configuration.png)

(7)設定ファイルの修正(VyOSのインスタンスIP修正)  
ダウンロードした設定ファイルのうち、検証では IPSec Tunnel #1のみ利用します。下記を修正する。
+ ＃1 IKE設定(IPSecプロトコルの一つ。秘密鍵情報の交換用プロトコル)
    + set vpn ipsec ike-group AWS proposal 1 encryption 'aes128': encriptionを、`aes128`→`aes256`に変更
    + set vpn ipsec site-to-site peer 13.114.183.29 local-address '13.231.208.95': local-addressを、PublicIPの`13.231.208.95`からVyattaのPrivateIPの`172.16.1.200`に変更
+ #2 IPSec ESP設定(IPパケットに認証暗号化を設定する通信プロトコル)
    + set vpn ipsec esp-group AWS proposal 1 encryption 'aes128': encriptionを、`aes128`→`aes256`に変更
+ #3 Tunnel内の設定
    + 変更点なし
+ #4 BGP設定
    + set protocols bgp 65000 network 0.0.0.0/0: BGPの広告対象範囲のCIDRを指定。デフォルトでは`全て`のため`ClientVPCのCIDR範囲`に限定:`0.0.0.0/0`→`172.16.0.0/16`

変更後のコンフィグ設定
```text
! --------------------------------------------------------------------------------
! IPSec Tunnel #1
! --------------------------------------------------------------------------------
! #1: Internet Key Exchange (IKE) Configuration
!
! A policy is established for the supported ISAKMP encryption,
! authentication, Diffie-Hellman, lifetime, and key parameters.
! Please note, these sample configurations are for the minimum requirement of AES128, SHA1, and DH Group 2.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! You will need to modify these sample configuration files to take advantage of AES256, SHA256, or other DH groups like 2, 14-18, 22, 23, and 24.
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
! The address of the external interface for your customer gateway must be a static address.
! Your customer gateway may reside behind a device performing network address translation (NAT).
! To ensure that NAT traversal (NAT-T) can function, you must adjust your firewall !rules to unblock UDP port 4500. If not behind NAT, we recommend disabling NAT-T.
!

set vpn ipsec ike-group AWS lifetime '28800'
set vpn ipsec ike-group AWS proposal 1 dh-group '2'
set vpn ipsec ike-group AWS proposal 1 encryption 'aes256'
set vpn ipsec ike-group AWS proposal 1 hash 'sha1'
set vpn ipsec site-to-site peer 13.114.183.29 authentication mode 'pre-shared-secret'
set vpn ipsec site-to-site peer 13.114.183.29 authentication pre-shared-secret 'gZStKK2jqOOm12sf2wnNhgWOTZIbYj3h'
set vpn ipsec site-to-site peer 13.114.183.29 description 'VPC tunnel 1'
set vpn ipsec site-to-site peer 13.114.183.29 ike-group 'AWS'
set vpn ipsec site-to-site peer 13.114.183.29 local-address '172.16.1.200'
set vpn ipsec site-to-site peer 13.114.183.29 vti bind 'vti0'
set vpn ipsec site-to-site peer 13.114.183.29 vti esp-group 'AWS'


! #2: IPSec Configuration
!
! The IPSec (Phase 2) proposal defines the protocol, authentication,
! encryption, and lifetime parameters for our IPSec security association.
! Category "VPN" connections in the GovCloud region have a minimum requirement of AES128, SHA2, and DH Group 14.
! Please note, you may use these additionally supported IPSec parameters for encryption like AES256 and other DH groups like 2, 5, 14-18, 22, 23, and 24.
! Higher parameters are only available for VPNs of category "VPN," and not for "VPN-Classic".
!

set vpn ipsec ipsec-interfaces interface 'eth0'
set vpn ipsec esp-group AWS compression 'disable'
set vpn ipsec esp-group AWS lifetime '3600'
set vpn ipsec esp-group AWS mode 'tunnel'
set vpn ipsec esp-group AWS pfs 'enable'
set vpn ipsec esp-group AWS proposal 1 encryption 'aes256'
set vpn ipsec esp-group AWS proposal 1 hash 'sha1'

! This option enables IPSec Dead Peer Detection, which causes periodic
! messages to be sent to ensure a Security Association remains operational.
!
set vpn ipsec ike-group AWS dead-peer-detection action 'restart'
set vpn ipsec ike-group AWS dead-peer-detection interval '15'
set vpn ipsec ike-group AWS dead-peer-detection timeout '30'

! --------------------------------------------------------------------------------
! #3: Tunnel Interface Configuration
!
!  The tunnel interface is configured with the internal IP address.

set interfaces vti vti0 address '169.254.26.186/30'
set interfaces vti vti0 description 'VPC tunnel 1'
set interfaces vti vti0 mtu '1436'

! --------------------------------------------------------------------------------

! #4: Border Gateway Protocol (BGP) Configuration
!
! BGP is used within the tunnel to exchange prefixes between the
! Virtual Private Gateway and your Customer Gateway. The Virtual Private Gateway
! will announce the prefix corresponding to your VPC.
!
! Your Customer Gateway may announce a default route (0.0.0.0/0),
! which can be done with the 'network' statement.
!
! The BGP timers are adjusted to provide more rapid detection of outages.
!
! The local BGP Autonomous System Number (ASN) (65000) is configured
! as part of your Customer Gateway. If the ASN must be changed, the
! Customer Gateway and VPN Connection will need to be recreated with AWS.
!

set protocols bgp 65000 neighbor 169.254.26.185 remote-as '64512'
set protocols bgp 65000 neighbor 169.254.26.185 soft-reconfiguration 'inbound'
set protocols bgp 65000 neighbor 169.254.26.185 timers holdtime '30'
set protocols bgp 65000 neighbor 169.254.26.185 timers keepalive '10'

! To advertise additional prefixes to Amazon VPC, replace the 0.0.0.0/0 from the
! the following line with the prefix you wish to advertise. Make sure the prefix is present
! in the routing table of the device with a valid next-hop.

set protocols bgp 65000 network 172.16.0.0/16
```

(8)VyOSへのIPSec設定(ダウンロードした定義ファイルの修正版を流し込み、commit, saveで設定を記録する)
```shell
$ configure

ダウンロードして修正した設定ファイルの
「#1: Internet Key Exchange (IKE) Configuration」〜「 #4: Border Gateway Protocol (BGP) Configuration 」までの setコマンドを全て流す

# commit
# save
# exit
exit
```

## Account-1: (パターン2)VPN+boutoundVPCにProxyを設置する構成
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


## Account-1: (パターン4)OutboundVPC(Proxy構成)+VPN+TGW構成
(1)事前準備
```shell
cd ＜ソースコードのディレクトリ＞
export Profile=＜aws cliに設定したプロファイル名＞
export KeyName=＜利用するキーペア名称＞
```
(2)Outbound VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-OutboundVPC --template-body "file://${PWD}/Account-1-TGW/OutboundVPC-Proxy.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```
(3)Client VPC作成  
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-ClientVPC --template-body "file://${PWD}/Account-1-TGW/ClientVPC.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```
(4)Transit Gateway作成  
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-TGW --template-body "file://${PWD}/Account-1-TGW/TGW.yaml"
```

(5)ClientVPとのTGWのVPN接続とTGWのRouteTableのアタッチ  
```shell
# Attach VPN to TGW
CustomerGWId=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-TGW --query "Stacks[].Outputs[?OutputKey==\`CustomerGWId\`].OutputValue")
TGWId=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-TGW --query "Stacks[].Outputs[?OutputKey==\`TgwVpnId\`].OutputValue")
TGWRouteID=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-TGW --query "Stacks[].Outputs[?OutputKey==\`TgwRouteTableId\`].OutputValue")
aws --profile ${Profile} ec2 create-vpn-connection --type "ipsec.1" --customer-gateway-id "${CustomerGWId}" --transit-gateway-id "${TGWId}"

# Add Routetable to VPN attachment as associate and propagation
AttacheTgwToVpn=$(aws --profile ${Profile} --output text ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=${TGWId}" "Name=resource-type,Values=vpn" --query "TransitGatewayAttachments[].TransitGatewayAttachmentId")
while [ "A$(aws --profile ${Profile} --output text ec2 describe-transit-gateway-attachments --transit-gateway-attachment-ids ${AttacheTgwToVpn} --query "TransitGatewayAttachments[].State")" != "Aavailable" ];do echo "Attachment(VPN) Status is not available.sleep 5secs";done
aws --profile ${Profile} ec2 associate-transit-gateway-route-table --transit-gateway-route-table-id ${TGWRouteID} --transit-gateway-attachment-id ${AttacheTgwToVpn}
aws --profile ${Profile} ec2 enable-transit-gateway-route-table-propagation --transit-gateway-route-table-id ${TGWRouteID} --transit-gateway-attachment-id ${AttacheTgwToVpn}
```

(6)OutbounVPCにTGWへのルーティング追加  
変数の設定
```shell
TGWId=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-TGW --query "Stacks[].Outputs[?OutputKey==\`TgwVpnId\`].OutputValue")
PubRouteTable=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-OutboundVPC --query "Stacks[].Outputs[?OutputKey==\`PublicSubnetRouteTableId\`].OutputValue")
PrivateRouteTable1=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-OutboundVPC --query "Stacks[].Outputs[?OutputKey==\`PrivateSubnet1RouteTableId\`].OutputValue")
PribateRouteTable2=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-OutboundVPC --query "Stacks[].Outputs[?OutputKey==\`PrivateSubnet2RouteTableId\`].OutputValue")
ClientCIDR=$(aws --profile ${Profile} --output text cloudformation describe-stacks --stack-name Dev-ClientVPC --query "Stacks[].Outputs[?OutputKey==\`VpcCidr\`].OutputValue")
echo "TGWId=$TGWId PubRouteTable=$PubRouteTable PrivateRouteTable1=$PrivateRouteTable1 PribateRouteTable2=$PribateRouteTable2 ClientCIDR=$ClientCIDR"
```
ルーティング追加
```shell
aws --profile ${Profile} ec2 create-route --route-table-id ${PubRouteTable} --destination-cidr-block ${ClientCIDR} --transit-gateway-id ${TGWId}
aws --profile ${Profile} ec2 create-route --route-table-id ${PrivateRouteTable1} --destination-cidr-block ${ClientCIDR} --transit-gateway-id ${TGWId}
aws --profile ${Profile} ec2 create-route --route-table-id ${PribateRouteTable2} --destination-cidr-block ${ClientCIDR} --transit-gateway-id ${TGWId}
```

(7)VyOSスタティックルート設定  
→「Account-1: (パターン１)VPNのみ構成」の「(5)VyOSスタティックルート設定」参照

(8)VyOS設定  
→「Account-1: (パターン１)VPNのみ構成」の「(6)VPN設定のダウンロード」以降を参照









## Account-1: (パターン3)OutboudVPC(Proxy構成)+VPN on VPN
(1)事前準備
```shell
cd ＜ソースコードのディレクトリ＞
export Profile=＜aws cliに設定したプロファイル名＞
export KeyName=＜利用するキーペア名称＞
```
(2)Outbound VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack  --stack-name Dev-OutboundVPC --template-body "file://${PWD}/Account-1-VPNonVPN/OutboundVPC-Proxy-VPNonVPN.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```
(3)Client VPC作成
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-ClientVPC --template-body "file://${PWD}/Account-1-VPNonVPN/ClientVPC-VPNonVPN.yaml" --capabilities CAPABILITY_NAMED_IAM --parameters "ParameterKey=KeyName,ParameterValue=${KeyName}"
```
(4)OutboundVPCとClientVPCのVPN接続
```shell
aws --profile ${Profile} cloudformation create-stack --stack-name Dev-VPN --template-body "file://${PWD}/Account-1-VPNonVPN/VPN.yaml"
```

(5)VyOSスタティックルート設定
+ ログイン   ※vyosユーザでログインします
```shell
ssh –i SSH秘密鍵ファイル   vyos@VyOSインスタンスのパブリックIP
```
+ VyOS Static route設定
    + ClientVPCのPublicSub2のルーティング追加
    ```shell
    Welcome to VyOS.
    This system is open-source software. The exact distribution terms for 
    each module comprising the full system are described in the individual 
    files in /usr/share/doc/*/copyright.
    Last login: Fri Aug 11 15:26:32 2017
    vyos@vyos:~$ configure
    [edit]
    vyos@vyos# set protocols static route 172.16.2.0/24 next-hop 172.16.1.1
    [edit]
    vyos@vyos# commit
    [edit]
    vyos@vyos# save
    Saving configuration to '/config/config.boot'...
    Done
    [edit]
    vyos@vyos# exit
    exit
    vyos@vyos:~$ show ip route
    Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
           I - ISIS, B - BGP, > - selected route, * - FIB route

    S>* 0.0.0.0/0 [210/0] via 172.16.1.1, eth0
    C>* 127.0.0.0/8 is directly connected, lo
    C>* 172.16.1.0/24 is directly connected, eth0
    S>* 172.16.2.0/24 [1/0] via 172.16.1.1, eth0
    ```

(6)VPN設定のダウンロード  
VyOSに設定するIPSec情報をマネージメントコンソールからダウンロードする。  
VyOSの場合、ベンダーは”Vyatta”を選択します。  
![VPN設定ダウンロード](https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/download_VPN_configuration.png)

(7)設定ファイルの修正(VyOS-1のインスタンスIP修正)  
ダウンロードした設定ファイルのうち、検証では IPSec Tunnel #1のみ利用します。下記を修正する。
+ ＃1 IKE設定(IPSecプロトコルの一つ。秘密鍵情報の交換用プロトコル)
    + set vpn ipsec ike-group AWS proposal 1 encryption 'aes128': encriptionを、`aes128`→`aes256`に変更
    + set vpn ipsec site-to-site peer 13.114.183.29 local-address '13.231.208.95': local-addressを、PublicIPからVyattaのPrivateIPの`172.16.1.200`に変更
+ #2 IPSec ESP設定(IPパケットに認証暗号化を設定する通信プロトコル)
    + set vpn ipsec esp-group AWS proposal 1 encryption 'aes128': encriptionを、`aes128`→`aes256`に変更
+ #3 Tunnel内の設定
    + 変更点なし
+ #4 BGP設定
    + set protocols bgp 65000 network 0.0.0.0/0: BGPの広告対象範囲のCIDRを指定。デフォルトでは`全て`のため`Public Subnet1のCIDR範囲`に限定:`0.0.0.0/0`→`172.16.1.0/24`と`172.16.2.0/24`の2つを設定

変更後のコンフィグ設定
```text
! --------------------------------------------------------------------------------
! IPSec Tunnel #1
! --------------------------------------------------------------------------------
! #1: Internet Key Exchange (IKE) Configuration

set vpn ipsec ike-group AWS lifetime '28800'
set vpn ipsec ike-group AWS proposal 1 dh-group '2'
set vpn ipsec ike-group AWS proposal 1 encryption 'aes256'
set vpn ipsec ike-group AWS proposal 1 hash 'sha1'
set vpn ipsec site-to-site peer 13.114.183.29 authentication mode 'pre-shared-secret'
set vpn ipsec site-to-site peer 13.114.183.29 authentication pre-shared-secret 'gZStKK2jqOOm12sf2wnNhgWOTZIbYj3h'
set vpn ipsec site-to-site peer 13.114.183.29 description 'VPC tunnel 1'
set vpn ipsec site-to-site peer 13.114.183.29 ike-group 'AWS'
set vpn ipsec site-to-site peer 13.114.183.29 local-address '172.16.1.200'
set vpn ipsec site-to-site peer 13.114.183.29 vti bind 'vti0'
set vpn ipsec site-to-site peer 13.114.183.29 vti esp-group 'AWS'

! #2: IPSec Configuration

set vpn ipsec ipsec-interfaces interface 'eth0'
set vpn ipsec esp-group AWS compression 'disable'
set vpn ipsec esp-group AWS lifetime '3600'
set vpn ipsec esp-group AWS mode 'tunnel'
set vpn ipsec esp-group AWS pfs 'enable'
set vpn ipsec esp-group AWS proposal 1 encryption 'aes256'
set vpn ipsec esp-group AWS proposal 1 hash 'sha1'

! This option enables IPSec Dead Peer Detection, which causes periodic
! messages to be sent to ensure a Security Association remains operational.
!

set vpn ipsec ike-group AWS dead-peer-detection action 'restart'
set vpn ipsec ike-group AWS dead-peer-detection interval '15'
set vpn ipsec ike-group AWS dead-peer-detection timeout '30'

! --------------------------------------------------------------------------------
! #3: Tunnel Interface Configuration

set interfaces vti vti0 address '169.254.26.186/30'
set interfaces vti vti0 description 'VPC tunnel 1'
set interfaces vti vti0 mtu '1436'

! --------------------------------------------------------------------------------

! #4: Border Gateway Protocol (BGP) Configuration

set protocols bgp 65000 neighbor 169.254.26.185 remote-as '64512'
set protocols bgp 65000 neighbor 169.254.26.185 soft-reconfiguration 'inbound'
set protocols bgp 65000 neighbor 169.254.26.185 timers holdtime '30'
set protocols bgp 65000 neighbor 169.254.26.185 timers keepalive '10'

! To advertise additional prefixes to Amazon VPC, replace the 0.0.0.0/0 from the
! the following line with the prefix you wish to advertise. Make sure the prefix is present
! in the routing table of the device with a valid next-hop.

set protocols bgp 65000 network 172.16.1.0/24
set protocols bgp 65000 network 172.16.2.0/24
```

(8)VyOSへのIPSec設定(ダウンロードした定義ファイルの修正版を流し込み、commit, saveで設定を記録する)

+ ログイン   ※vyosユーザでログインします
```shell
ssh –i SSH秘密鍵ファイル   vyos@VyOSインスタンスのパブリックIP
```
```shell
$ configure

ダウンロードして修正した設定ファイルの「#1: Internet Key Exchange (IKE) Configuration」〜「 #4: Border Gateway Protocol (BGP) Configuration 」までの setコマンドを全て流す

# commit
# save
# exit
$ show interfaces vti detail 
vti0@NONE: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1436 qdisc noqueue state UNKNOWN group default 
    link/ipip 172.16.1.200 peer 52.68.216.47
    inet 169.254.27.246/30 scope global vti0
       valid_lft forever preferred_lft forever
    Description: VPC tunnel 1

    RX:  bytes    packets     errors    dropped    overrun      mcast
           834         12          0          0          0          0
    TX:  bytes    packets     errors    dropped    carrier collisions
          2116         14          0          0          0          0

$ show vpn ipsec sa
Peer ID / IP                            Local ID / IP               
------------                            -------------
13.114.183.29                           172.16.1.200                           

    Description: VPC tunnel 1

    Tunnel  State  Bytes Out/In   Encrypt  Hash    NAT-T  A-Time  L-Time  Proto
    ------  -----  -------------  -------  ----    -----  ------  ------  -----
    vti     up     463.0/413.0    aes256   sha1    no     996     3600    all

$ show ip route 
Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
       I - ISIS, B - BGP, > - selected route, * - FIB route

S>* 0.0.0.0/0 [210/0] via 172.16.1.1, eth0
B>* 10.1.0.0/16 [20/100] via 169.254.27.245, vti0, 00:00:21
C>* 127.0.0.0/8 is directly connected, lo
C>* 169.254.27.244/30 is directly connected, vti0
C>* 172.16.1.0/24 is directly connected, eth0
S>* 172.16.2.0/24 [1/0] via 172.16.1.1, eth0
```

(9) VyOS-2とVyOS-3の間のIPSec設定  
(a)設定概要  


(b)VyOS-3の設定(OutboundVPCのVyOS)  
- ログインする(ClientVPCのPublicSub2のBastionForOutbounndRouter経由でVyOS-3にSSHログインする)
- 下記設定を行う
    - vti設定
    - IKE設定
    - ESP設定
    - Side by SideのVPN接続
    - BGP設定
    - ルーティング設定
    ```shell
    #コンフィグモードへの変更
    configure 

    #(i) 仮想インターフェース(vti)設定
    set interfaces vti vti0 address '169.254.24.117/30'
    set interfaces vti vti0 description 'VPN over VPN'
    set interfaces vti vti0 mtu '1374'
    commit

    #(ii) IKE,ESP, Side-by-SideのVPN接続まで一気に設定する
    # Set IKE
    set vpn ipsec ike-group VPNoverVPN lifetime 28800
    set vpn ipsec ike-group VPNoverVPN proposal 1 dh-group 2
    set vpn ipsec ike-group VPNoverVPN proposal 1 encryption aes256
    set vpn ipsec ike-group VPNoverVPN proposal 1 hash sha1

    # Set ESP
    set vpn ipsec ipsec-interfaces interface 'eth0'
    set vpn ipsec esp-group VPNoverVPN compression 'disable'
    set vpn ipsec esp-group VPNoverVPN lifetime '3600'
    set vpn ipsec esp-group VPNoverVPN mode 'tunnel'
    set vpn ipsec esp-group VPNoverVPN pfs 'enable'
    set vpn ipsec esp-group VPNoverVPN proposal 1 encryption 'aes256'
    set vpn ipsec esp-group VPNoverVPN proposal 1 hash 'sha1'

    set vpn ipsec ike-group VPNoverVPN dead-peer-detection action 'restart'
    set vpn ipsec ike-group VPNoverVPN dead-peer-detection interval '15'
    set vpn ipsec ike-group VPNoverVPN dead-peer-detection timeout '30'

    #Side-by-Side
    set vpn ipsec site-to-site peer 172.16.2.200 authentication mode 'pre-shared-secret'
    set vpn ipsec site-to-site peer 172.16.2.200 authentication pre-shared-secret 'PbL9imJdaUudQZiOnq06yIgti9bIs_Xq'
    set vpn ipsec site-to-site peer 172.16.2.200 description 'VPN over VPN'
    set vpn ipsec site-to-site peer 172.16.2.200 ike-group 'VPNoverVPN'
    set vpn ipsec site-to-site peer 172.16.2.200 local-address '10.1.80.200'
    set vpn ipsec site-to-site peer 172.16.2.200 vti bind 'vti0'
    set vpn ipsec site-to-site peer 172.16.2.200 vti esp-group 'VPNoverVPN'
    commit

    #(iii) BGP設定
    set protocols bgp 65499 neighbor 169.254.24.118 remote-as '65498'
    set protocols bgp 65499 neighbor 169.254.24.118 soft-reconfiguration 'inbound'
    set protocols bgp 65499 neighbor 169.254.24.118 timers holdtime '30'
    set protocols bgp 65499 neighbor 169.254.24.118 timers keepalive '10'

    set protocols bgp 65499 network 10.1.0.0/16

    #(iv)Static route設定
    set protocols static route 172.16.3.0/24 next-hop 169.254.24.117
    commit

    #showコマンドで設定内容確認しセーブ後、コンフィグモード終了
    show
    save
    ```
(c)VyOS-2の設定(ClientVPC Subnet2のVyOS)  
- ログインする
- 下記設定を行う
    - vti設定
    - IKE設定
    - ESP設定
    - Side by SideのVPN接続
    - BGP設定
    - ルーティング設定
    ```shell
    #コンフィグモードへの変更
    configure 

    #(i) 仮想インターフェース(vti)設定
    set interfaces vti vti0 address '169.254.24.118/30'
    set interfaces vti vti0 description 'VPN over VPN'
    set interfaces vti vti0 mtu '1374'
    commit

    #(ii) IKE,ESP, Side-by-SideのVPN接続まで一気に設定する
    # Set IKE
    set vpn ipsec ike-group VPNoverVPN lifetime 28800
    set vpn ipsec ike-group VPNoverVPN proposal 1 dh-group 2
    set vpn ipsec ike-group VPNoverVPN proposal 1 encryption aes256
    set vpn ipsec ike-group VPNoverVPN proposal 1 hash sha1

    # Set ESP
    set vpn ipsec ipsec-interfaces interface 'eth0'
    set vpn ipsec esp-group VPNoverVPN compression 'disable'
    set vpn ipsec esp-group VPNoverVPN lifetime '3600'
    set vpn ipsec esp-group VPNoverVPN mode 'tunnel'
    set vpn ipsec esp-group VPNoverVPN pfs 'enable'
    set vpn ipsec esp-group VPNoverVPN proposal 1 encryption 'aes256'
    set vpn ipsec esp-group VPNoverVPN proposal 1 hash 'sha1'

    set vpn ipsec ike-group VPNoverVPN dead-peer-detection action 'restart'
    set vpn ipsec ike-group VPNoverVPN dead-peer-detection interval '15'
    set vpn ipsec ike-group VPNoverVPN dead-peer-detection timeout '30'

    #Side-by-Side
    set vpn ipsec site-to-site peer 10.1.80.200 authentication mode 'pre-shared-secret'
    set vpn ipsec site-to-site peer 10.1.80.200 authentication pre-shared-secret 'PbL9imJdaUudQZiOnq06yIgti9bIs_Xq'
    set vpn ipsec site-to-site peer 10.1.80.200 description 'VPN over VPN'
    set vpn ipsec site-to-site peer 10.1.80.200 ike-group 'VPNoverVPN'
    set vpn ipsec site-to-site peer 10.1.80.200 local-address '172.16.2.200'
    set vpn ipsec site-to-site peer 10.1.80.200 vti bind 'vti0'
    set vpn ipsec site-to-site peer 10.1.80.200 vti esp-group 'VPNoverVPN'
    commit

    #(iii) BGP設定
    set protocols bgp 65498 neighbor 169.254.24.117 remote-as '65499'
    set protocols bgp 65498 neighbor 169.254.24.117 soft-reconfiguration 'inbound'
    set protocols bgp 65498 neighbor 169.254.24.117 timers holdtime '30'
    set protocols bgp 65498 neighbor 169.254.24.117 timers keepalive '10'
    set protocols bgp 65498 network 10.1.0.0/16
    commit

    #(iv)Static route設定
    set protocols static route 27.0.3.0/24   next-hop 172.16.2.1
    set protocols static route 172.16.0.0/16 next-hop 172.16.2.1
    set protocols static route 10.1.80.0/24  next-hop 172.16.2.1
    set protocols static route 10.1.144.0/24 next-hop 172.16.2.1
    set protocols static interface-route 0.0.0.0/0 next-hop-interface vti0
    commit

    #showコマンドで設定内容確認しセーブ後、コンフィグモード終了
    show
    save
    ```
 (d)状態確認  
 vpnがアップしているか状態を確認する。
 ```shell
vyos@vyos:~$ show vpn ipsec sa
Peer ID / IP                            Local ID / IP               
------------                            -------------
172.16.2.200                            10.1.80.200                            

    Description: VPN over VPN

    Tunnel  State  Bytes Out/In   Encrypt  Hash    NAT-T  A-Time  L-Time  Proto
    ------  -----  -------------  -------  ----    -----  ------  ------  -----
    vti     up     0.0/0.0        aes256   sha1    no     1016    3600    all
```



https://ecl.ntt.com/documents/tutorials/rsts/networkfunction/function_c_a.html
https://qiita.com/Mitsu-Murakita/items/9bb09f54494345b51ce8