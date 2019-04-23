# AWS-sidebyside-vpn-by-vyos



aaa



## VPN構成
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
