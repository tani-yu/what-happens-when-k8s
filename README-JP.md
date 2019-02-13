# What happens when ... Kubernetes edition!

想像してみてください。私は Kubernetes のクラスタに対して nginx を展開したいと思っています。きっと私は端末にこのように入力するでしょう。

```bash
kubectl run --image=nginx --replicas=3
```

そしてエンターキーを押します。2,3 秒後には 3 つの nginx の Pod がどこかの worker node に広がっていく様子が見えるはずです。まるで魔法のようですね。素晴らしいことです！ でも本当は内部で何が起こっているのでしょうか。

Kubernetes の素晴らしい部分の 1 つは、使い勝手の良い API を通じてインフラに対してワークフローの構築を操作することです。複雑な部分は抽象化されて隠されます。しかし Kubernetes がもたらしてくれるその恩恵を完全に理解する為には、その内部を理解することもまた必要になってきます。
本ガイドは、何が起こっているのかを説明する為に必要ならばソースコードの場所へと移動しつつ、クライアントから kubelet へのリクエスト処理の一部始終をあなたにお見せします。

この文章は常に更新されています。もしあなたが加筆、修正することができる部分がありましたら、コントリビュートを歓迎します！

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

検証後、kubectl は kube-apiserver へ送信する HTTP リクエストの組み立てを開始します。全ての Kubernetes へのアクセスや状態の変更の試みは API サーバを介して行われ、API サーバは etcd と通信を行います。kubectl クライアントも同様です。HTTPリクエストを組み立てるために kubernetes は[ジェネレータ](https://kubernetes.io/docs/reference/kubectl/conventions/#generators)と呼ばれるものを使用します。ジェネレータとはシリアライズを行う抽象概念です。

分かりづらいかもしれませんが、実は kubectl run で Deployment だけでなく複数のリソースタイプを指定できます。これを機能させるために、kubectl は ジェネレータ名が --generator で明示的に指定されていない限り、リソースタイプを[推測します](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/cmd/run.go#L231-L257)。

例えば、--restart-policy=Always を設定したリソースは Deployments とみなされ、--restart-policy=Never を設定したリソースは Pods とみなされます。kubectl は他のアクション、例えば、コマンドの記録(ロールアウトや監査のため)、などをトリガーする必要があるか判断します。また、このコマンドが単にドライランかどうか(--dry-run フラグの指定)も判断します。

Deployment を作成したいということが分かると、DeploymentV1Beta1 ジェネレータを使用して、提供されたパラメータから[ランタイムオブジェクト](https://github.com/kubernetes/kubernetes/blob/7650665059e65b4b22375d1e28da5306536a12fb/pkg/kubectl/run.go#L59)を生成します。"ランタイムオブジェクト" はリソースの総称です。


### API groups and version negotiation

話を続ける前に説明すべきこととして、Kubernetes はバージョン管理された API を使っており、それらは API グループに分類されているということです。API グループは、似たリソースをカテゴライズしてより簡単に推測できるようにすることを目的にしています。それはまた、単一のモノリシックな API に対する良い代替手段を提供します。Deployment の API グループは app と呼ばれ、最新バージョンは v1 です。Deployment のマニフェストの上部に、apiVersion: apps/v1 と記載する必要があるのはこのためです。

いずれにしても、kubectl がランタイムオブジェクトを生成した後、[適切な API グループとバージョンを見つけ](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L580-L597)、その後、そのリソースの様々な REST セマンティクスを認識する[バージョン化されたクライアントを組み立てます](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubectl/cmd/run.go#L598)。この検出ステージはバージョンネゴシエーションと呼ばれ、全ての利用可能な API グループを取得するために、kubectl がリモート API 上の /api パスをスキャンすることを含みます。kube-apiserver はそのスキーマドキュメントをこのパスで公開しているため、クライアントは容易にディスカバリを行うことができます。

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

ここに来るまで、Kubernetesは受け取ったリクエストを入念にチェックし、次に進むように許可しました。次のステップではkube-apiserverはHTTPリクエストをデシリアライズし、ランタイムオブジェクトを構築し（これはkubectlのジェネレーターの逆の処理をやっているようなものです）、データストアに永続化します。詳細を追ってみましょう。

どのようにしてkube-apiseverはリクエストを受け取った際に何をすべきか知るのでしょうか。実際のところ、リクエストが実際に届く”前”に極めて複雑で連続した処理が実行されます。1から始めましょう。バイナリファイルが実行されるときです。

1. `kube-apiserver`のバイナリが実行されたとき、[サーバーチェイン（server chain）が作成](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L119)され、API Aggregationを可能にします。これは基本的にはマルチapiserverををサポートするものです（私達が気にする必要はありません）。
1. このとき、デフォルトの実装のみを提供する[汎用的なapiserverが作られます](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-apiserver/app/server.go#L149)。
1. 生成されたOpenAPIスキーマは[apiserverの設定情報](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149)に配置されます。
1. kube-apiserverはスキーマの中で明示されたすべてのAPIグループに対して、抽象化された汎用的なストレージとして動作するように[ストレージ・プロバイダー（storage provider）](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410)を設定します。これがkube-apiserverがリソースの状態にアクセスまたは変更するときに対話する相手です。
1. kube-apiserverは各APIグループについて、それぞれのグループバージョンの各HTTPルートに[RESTマッピングを登録します](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92)。これでkube-apiserverが各リクエストのマッピングを保持し、マッチするリクエストを見つけた際に正しいロジックに導くことができるようになります。
1. 私達のユースケースでは、[POSTハンドラ](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710)が登録され、最終的に[リソースを生成するハンドラ](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)が紐付きます。

ここまでで、kube-apiserverはどのルートが存在し、リクエストがマッチしたときに、どのハンドラーとどのストレージ・プロバイダーが呼び出されるのかという内部的なマッピングを保持していることとなります。本当に賢いやつですね！さあ、次は私達のHTTPリクエストが飛んでくるところを想像してみましょう。

1. ハンドラチェーンがリクエストのパターン（すなわち登録したルート）にマッチする場合、ルートに登録した[ハンドラーを実行します](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143)。そうでない場合、[パスに基づいたハンドラー](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248)に戻ってきます（これが`/api`を呼んだ際に起こることです）。パス対してどのハンドラーも登録されていない場合、[not found handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254)が呼ばれ、結果は404が返されます。
1. 幸いにも私達には[`createHandler`](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)というルートが登録されています!これが何をするかというと、まず初めにHTTPリクエストをデコードし、APIリソースのバージョンが期待されるJSONであるかどうかといった基本的なバリデーションを行います。
1. 次に監視と最終的な許可の[処理が行われます](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104)。
1. リソースは[指定されたストレージ・プロバイダー](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327)によって[etcdに保存されます](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111) 。通常はキーは`<namespace>/<name>`の形ですが、変更が可能です。
1. 生成時のエラーはすべてキャッチされ、最終的にストレージ・プロバイダーが`get`を呼び出してオブジェクトが作成されたかどうかを確認します。もし追加でファイナライズ処理が必要であれば、生成後に呼び出されるハンドラーとデコレータ（訳注：ミドルウェアに相当する処理）が呼び出されます。
1. HTTPレスポンスが[生成され](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142)、返却されます。

たくさんのステップがありましたね！複雑な処理を詳細まで追っていくことは本当に素晴らしい体験です。なぜならapiserverがどれだけのことをしているかを知ることができるからです。まとめると、今や私達のDeploymentはetcdに存在しています。しかし、断っておきますが、まだそれを実際に見ることはできないのです...

## Initializers

オブジェクトがデータストアに永続化された後、一連の [イニシャライザ (intializers)](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers) の実行が完了するまで、オブジェクトは apiserver によって完全に公開されたりスケジュールされたりすることはありません。イニシャライザはリソースタイプに関連付けられたコントローラで、リソースが外の世界から利用可能になる前にそのリソースに対してロジックを実行します。もしリソースタイプにイニシャライザが登録されていない場合は、この初期化ステップは省略され、リソースはすぐに公開されます。

[多くの素晴らしいブログ記事](https://ahmet.im/blog/initializers/) が指摘したように、イニシャライザは汎用的なブートストラップ操作を実行することを可能にするので強力な機能です。次のような例が挙げられます:

- ポート 80 番の公開や特定のアノテーションを持つプロキシサイドカーコンテナを Pod に注入する
- テスト用の証明書を持つボリュームを特定のネームスペース内の全ての Pod へ注入する
- Secret が 20 文字よりも短い場合 (パスワードなど) 、その作成を防ぐ

`initializerConfiguration` オブジェクトを使うことで、特定のリソースタイプに対してどのイニシャライザを実行するかを宣言することができます。Pod が作成される度にカスタムイニシャライザを実行したいときには、次のようにします:

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

このコンフィグが作成されると、`custom-pod-initializer` という名前が各 Pod の `metadata.initializers.pending` フィールドに追加されます。イニシャライザのコントローラはすでにデプロイされており、新しい Pod を定期的にスキャンしています。Pod の pending フィールドが名前を持っているものを検出すると、イニシャライザはロジックを実行します。この処理が完了した後、イニシャライザは pending リストから名前を削除します。名前がリストの先頭にあるイニシャライザのみがリソースを操作できます。全てのイニシャライザが終了し `pending` フィールドが空になると、オブジェクトは初期化されたものとみなされます。

鋭い観察眼を持つあなたは些細な潜在的な問題を見つけるかもしれません。リソースが kube-apiserver によって公開されていない場合、どのようにしてユーザランドのコントローラはリソースを処理することができるのでしょうか？この問題を回避するために、 kube-apiserver は初期化されていないものを含む _全ての_ オブジェクトを返す `?includeUninitialized` クエリパラメータを公開しています。

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

