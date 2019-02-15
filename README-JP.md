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

リクエストが送られて、次に何が起こるでしょうか。ここで kube-apiserver が登場します。すでに説明したように、kube-apiserver は永続化そしてクラスタの状態を取得するためのシステムコンポーネントとクライアントの主要なインターフェースです。その機能を実行するには、誰が送信者か検証できる必要があります。この過程は認証と呼ばれます。

apiserver はどのようにリクエストを認証するでしょうか。サーバーが最初に起動する時、ユーザーに与えられたすべての[CLIフラグ](https://kubernetes.io/docs/admin/kube-apiserver/)を確認し、適切な認証方式のリストを組み立てます。例を見てみましょう。`--client-ca-file`が渡された場合は、x509認証方式を追加し、`--token-auth-file`が与えられた事を確認した時、token認証方式をリストに追加します。リクエストを受信する度に、[1つでも成功するまで認証方式チェーンを通して実行されます](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54):

- [x509 ハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60) はHTTPリクエストがCAルート証明書によって署名されたTLS鍵でエンコードされていることを検証します
- [bearer トークンハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38) は提供された(HTTP Authorizationヘッダーに指定されている)トークンが`--token-auth-file`で指定されたディスク上のファイルに存在するか検証します
- [basicauth ハンドラー](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37) はHTTP Basic認証の資格情報が自身のローカル状態と同様に一致するか検証します

_すべての_ 認証方式が失敗すると、[リクエストは失敗](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71)して集約されたエラーが返却されます。認証が成功すると`Authorization`ヘッダがリクエストから取り除かれ、リクエストのコンテキストに[ユーザー情報が追加されます](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75)。これにより将来の段階(認可およびアドミッションコントローラーなど)で、以前に確立されたユーザーのIDにアクセスできるようになります。1つの認証方式が成功すると、リクエストは続行します。

### Authorization

さて、リクエストは送信されて、kube-apiserverは我々が我々であると言ったことを検証することに成功しました。なんて安心なんでしょう。しかしながら、まだ終わっていません。我々であると言ったのは我々かもしれませんが、操作を実行する権限を持っているでしょうか。結局の所、同一性と権限は同じ事ではありません。続けるためには、kube-apiserverは我々を認可する必要があります。

kube-apiserverが認可を扱う方法は認証ととても似ています。フラグの入力に基づいて、すべての受信リクエスト毎に認可方式を組み立てます。_すべての_ 認可方式がリクエストを拒否した場合、リクエストの結果として`Forbidden`が応答され、[それ以降は処理されません](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60)。

v1.8に含まれている認可方式の一例です:

- [webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143), クラスタ外のHTTP(S)サービスと対話します
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223), 静的ファイルで定義されているポリシーを適用します
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43), 管理者がk8sリソースとして追加したRBACロールを適用します
- [Node](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67), ノードクライアントすなわちkubeletは、自分自身にホストされているリソースのみアクセスできる事を保証します

それぞれの`Authorize`メソッドを見てどのように動くか確認してみてください。

### Admission control

さて、この時点で我々はkube-apiserverに認証され認可されました。なにが残っているでしょうか。kube-apiserverの視点では、誰であるか信じ続行することを許可していますが、Kubernetesではシステムの他の部分が、何が起こるべきで何が起こるべきでないかについて、強い判断を持ちます。ここで[アドミッションコントローラー](https://kubernetes.io/docs/admin/admission-controllers/#what-are-they)が登場します。

認可はユーザーが権限を持っているかどうかについて答える事に焦点を当てていますが、一方でアドミッションコントローラーはリクエストを傍受してクラスタのより広い期待値とルールにマッチするよう保証します。これらはオブジェクトがetcdで永続化される前の最後の砦です。そのため操作が予期せぬ結果や悪影響を生じないよう、残りのシステム検査をまとめて行います。

アドミッションコントローラーの仕組みは認証と認可の仕組みに似ていますが、1つ異なる所があります。認証方式、認可方式のチェーンと違い、1つのアドミッションコントローラーが失敗すると、チェーン全体が壊れリクエストは失敗します。

アドミッションコントローラーの設計に関して本当にクールなのは、拡張性の促進に焦点を当てていることです。各コントローラーは[`plugin/pkg/admission`ディレクトリ](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission)にプラグインとして格納されており、小さなインターフェースを満たすように作られています。それぞれがメインのkubernetesバイナリ自身にコンパイルされます。

アドミッションコントローラーは通常、リソース管理、セキュリティ、デフォルトおよび参照整合性に分類されます。これらは、ちょうどリソース管理の面倒を見ているアドミッションコントローラーの一例です:

- `InitialResources`は過去の使用量に基づいて、コンテナのリソースにデフォルトのリソース制限を設定します。
- `LimitRanger`はコンテナのrequestsとlimitsにデフォルトを設定するか、特定のリソースの上限を適用します。(2GB以下のメモリ、デフォルトは512MB)
- `ResourceQuota`はネームスペース内のオブジェクト(pods, rc, service load balancers)の数もしくは消費リソースの総計(cpu, memory, disk)を算出して拒否します。

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

この段階で Deployment のレコードは etcd 内に存在し、全ての初期化処理は完了しています。次の段階では Kubernetes にとって重要なリソーストポロジーの設定をします。よく考えてみれば、Deployment はただの ReplicaSet の集まりであり、ReplicaSet は Pod の集まりです。では、Kubernetes はどのようにして 1 つの HTTP リクエストからその階層構造を作り出すのでしょうか？これに関しては Kubernetes にビルトインされているコントローラーが受け持っています。

Kubernetes はシステム全体に強力な「コントローラー」を作りました。コントローラーは Kubernetes の状態を望んだ状態に調整するためのスクリプトであり、非同期で実行されます。それぞれのコントローラーは小さな担当を持ち、`kube-controller-manager` コンポーネントによって並列実行されます。それでは、そのコントローラーである最初の 1 つとして、Deployment コントローラーについて紹介しましょう。

Deployment レコードが etcd に保存されて初期化が終わると、kube-apiserver を介して見えるようになります。こうして新しいリソースが利用可能になったとき、Deployment コントローラーがそれを検出します。このように Deployment コントローラーは Deployment レコードに変化がないか監視する役目があります。このような場合には、コントローラーはインフォーマーを介して、イベントを作成する [とあるコールバックを登録](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122) します。(インフォーマーについては下で解説しています)

このハンドラーは Deloyment が最初に利用可能になったときに実行され、その Deployment オブジェクトが [内部のワークキューに追加されたとき](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L170) に開始されます。その頃には、そのオブジェクトを処理する時間的余裕ができていて、コントローラーは Deployment を [調べて](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572) 、ReplicaSet と Pod が紐付けられていないことを [認識](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633) します。それはラベルセレクターで kube-apiserver に問い合わせることで調べています。注目すべき点はこのように同期するプロセスが状態にとらわれないという点です。新規のレコードの調整は既存のレコードの更新時と同じ方法で行われます。

何も紐付けられていないことを認識した後、その状態を直すべく [スケーリングプロセス](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385) が働きます。これは ReplicaSet のリソースをロールアウト (作成) してラベルセレクターを割り当て、そしてリビジョン番号 1 を付与することによって実現されています。その ReplicaSet の PodSpec は他の関係するメタデータと同様に Deployment のマニフェストからコピーされます。場合によっては、その Deployment レコードも一緒にアップデートする必要があります。(例えば Progress Deadline が設定されている場合など)

それから、その [状態はアップデート](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70) され、Deployment が望んだ状態とマッチするまで再度同じ調整ループに入ります。Deployment コントローラーは ReplicaSet を作成することに関してのみ気にしているため、この調整ステージは次のコントローラー、ReplicaSet コントローラーに続きます。

### ReplicaSets controller

前のステップでは、Deployment コントローラーは Deployment の最初の ReplicaSet を作成しましたが Pod がまだありません。ここで ReplicaSet コントローラーの登場です。このコントローラーの仕事は ReplicaSet のライフサイクルと、依存したリソース (Pod) を監視するものです。他のほとんどのコントローラーと同様、あるイベントのハンドラーがトリガーされることにより実行されます。

その気になるイベントとは「作成」です。Deployment コントローラーにより ReplicaSet が作成されたとき、その ReplicaSet コントローラーは新規の ReplicaSet の [状態を調べ](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583) 、今の状態と求められている状態の間に偏りがあることを認識します。それから、ReplicaSet に属す [Pod の数を増やす](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460) ことにより状態を調整しようとします。この増加処理は ReplicaSet のバーストカウント (親の Deployment から引き継いだもの) が常に一致するように慎重に作成されます。

Pod の作成操作も [バッチ的処理](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460) であり、`SlowStartInitialBatchSize` の数の作成から始まり、一種の「スロースタート」で作成が成功する度に 2 倍されていきます。これは Pod の起動に大量に失敗したとき (リソースクオータに引っかかったときなど)、不要な HTTP リクエストにより kube-apiserver が溢れてしまうリスクを軽減するために行われています。この仕組みにより、起動に失敗した場合は、他のシステムコンポーネントに対する影響を最小に抑えつつグレースフルに失敗するでしょう。

Kubernetes は Owner References (親の ID を参照する子のフィールド) を通してオブジェクトの階層構造を強制します。これはコントローラーによって管理されているリソースが削除されるとすぐに子がガベージコレクト (cascading deletion) されることを保証するだけでなく、親リソースが子を取り合わないようにするための効果的な方法も提供します (2 つの親リソースが同じ子リソースを所有する事態を想像してみてください！)

Owner Reference の設計がもたらすもう 1 つの小さな利点はステートフルである、ということです。リソーストポロジーがコントローラーから独立しているため、仮に何らかのコントローラーが再起動したとしても、そのダウンタイムがシステムワイドに影響を及ぼすことはないでしょう。この分離性に対する焦点はコントローラー自体の設計にも浸透しています。コントローラーは明示的に所有していないリソースを操作すべきではありません。その代わりにコントローラーはその所有権アサーションに対して選択的、非干渉、かつ非共有であるべきです。

とにかく Owner Reference に戻ります！たまに次のような時に普段のシステムに「孤立した」リソースが発生することがあります。

1. 親は削除されるが子が削除されない
1. ガベージコレクションのポリシーが子の削除を禁止している

これが発生すると、コントローラーは孤立したリソースを新しい親リソースに適用します。多数の親リソースが子を適用するために取り合うことがありますが、成功するのは 1 つの親のみです。(他の親はバリデーションエラーを受け取ることになります)

### Informers

もう気づいているかも知れませんが、RBAC Authorizer や Deployment コントローラーなどの一部のコントローラーは動作するためにクラスターの状態を持ってくる必要があります。RBAC Authorizer の例に戻ると、我々はリクエストが来たときに Authenticator が初期のユーザー状態の表現をあとの処理で使用するために保存するということを知っています。それから RBAC Authorizer はこれを使用してそのユーザーに関連する全ての Role と Role Binding を etcd 内から取得します。コントローラーはどのようにしてそのようなリソースにアクセスや修正をすることになっているのでしょうか？これは一般的なユースケースであり、Kubernetes 内のインフォーマーによって解決されていることがわかります。

インフォーマーはコントローラーにストレージイベントをサブスクライブさせ、それらに関係するリソースのリストを簡単に取得するパターンです。うまく動くように抽象化を提供することの他に、キャッシング (キャッシングは不要な kube-apiserver のコネクションを減らすのと、サーバー側とコントローラー側での多重のシリアライゼーションを減らすため重要です) などの多くの仕組みも管理しています。この設計を使用することで、他のコントローラーのことを気にすること無くスレッドセーフな方法で通信できるようになります。

インフォーマーがコントローラーに関してどのように機能するかについての詳細は、この [ブログ](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores) をチェックしてください。

### Scheduler

すべてのコントローラーが動いたあと、etcd 内に保存された Deployment と ReplicaSet と 3 つの Pod が kube-apiserver を通して利用可能になりました。しかしながら我々の Pod はまだ Node にスケジュールされていないため `Pending` 状態で止まっています。これらを解決する最後のコントローラーがスケジューラーです。

スケジューラーはコントロールプレーンのコンポーネントとして単独で動作し、他のコントローラー同様にイベントの監視と状態調整の試行といった操作を行います。今回の場合は PodSpec の `NodeName` が空になっている [Pod をフィルタ](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190) し、その Pod が起動するのにふさわしい Node を探そうとします。

適切な Pod を探すために、とあるスケジューリングアルゴリズムが使用されます。デフォルトのスケジューリングアルゴリズムの流れは下記のようになっています:

1. スケジューラーが起動すると [デフォルトの Predicate チェインが登録](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81) されます。これら Predicate は評価すると Pod を動かすのに適している [Node をフィルタ](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117) する効率的な関数です。例えば PodSpec が明示的に CPU または RAM リソースを要求しており、かつとある Node が容量不足により要求を満たせない場合は、その Node を Pod のスケジューリング候補から除外します。(リソース容量は "_合計容量_ - _現在動いているコンテナの総リクエスト容量_" により求められます)

1. 適切な Node がすべて選択されると、適切度をランク付けするために一連の [優先度関数](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360) がそれらの Node に対して実行されます。例えば、システム上でワークロードを拡散させるためには、他の Node よりリソース要求が少ない Node が好まれます (これは実行中のワークロードが少ないことを示すためです) 。それらの関数を実行することにより、各 Node に数値でランク付けを行います。それから、高くランク付けされた Node がスケジューリングに採用されます。

そのアルゴリズムが Node を見つけると、スケジューラーは Pod の Name、UID とマッチする [Binding オブジェクトを生成](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342) します。そして、そのオブジェクトの ObjectReference フィールドは選択された Node の名前を保持しています。それから apiserver に POST リクエストを [送信](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095) します。

kube-apiserver がその Binding オブジェクトを受け取ると、レジストリーはそのオブジェクトをデシリアライズして、Pod オブジェクトの以下のフィールドを更新します: NodeName を ObjectReference 内の NodeName に [設定](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L170) し、 [関連するアノテーションを追加](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L174-L176) し、`PodScheduled` ステータスを `True` に [設定](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177-L180) します。

スケジューラーが Pod を Node にスケジュールすると、その Node 上の kubelet がデプロイを開始するために引き継ぎます。面白いですね！

**余白注記: スケジューラーのカスタマイズ:** 面白いことに Predicate 関数と優先度関数は拡張可能で、`--policy-config-file` フラグを使用することにより定義することが可能です。これによりある程度の柔軟性がもたらされます。また、管理者は独立した Deployment でカスタムスケジューラー (カスタム処理ロジックが組み込まれたコントローラー) を実行することも可能です。PodSpec が `schedulerName` が含まれている場合、Kubernetes はその Pod のスケジューリングをその名前で登録されているスケジューラーに引き継ぎます。

## kubelet 

### Pod sync
これまでのメインコントローラのループの流れについてまとめます。HTTPリクエストは認証、認可、アドミッションコントロールのステージを通って来ます。Deployment, Replicaset, 3つのPodのリソースはetcdに保存されます。一連のinitializerが実行されます。最後にPodは適切なノードに配置されるようにスケジュールされます。私達が想像している状態はetcd内に保存されています。次のステップではワーカーノードをまたいで分散させます。これは、kubernetesのような分散システムではポイントとなります！これはkubeletと呼ばれるコンポーネントを通して実行されます。さあ流れを追ってみましょう！

kubeletはkubernetesクラスターすべてのノードで動いるエージェントで、Podのライフルサイクルの責任を負っています。これは、Pod（kubernetesの概念）の抽象化と構成要素であるコンテナの間のすべての変換ロジックを取り扱うことを意味します。また、ボリュームのマウント、コンテナのlogging、ガベージコレクションとその他の多くの重要なことに関連するすべてのロジックを取り扱っています。

kubeletについて簡潔に説明するとコントローラのようなものです！20秒ごと（この値は設定可能）にkube-apiserverにPodを問い合わせ、 `NodeName` がkubeletの動いているノードの名前と一致するものをフィルタリングします。kubeletがリストを持っていると、自身の内部キャッシュと比較することによって新たな追加を検出し、不一致が存在すれば同期を始めます。その同期のプロセスはがどのようなものか見てみましょう。


1. Podが作成されると、PodのレイテンシをトラッキングするためのPrometheusで使われているいくつかの[スタートアップメトリクスを登録します](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519)。
1. 次に、Podの現在のフェーズの状態を表す[PodStatusオブジェクト](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287)を生成します。Podのフェーズは、Podがライフサイクルのどの状態にあるかの概要です。例として `Pending` , `Running` , `Succeeded` , `Failed` , `Unknown` などがあります。この状態を生成するのはとても複雑なので、正確に何が起こるか見てみます。
    - 初めに `PodSyncHandlers` のチェーンが順番に実行されます。それぞれのハンドラはPodがまだノード上にあるべきかをチェックします。もしPodがもうそこに属していないと判断された場合、Podのフェーズは [`PodFailed` に変わり](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297)、最終的にノードから削除されます。これらの例としては `activeDeadlineSecounds` を超えた（ジョブで使用された）後にPodを削除することが挙げられます。
    - 次に、Podのフェーズはinitコンテナと実際のコンテナのステータスによって決まります。コンテナがまだ起動されてない場合は [waiting](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244) として分類されます。`waiting`コンテナがいるPodは [Pending](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261) フェーズになります。
    - 最後に、Podの状態はそのコンテナの状態によって決定されます。コンテナランタイムによって、まだコンテナを作成していないときは [`PodReadyが` Falseに設定されます](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81)。
1. PodStatusが作られた後、その情報はPodステータスマネージャーに送信されます。これはapiserverを通してetcdに非同期で更新しています。
1. 次に、Podが正しいセキュリティ権限を持っているか確認するために、一連のadmissionハンドラーが実行されます。これらには、[AppArmorプロファイルとNO_NEW_PRIVS](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884)の適用が含まれています。この段階で拒否されたポッドは無期限に `pending` 状態となります。
1. もし`cgroups-per-qos` ランタイムフラグが指定されていたら、kubeletはpodのためのcgroupを作成し、リソースパラメータを適用します。これは、Podのサービス品質(QoS)処理を向上させるためです。
1. Pod用のデータディレクトリが作成されます。Podディレクトリ（通常は `/var/run/kubelet/pods/<pod ID>` ）、ボリュームディレクトリ ( `<podDir>/volumes` )、pluginsディレクトリ（ `<podDir>/plugins` ）が含まれます。
1. ボリュームマネージャはSpec.Volumesで定義されている関連ボリュームを接続して待機します。マウントされているボリュームの種類によっては、長い待ち時間が必要になります。（cloudやNFSボリューム等）
1. `Spec.ImagePullSecret`で定義されたすべてのシークレットは、後でコンテナに注入できるように、apiserverから取得されます。
1. その後、コンテナランタイムはコンテナを実行します。（詳細は後述）

### CRI and pause containers
現時点でセットアップのほとんどが終わり、コンテナを立ち上げる準備ができました。立ち上げるソフトウェアは、コンテナランタイムと呼ばれています（例えば、dockerやrkt）。拡張性を高めるために、kubeletのv1.5.0からCRI(Container Runtime Interface)と呼ばれる概念を使用しています。一言で言うと、CRIはkubeletと特定のランタイム実装の間の抽象化を提供します。 通信はprotocol buffe（早いJSONのようなもの）、gRPC API(Kubernetesの操作に適した)を介して行われます。これは、kubeletとランタイムの間の定義済みの契約を使うことによって、コンテナの実際の詳細実装は、大部分が無関係になるため非常に素晴らしいアイディアです。重要であるのは契約です。これはKubernetesのコアのコードを変更する必要がないので、最小限のオーバーヘッドで新しいランタイムを追加することができます。

ここで、コンテナのデプロイの話に戻りましょう… 最初にpodが開始されたとき、kubeletは[ `RunPodSandbox`リモートプロシジャーコール（RPC）を呼び出します](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51)。 sandboxはコンテナのひとまとまりを表すCRIの用語であり、Kubernetes用語だと、ご想像のとおり、podです。この用語は故意に曖昧になっているので、実際にコンテナを使用できない他のランタイムの意味を失うことはありません。（sandboxがVMになるハイパーバイザーベースのランタイムを想像してください）

Dockerを使うケースでは、ランタイム内でsandboxが”pause”コンテナーを作成させます。pauseコンテナーサーバは、ワークロードコンテナで使われているような多数のトップレベルのリソースをホストするため、Pod内の他のすべてのコンテナの親のように機能します。これらの "resource" は Linux namespacesです（IPC, network,PID）。もしLinuxで動くコンテナに馴染みがないなら、簡単に復習しましょう。Linuxカーネルはnamespaceの概念を持っていて、それはホストOSが専用の一連のリソース(CPUやメモリなど)を作り出すのを許可し、プロセスに提供します。
Cgroupsは、Linuxがリソースを割り与える方法であるため（リソース使用量を監視する警察のような）重要になります。リソースの確保が保証され、強制隔離されたプロセスをホストするために、Dockerはこれらの両方の機能を使用します。詳細は `b0rk` の驚くべき投稿を確認して下さい。 [What even is a Container?](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/)

pauseコンテナはnamespaceのすべてをホストし、子コンテナがそれを共有できる方法を提供します。同じネットワークnamespaceの一部であることによって得られる一つのメリットは、同じPod内のコンテナが`localhost` を使って他のものを参照できることです。 
pauseコンテナの２つ目の役割はPID ネームスペースを関連付けることです。これらのnamespaceのタイプは木の階層構造を形成し、 トップの"init"プロセスが死んだプロセスの ”刈り取り” をする責任を持ちます。どのように動くかの詳細は、[great blog post](https://www.ianlewis.org/en/almighty-pause-container)をチェックして下さい。pauseコンテナが作られた後、ディスクにチェックポイント(保持)され動き始めます。

### CNI and pod networking

さて、Podの土台が出来上がりました。　Pod間通信を可能にする全Namespaceの情報を持つPuuseコンテナです。　しかし、ネットワークはどのように機能しどのように設定されているのでしょうか？

kubeletがPodのネットワークを設定すると、タスクを"CNI"プラグインに委任します。CNIは「Container Network Interface」の略で、Container Runtime Interfaceと同じように動作します。端的に言うと、CNIは、さまざまなネットワークプロバイダがさまざまなネットワーク実装をコンテナに使用できるようにするための抽象概念です。プラグインは登録されており、kubeletはJSONデータ（設定ファイルはここです。　/etc/cni/net.d　）を関連するCNIバイナリ（ここにあります　/opt/cni/bin）にstdinを介してストリーミングすることによってプラグインとやりとりします。これはJSONの設定例です。

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

またCNI_ARGS環境変数を介して、nameやnamespaceなど、podの追加のメタデータも指定します。

この次の話はCNIプラグインに依存していますが、引き続きbridgeCNIプラグインを見てみましょう。

1. プラグインは最初にrootのネットワークNamespaceにローカルのLinuxブリッジを設定し、そのホスト上のすべてのコンテナにサービスを提供します。
2. 次に、Pauseコンテナのネットワーク Namespace にインターフェイス（vethペアの一方の端）を挿入し、もう一方の端をブリッジに接続します。vethペアの概念を捉えるのに最適なのは、これを大きなチューブと捉えることです。一方がコンテナに接続され、もう一方がルートネットワークNamespaceにあり、パケットがその間を通過できるようになります。
3. 次に、PauseコンテナのインターフェイスにIPを割り当て、ルーティングを設定します。これにより、Podに独自のIPアドレスが割り当てられます。IP割り当ては、JSON構成に指定されているIPAMプロバイダーに委任されます。
  - IPAMプラグインは、メインネットワークのプラグインと似ています。バイナリを介して呼び出され、標準化されたインターフェースを持ちます。それぞれがコンテナのインターフェースのIP /サブネットを、ゲートウェイルーティングとともに決定し、この情報をメインプラグインに返す必要があります。最も一般的なIPAMプラグインは`host-local` と呼ばれ、事前定義されたアドレス範囲のセットからIPアドレスを割り当てます。状態をホストファイルシステムのローカルに保存するため、単一ホスト上のIPアドレスの一意性が保証されます。
4. DNSの場合、kubeletはCNIプラグインに内部DNSサーバーのIPアドレスを指定します。これにより、コンテナのresolv.confファイルが適切に設定されます。

この一連のプロセスが完了すると、プラグインはJSONデータをkubeletに返して操作の結果を示します。

### inter-host networking

これまで、コンテナがホストに接続する方法について説明してきましたが、ホストはどのように通信するのでしょうか。異なるマシン上の2つのPodが通信したい場合に必ず発生します。

これは通常、オーバーレイネットワーキングと呼ばれる概念を使用して実現されます。これは、複数のホスト間でルーティングを動的に同期させる方法です。人気のオーバーレイネットワークプロバイダの1つがFlannelです。インストール時のメインの役割は、クラスタ内の複数のノード間にレイヤ3のIPv4ネットワークを提供することです。Flannelはコンテナがホストにネットワーク接続される方法（これはCNIが担っている仕事です）を制御するのではなく、トラフィックが _ホスト間_ で転送される方法を制御します。これを行うには、ホストのサブネットを選択してetcdに登録します。次に、クラスタルートのローカル設定を維持し、発信パケットをUDPデータグラムにカプセル化して、正しいホストに到達できるようにします。詳細については、[CoreOSのドキュメント](https://github.com/coreos/flannel)を確認ください。


### container startup

すべてのネットワーク関連の動作が完了しました。ではあとは何が残っているでしょうか？　そう、コンテナを実際に起動する必要があります。

サンドボックスの初期化が完了してアクティブになると、kubeletはコンテナの作成を開始できます。最初にPodSpecで定義されている[initコンテナを起動](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690) し、次にメインコンテナ自体を起動します。実行するプロセスは以下になります。

1. [コンテナイメージのpull](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90) コンテナイメージをpullします。PodSpecで定義のsecretsはすべてプライベートレジストリで使用されます。

2. [コンテナの作成](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115) CRIを介してコンテナを作成します。これは`ContainerConfig`、親PodSpecから構造（コマンド、画像、ラベル、マウント、デバイス、環境変数などが定義されているもの）を作成してからprotobufsを介してCRIプラグインに送信することによって行われます。Dockerの場合は、ペイロードを逆シリアル化して、デーモンAPIに送信するための独自の構成構造体を生成します。その過程で、コンテナにいくつかのメタデータラベル（コンテナタイプ、ログパス、サンドボックスIDなど）が適用されます。

3. 次に、コンテナーをCPUマネージャーに登録します。これは、UpdateContainerResourcesCRIメソッドを使用してコンテナーをローカルノード上のCPUのセットに割り当てるversion 1.8の新しい機能です。

4. その後、コンテナーが[起動](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135)します。

5. 開始後のコンテナライフサイクルフックが登録されている場合は、それらが実行されます。フックはタイプExec（コンテナー内で特定のコマンドを実行する）またはHTTP（コンテナーエンドポイントに対してHTTP要求を実行する）のいずれかです。PostStartフックの実行に時間がかかりすぎたり、ハングアップしたり、失敗した場合、コンテナは決してrunningステータスに到達しません。


## Wrap-up

