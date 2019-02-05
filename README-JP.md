# What happens when ... Kubernetes edition!

```bash
kubectl run --image=nginx --replicas=3
```

## contents

1. [kubectl](#kubectl)
   - [validation and generators](#validation-and-generators)
   - [API groups and version negotiation](#api-groups-and-version-negotiation)
   - [client-side auth](#client-auth)
1. [kube-apiserver](#kube-apiserver)
   - [authentication](#authentication)
   - [authorization](#authorization)
   - [admission control](#admission-control)
1. [etcd](#etcd)
1. [initializers](#initializers)
1. [control loops](#control-loops)
   - [deployments controller](#deployments-controller)
   - [replicasets controller](#replicasets-controller)
   - [informers](#informers)
   - [scheduler](#scheduler)
1. [kubelet](#kubelet)
   - [pod sync](#pod-sync)
   - [CRI and pause containers](#cri-and-pause-containers)
   - [CNI and pod networking](#cni-and-pod-networking)
   - [inter-host networking](#inter-host-networking)
   - [container startup](#container-startup)
1. [wrap-up](#wrap-up)

## kubectl

### Validation and generators

はい、それでは始めましょう。ターミナルでエンターキーを押しました。何が起きるのでしょうか？

kubectl が最初に行うことはクライアントサイドの検証です。これにより、必ず失敗するリクエスト(例：サポートされていないリソースの作成や、[不正なイメージ](https://github.com/kubernetes/kubernetes/blob/9a480667493f6275c22cc9cd0f69fb0c75ef3579/pkg/kubectl/cmd/run.go#L251)の使用)は即座に失敗し、kube-apiserver へリクエストが行われません。このため不要な負荷が軽減され、システムのパフォーマンスが向上します。

検証後、kubectl は kube-apiserver へ送信する HTTP リクエストの組み立てを開始します。全ての Kubernetes へのアクセスや状態の変更の試みは API サーバを介して行われ、API サーバは etcd と通信を行います。kubectl クライアントも同様です。HTTPリクエストを組み立てるために kubernetes は[ジェネレータ](https://kubernetes.io/docs/reference/kubectl/conventions/#generators)と呼ばれるものを使用します。ジェネレータとはシリアライズを行う抽象概念です。

分かりづらいかもしれませんが、実は kubectl run で Deployment だけでなく複数のリソースタイプを指定できます。これを機能させるために、kubectl は ジェネレータ名が --generator で明示的に指定されていない限り、リソースタイプを[推測します](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L231-L257)。

例えば、--restart-policy=Always を設定したリソースは Deployments とみなされ、--restart-policy=Never を設定したリソースは Pods とみなされます。kubectl は他のアクション、例えば、コマンドの記録(ロールアウトや監査のため)、などをトリガーする必要があるか判断します。また、このコマンドが単にドライランかどうか(--dry-run フラグの指定)も判断します。

Deployment を作成したいということが分かると、DeploymentV1Beta1 ジェネレータを使用して、提供されたパラメータから[ランタイムオブジェクト](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/run.go#L59)を生成します。"ランタイムオブジェクト" はリソースの総称です。


### API groups and version negotiation

話を続ける前に説明すべきこととして、Kubernetes はバージョン管理された API を使っており、それらは API グループに分類されているということです。API グループは、似たリソースをカテゴライズしてより簡単に推測できるようにすることを目的にしています。それはまた、単一のモノリシックな API に対する良い代替手段を提供します。Deployment の API グループは app と呼ばれ、最新バージョンは v1 です。Deployment のマニフェストの上部に、apiVersion: apps/v1 と記載する必要があるのはこのためです。

いずれにしても、kuberctl がランタイムオブジェクトを生成した後、[適切な API グループとバージョンを見つけ](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L580-L597)、その後、そのリソースの様々な REST セマンティクスを認識する[バージョン化されたクライアントを組み立てます](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L598)。この検出ステージはバージョンネゴシエーションと呼ばれ、全ての利用可能な API グループを取得するために、kubectl がリモート API 上の /api パスをスキャンすることを含みます。kube-apiserver はそのスキーマドキュメントをこのパスで公開しているため、クライアントは容易にディスカバリを行うことができます。

パフォーマンスを向上させるために、kubectl はその OpenAPI スキーマを ~/.kube/cache/discovery ディレクトリに[キャッシュします](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/util/factory_client_access.go#L117)。もし、API ディスカバリの動作を実際に確認したいのであれば、そのディレクトリを削除し、-v フラグを最大にして実行してください。API バージョンを見つけようとする全てのリクエストを確認することができます。
たくさん出力されます！

最後のステップは、実際にリクエストを[送信](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L628)することです。それが実行され、成功したレスポンスを受け取ると、kubectl は[希望したアウトプットフォーマット](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L403-L407)で成功のメッセージを出力します。

### Client auth

前のステップの説明していないことの一つに、クライアント認証があります、(これは HTTP リクエストが送信される前に処理されます)、では、それを今見てみましょう。

リクエストを正常に送信するために、kubectl は認証できる必要があります。ユーザの資格情報はほとんどの場合、ディスク上のkubeconfigファイルに格納されています。しかし、そのファイルは別の場所に格納することも可能です。そのファイルを見つけるために kubectl は以下のことを行います:

- もし、--kubeconfig フラグが提供されている場合は、それを使います。
- もし、$KUBECONFIG 環境変数が定義されている場合は、それを使います。
- そうでなければ、~/.kube のような[予測可能なホームディレクトリ](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L403-L407)の中を調べて、最初に見つかったファイルを使用します。

そのファイルをパースした後、使用する現在のコンテキスト、指し示す現在のクラスタ、そして、現在のユーザに関連付けられている認証情報を決定します。もしユーザがフラグで指定された値として提供されていた場合(たとえば --username)これらは優先され、kubeconfig で指定された値をオーバーライドします。この情報を取得すると、kubectl はクライアントの設定を追加し、それにより HTTP リクエストを適切に修飾できるようになります:

- x509 証明書は [tls.TLSConfig](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89) を利用して送信されます。(これはルートCAも含みます)
- bearer トークンは "Authorization" HTTP ヘッダで[送信されます](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314)
- ユーザ名とパスワードは HTTP ベーシック認証を通して[送信されます](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223)
- OpenID 認証プロセスはあらかじめユーザーによって手動で処理され、bearerトークンのように送信されるトークンを生成します

## kube-apiserver

### Authentication



### Authorization



### Admission control



## etcd



## Initializers



```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```



## Control loops

### Deployments controller



### ReplicaSets controller



### Informers



### Scheduler



## kubelet 

### Pod sync



### CRI and pause containers



### CNI and pod networking



```yaml
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```



### Inter-host networking



### Container startup



## Wrap-up

