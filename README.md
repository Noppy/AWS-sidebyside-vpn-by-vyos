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

(5)Vyattaスタティックルート設定
+ ログイン   ※vyattaユーザでログインします
```shell
ssh –i SSH秘密鍵ファイル   vyatta@VyOSインスタンスのパブリックIP
```
+ Vyatta Static route設定
    + メンテナンス用通信、VPN接続先のAWS VPNの「サイト間のVPN接続」(Site-to-Site VPN Connection)のPublicIPは、SubnetのGWを利用するよう設定を行う。(デフォルトGWをVPNのtunnel接続先に変更するため、tunnelを介してはいけない通信をstatic routeで設定する)
    + 前提(実態に合わせ適時修正)
        + メンテ用CIDR: `XX.XX.XX.0/24`
        + VPN接続先のAWS側のPublicIP: `13.114.183.29`, `52.199.46.211`
        + SubnetのデフォルトGW: `172.16.1.1`
        + vyattaのPrivateIP: `172.16.1.200`

    + メンテナンス通信用のstatic route追加
    ```shell
    Welcome to Vyatta
    Linux vyatta-64bit 3.3.8-1-amd64-vyatta #1 SMP Mon Feb 17 14:46:16 PST 2014 x86_64
    Welcome to Vyatta.
    <中略>
    vyatta@vyatta-64bit:~$ configure
    [edit]
    vyatta@vyatta-64bit# set protocols static route XX.XX.XX.0/24 next-hop 172.16.1.1
    [edit]
    vyatta@vyatta-64bit# commit
    [edit]
    vyatta@vyatta-64bit# save
    Saving configuration to '/config/config.boot'...
    Done
    [edit]
    vyatta@vyatta-64bit# exit
    exit
    vyatta@vyatta-64bit:~$ show ip route
    Codes: K - kernel route, C - connected, S - static, R - RIP, O - OSPF,
           I - ISIS, B - BGP, > - selected route, * - FIB route

    S>* 0.0.0.0/0 [210/0] via 172.16.1.1, eth0
    B>* 10.1.0.0/16 [20/100] via 169.254.26.185, vti0, 2d06h25m
    S>* 27.0.3.0/24 [1/0] via 172.16.1.1, eth0
    C>* 127.0.0.0/8 is directly connected, lo
    C>* 169.254.26.184/30 is directly connected, vti0
    S>* 169.254.169.254/32 [1/0] is directly connected, eth0
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
Vyattaに設定するIPSec情報をマネージメントコンソールからダウンロードします。
Vyattaの場合、ベンダーは”Vyatta”を選択します。
![VPN設定ダウンロード](https://raw.githubusercontent.com/Noppy/AWS-sidebyside-vpn-by-vyos/master/Document/download_VPN_configuration.png)

(7)設定ファイルの修正(VyOSのインスタンスIP修正)

ダウンロードした設定ファイルのうち、検証では IPSec Tunnel #1のみ利用します。下記を修正します。
+ ＃1 IKE設定(IPSecプロトコルの一つ。秘密鍵情報の交換用プロトコル)
    + set vpn ipsec ike-group AWS proposal 1 encryption 'aes128': encriptionを、`aes128`→`aes256`に変更
    + set vpn ipsec site-to-site peer 13.114.183.29 local-address '13.231.208.95': local-addressを、PublicIPの`13.231.208.95`からVyattaのPrivateIPの`172.16.1.200(適時確認)`に変更
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

(8)vyattaへのIPSec設定(ダウンロードした定義ファイルの修正版を流し込み、commit, saveで設定を記録する)
```shell
$ configure

ダウンロードして修正した設定ファイルの
「#1: Internet Key Exchange (IKE) Configuration」〜「 #4: Border Gateway Protocol (BGP) Configuration 」までの setコマンドを全て流す

# commit
# save
# exit
exit
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
