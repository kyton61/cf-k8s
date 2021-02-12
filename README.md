## 全体構成図
![k8s_cluster](https://user-images.githubusercontent.com/64095272/107789467-f94b9680-6d94-11eb-994d-d66bac746156.png)


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

## スタック情報を表示する
```
aws cloudformation describe-stacks --stack-name cf-k8s
```

## k8s master nodeの設定
```
sudo su -

kubeadm init \
--control-plane-endpoint="10.5.10.40:6443" \
--apiserver-advertise-address=10.5.10.11 \
--pod-network-cidr=10.5.12.0/24 \
--service-cidr=10.5.13.0/24 \
--upload-certs | tee -a /etc/kubernetes/kubeadm-init.result
```
configファイルのコピー
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
calioでCNIの設定
```
curl https://docs.projectcalico.org/v3.13/manifests/calico.yaml -o calico-v3.13.yaml
kubectl apply -f calico-v3.13.yaml
```
以下のコマンドでmaster nodeがReady状態であることを確認
```
kubectl get node

```

## k8s master node secondの設定
master node 1台目をkubeadm initコマンドで設定した際の実行結果から
kubeadm joinコマンドを生成する
```
tail -20 /etc/kubernetes/kubeadm-init.result
```
で以下のようなコマンド内容をコピーする
```
kubeadm join 10.5.10.40:6443 --token zu0mm6.afgzkqp4lidb6uvv \
    --discovery-token-ca-cert-hash sha256:1625ed445979817c95f77cb528dba34167a2c04b1c49e900a57f04ed0cdc0a5f \
    --control-plane --certificate-key b0d9e6a96454e0eaa90a00543aa0c5daff5bc2af37f86529bc068156393d1743
```
このとき、以下のコマンドもオプションに追加する
```
--apiserver-advertise-address="10.5.10.12"
```
結果的に、以下のコマンドを実行する
```
kubeadm join 10.5.10.40:6443 --token zu0mm6.afgzkqp4lidb6uvv \
--discovery-token-ca-cert-hash sha256:1625ed445979817c95f77cb528dba34167a2c04b1c49e900a57f04ed0cdc0a5f \
--control-plane --certificate-key b0d9e6a96454e0eaa90a00543aa0c5daff5bc2af37f86529bc068156393d1743 \
--apiserver-advertise-address="10.5.10.12"
```
configファイルのコピー
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```

## k8s master node thirdの設定
apiserver-advertise-addressは自分自身のipアドレスを指定すること
```
kubeadm join 10.5.10.40:6443 --token zu0mm6.afgzkqp4lidb6uvv \
--discovery-token-ca-cert-hash sha256:1625ed445979817c95f77cb528dba34167a2c04b1c49e900a57f04ed0cdc0a5f \
--control-plane --certificate-key b0d9e6a96454e0eaa90a00543aa0c5daff5bc2af37f86529bc068156393d1743 \
--apiserver-advertise-address="10.5.10.13"
```
configファイルのコピー
```
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```


## k8s worker nodeの設定
master node 1台目をkubeadm initコマンドで設定した際の実行結果から
kubeadm joinコマンドを生成する.
こちらは、--control-planeオプション、apiserver-advertise-addressオプションは不要
```
kubeadm join 10.5.10.40:6443 --token 9y6p3q.ucxx2rssk826axjx \
--discovery-token-ca-cert-hash sha256:83425ab2c38d842b294b6ae90e1046cc0ed330592d2e92c35cf026b743f901c2
```

## サンプルコンテナのデプロイと動作確認
`TODO:手順作成`
`TODO:HAProxyのアクセスログ確認手順`
`TODO:HAProxy→podへのルート設定確認`


## スタックを削除する
```
aws cloudformation delete-stack --stack-name cf-k8s
```

## 参考
https://kun432.hatenablog.com/entry/kubernetes-by-kubeadm-on-vagrant-01
https://knowledge.sakura.ad.jp/20955/
