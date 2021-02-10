## テンプレートをチェックする
```
aws cloudformation validate-template --template-body file://cf-k8s.yml
```

## スタックを作成する
```
aws cloudformation create-stack --stack-name cf-k8s --region ap-northeast-1 --template-body file://cf-k8s.yml \
--parameters ParameterKey=KeyName,ParameterValue=my-key \
ParameterKey=MyIP,ParameterValue=200.200.200.200/32 \
ParameterKey=GssnIP,ParameterValue=100.100.100.100/32 \
--capabilities CAPABILITY_NAMED_IAM \
```

## スタックを更新する
```
aws cloudformation update-stack --stack-name cf-k8s --template-body file://cf-k8s.yml --parameters ParameterKey=KeyName,ParameterValue=my-key ParameterKey=MyIP,ParameterValue=200.200.200.200/32 ParameterKey=GssnIP,ParameterValue=100.100.100.100/32 --capabilities CAPABILITY_NAMED_IAM
```

## スタック情報を表示する
```
aws cloudformation describe-stacks --stack-name cf-k8s
```

## スタックを削除する
```
aws cloudformation delete-stack --stack-name cf-k8s
```
