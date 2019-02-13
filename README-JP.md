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
pauseコンテナの２つ目の役割はPID ネームスペースを関連付けることです。これらのnamespaceのタイプは木の階層構造を形成し、 トップの"init"プロセスが死んだプロセスの ”刈り取り” をする責任を持ちます。どのように動くかの詳細は、[great blog post](https://www.ianlewis.org/en/almighty-pause-container)をチェックして下さい。pauseコンテナが作られた後、ディスクにチェックポイントされ動き始めます。

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

