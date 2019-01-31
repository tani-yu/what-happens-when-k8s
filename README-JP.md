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



### API groups and version negotiation



### Client auth



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

