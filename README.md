## 全体構成図
![k8s_cluster](https://user-images.githubusercontent.com/64095272/107789467-f94b9680-6d94-11eb-994d-d66bac746156.png)

## Cloudformationでインフラ構築
### 前提条件
・AWS-CLIがインストール済みであること

・管理者権限（or 必要な権限）をもつIAMアクセスキーが設定されていること


### テンプレートをチェックする
```
aws cloudformation validate-template --template-body file://cf-k8s.yml
```


### スタックを作成する
```
aws cloudformation create-stack \
--stack-name cf-k8s \
--region ap-northeast-1 \
--template-body file://cf-k8s.yml \
--parameters ParameterKey=KeyName,ParameterValue=[your key name] \
ParameterKey=MyIP,ParameterValue=[XXX.XXX.XXX.XXX/32] \
--capabilities CAPABILITY_NAMED_IAM
```

[your key name]にはEC2作成時に利用するシークレットキー名を指定してください。

[XXX.XXX.XXX.XXX/32]にはEC2へのSSH元IPアドレスを指定してください。

### スタック情報を表示する
```
watch aws cloudformation describe-stacks --stack-name cf-k8s
```

CREATE-COMPLETEが表示するまで待ちます。


## kubernetes部分
### k8s master nodeの設定
```
sudo su -

kubeadm init \
--control-plane-endpoint="10.5.10.40:6443" \
--apiserver-advertise-address=10.5.10.11 \
--pod-network-cidr=192.168.0.0/16 \
--service-cidr=10.5.12.0/24 \
--upload-certs | tee -a /etc/kubernetes/kubeadm-init.result
```
- `--kubernetes-version`はstableを指定している場合がありますが、バージョンコントロールは自分であるということで厳密にバージョン指定しています。
- `--apiserver-advertise-address`はノードのIPアドレスを指定します。Vagrantでは10.0.2.15というNATインタフェースがあるのですが、これはVMからホストを経由してインターネットに抜けるためのインタフェースであり、ノード間の通信には使えません。kubeadm initするとこのインタフェースがデフォルトでapiserverにbindされてしまうため、明示的に指定しています。
- `--control-plane-endpoint`はmasterの負荷分散代表IPアドレスとなる、haproxyのIPとポートを指定します。
- `--pod-network-cidr`は、Calicoのデフォルトである192.168.0.0/16を指定しています。これがpodに払い出されるIPアドレスになります。
- `--upload-certs`で、2台め以降のmasterをjoinさせる場合に必要となる証明書をsecretに保存します。これにより、証明書の受け渡しが不要になります。出力結果が後で必要になるのでteeでファイルに出力しています。

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

### k8s master node secondの設定
master node 1台目をkubeadm initコマンドで設定した際の実行結果からkubeadm joinコマンドを生成する

```
tail -20 /etc/kubernetes/kubeadm-init.result
```

で以下のようなコマンド内容をコピーする

※tokenやhash値はコマンド実行毎に異なります

```
kubeadm join 10.5.10.40:6443 --token zu0mm6.afgzkqp4lidb6uvv \
    --discovery-token-ca-cert-hash sha256:1625ed445979817c95f77cb528dba34167a2c04b1c49e900a57f04ed0cdc0a5f \
    --control-plane --certificate-key b0d9e6a96454e0eaa90a00543aa0c5daff5bc2af37f86529bc068156393d1743
```

このとき、以下のコマンドもオプションに追加する

※マスターノード自分自身のアドレスを指定する

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

### k8s master node thirdの設定
こちらも同様にapiserver-advertise-addressは自分自身のipアドレスを指定すること

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


### k8s worker nodeの設定
master node 1台目をkubeadm initコマンドで設定した際の実行結果からkubeadm joinコマンドを生成する.

こちらは、--control-planeオプション、apiserver-advertise-addressオプションは不要

```
kubeadm join 10.5.10.40:6443 --token 9y6p3q.ucxx2rssk826axjx \
--discovery-token-ca-cert-hash sha256:83425ab2c38d842b294b6ae90e1046cc0ed330592d2e92c35cf026b743f901c2
```

### サンプルコンテナのデプロイと動作確認
[kubernetesドキュメントのサンプルアプリケーション](https://kubernetes.io/ja/docs/tasks/access-application-cluster/service-access-application-cluster/)を参考に動作確認をしていきます

1. クラスタでHello Worldアプリケーションを稼働させます: 上記のファイルを使用し、アプリケーションのDeploymentを作成します:

```
kubectl apply -f https://k8s.io/examples/service/access/hello-application.yaml
```

このコマンドはDeploymentオブジェクトとそれに紐付くReplicaSetオブジェクトを作成します。ReplicaSetは、Hello Worldアプリケーションが稼働している2つのPodから構成されます。

2. Deploymentの情報を表示します:

```
kubectl get deployments hello-world
kubectl describe deployments hello-world
```

3. ReplicaSetオブジェクトの情報を表示します:

```
kubectl get replicasets
kubectl describe replicasets
```

4. Deploymentを公開するServiceオブジェクトを作成します:

```
kubectl expose deployment hello-world --type=NodePort --name=example-service

```
5. Serviceに関する情報を表示します:
```
kubectl describe services example-service
```

出力例は以下の通りです:

```
Name:                   example-service
Namespace:              default
Labels:                 run=load-balancer-example
Annotations:            <none>
Selector:               run=load-balancer-example
Type:                   NodePort
IP:                     10.32.0.16
Port:                   <unset> 8080/TCP
TargetPort:             8080/TCP
NodePort:               <unset> 31496/TCP
Endpoints:              10.200.1.4:8080,10.200.2.5:8080
Session Affinity:       None
Events:                 <none>
```

NodePortの値を記録しておきます。上記の例では、31496です。

6. Hello Worldアプリーションが稼働しているPodを表示します:
```
kubectl get pods --selector="run=load-balancer-example" --output=wide
```

出力例は以下の通りです:

```
NAME                           READY   STATUS    ...  IP           NODE
hello-world-2895499144-bsbk5   1/1     Running   ...  10.200.1.4   worker1
hello-world-2895499144-m1pwt   1/1     Running   ...  10.200.2.5   worker2
```

7. Hello World podが稼働するNodeのうち、いずれか1つのパブリックIPアドレスを確認します。

8. Hello World applicationにアクセスするために、Nodeのアドレスとポート番号を使用します:

```
curl http://<public-node-ip>:<node-port>

```

ここで <public-node-ip> はNodeのパブリックIPアドレス、 <node-port> はNodePort Serviceのポート番号の値を表しています。 リクエストが成功すると、下記のメッセージが表示されます:

```
Hello Kubernetes!
```

## 後片付け
### スタックを削除する
```
aws cloudformation delete-stack --stack-name cf-k8s
```

必ず使い終わったらEC2を削除しましょう！

## 参考にさせていただいたサイト
https://kun432.hatenablog.com/entry/kubernetes-by-kubeadm-on-vagrant-01

https://knowledge.sakura.ad.jp/20955/

https://qiita.com/saka1_p/items/3634ba70f9ecd74b0860

https://kubernetes.io/ja/docs/tasks/access-application-cluster/service-access-application-cluster/
