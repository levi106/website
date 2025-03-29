---
title: いくつかのNode上でのみPodを実行する
content_type: task
weight: 30
---
<!-- overview -->

このページでは、{{<glossary_tooltip term_id="daemonset" text="DaemonSet">}}の一部として、いくつかの{{<glossary_tooltip term_id="node" text="Node">}}上でのみ{{<glossary_tooltip term_id="pod" text="Pod">}}を実行させる方法について示します。

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}}

## いくつかのNode上でのみPodを実行する

{{<glossary_tooltip term_id="daemonset" text="DaemonSet">}}を実行したいが、デーモンPodを実行する必要があるのはローカルのソリッドステート(SSD)ストレージを備えたNode上のみである場合を想像してください。
例えば、ノードに対してキャッシュサービスを提供するPodがありますが、キャッシュは低レイテンシーのローカルストレージが利用可能な時のみ有用です。

### ステップ1: Nodeにラベルを追加する

SSDを持つNodeに対してラベル`ssd=true`を付与します。

```shell
kubectl label nodes example-node-1 example-node-2 ssd=true
```

### ステップ2: マニフェストを作成する

SSDラベルを持つ{{<glossary_tooltip term_id="node" text="Node">}}上でのみデーモンPodが展開されるように、{{<glossary_tooltip term_id="daemonset" text="DaemonSet">}}を作成しましょう。

次に`nodeSelector`を使用して、`ssd`ラベルに`"true"`の値を持つNodeに対してのみDaemonSetがPodを実行するようにします。

{{% code_sample file="controllers/daemonset-label-selector.yaml" %}}

### ステップ3: DaemonSetを作成する

`kubectl create`または`kubectl apply`を使用してマニフェストからDaemonSetを作成します。

もう一つのNodeに`ssd=true`のラベルをつけましょう。

```shell
kubectl label nodes example-node-3 ssd=true
```

Nodeにラベルをつけると、新しいデーモンPodをNode上で実行するために、自動的にコントロールプレーン(特にDaemonSetコントローラー)がトリガーされます。

```shell
kubectl get pods -o wide
```
次のような出力となります:

```
NAME                              READY     STATUS    RESTARTS   AGE    IP      NODE
<daemonset-name><some-hash-01>    1/1       Running   0          13s    .....   example-node-1
<daemonset-name><some-hash-02>    1/1       Running   0          13s    .....   example-node-2
<daemonset-name><some-hash-03>    1/1       Running   0          5s     .....   example-node-3
```