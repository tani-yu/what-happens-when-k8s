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



## Initializers

オブジェクトがデータストアに永続化された後、オブジェクトは apiserver によって完全に可視化されることや、一連の [イニシャライザ (intializers)](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers) が実行されるまでスケジュールされることはありません。イニシャライザはリソースタイプに関連付けられたコントローラで、リソースが外の世界から利用可能になる前にそのリソースに対してロジックを実行します。もしリソースタイプにイニシャライザが登録されていない場合は、この初期化ステップは省略され、リソースはすぐに可視化されます。

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

あなたの鋭い目は潜在的な問題を発見したかもしれません。リソースが kube-apiserver によって可視化されていない場合、どのようにしてユーザランドのコントローラはリソースを処理することができるのでしょうか？この問題を回避するために、 kube-apiserver は初期化されていないものを含む _全ての_ オブジェクトを返す `?includeUninitialized` クエリパラメータを公開しています。

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

