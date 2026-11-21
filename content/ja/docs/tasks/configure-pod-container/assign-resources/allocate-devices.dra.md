---
title: DRAを使用してワークロードにデバイスを割り当てる
content_type: task
min-kubernetes-server-version: v1.34
weight: 20
---
{{< feature-state feature_gate_name="DynamicResourceAllocation" >}}

<!-- overview -->

このページでは、_Dynamic Resource Allocation(DRA)_ を使用してPodにデバイスを割り当てる方法を示します。
これらの手順は、ワークロード運用者向けです。
このページを読む前に、DRAの仕組みと、{{< glossary_tooltip text="ResourceClaim" term_id="resourceclaim" >}}や{{< glossary_tooltip text="ResourceClaimTemplate" term_id="resourceclaimtemplate" >}}などのDRAの用語を理解してください。
詳細については、[Dynamic Resource Allocation(DRA)](/docs/concepts/resource-management/dynamic-resource-allocation/)を参照してください。

<!-- body -->

## DRAによるデバイス割り当てについて {#about-device-allocation-dra}

ワークロード運用者は、ResourceClaimまたはResourceClaimTemplateを作成することで、ワークロード用のデバイスを _要求(claim)_ できます。
ワークロードをデプロイすると、Kubernetesとデバイスドライバーが利用可能なデバイスを見つけ、それらをPodに割り当て、それらのデバイスにアクセスできるノードにPodを配置します。

<!-- prerequisites -->

## {{% heading "prerequisites" %}}

{{< include "task-tutorial-prereqs.md" >}} {{< version-check >}}

* クラスター管理者がDRAのセットアップ、デバイスの接続、およびドライバーのインストールを行なっていることを確認してください。
  詳細については、[クラスターにDRAをセットアップする](/docs/tasks/configure-pod-container/assign-resources/set-up-dra-cluster)を参照してください。

<!-- steps -->

## 要求するデバイスを特定する {#identify-devices}

クラスター管理者またはデバイスドライバーは、デバイスのカテゴリーを定義する _{{< glossary_tooltip term_id="deviceclass" text="DeviceClass" >}}_ を作成します。
{{< glossary_tooltip term_id="cel" >}}を使用して特定のデバイスプロパティで絞り込み、デバイスを要求できます。

クラスター内のDeviceClassの一覧を取得します:

```shell
kubectl get deviceclasses
```
出力は次のようになります:

```
NAME                 AGE
driver.example.com   16m
```
権限エラーが発生した場合は、DeviceClassを取得する権限がない可能性があります。
利用可能なデバイスプロパティについては、クラスター管理者またはドライバープロバイダーに確認してください。

## リソースを要求する {#claim-resources}

{{< glossary_tooltip text="ResourceClaim" term_id="resourceclaim" >}}を使用して、DeviceClassからリソースを要求できます。
ResourceClaimを作成するには、次のいずれかを行います:

* 複数のPodで同じデバイスへのアクセスを共有する場合、またはPodの存続期間を超えてクレームを存続させる場合は、ResourceClaimを手動で作成します。
* KubernetesにPodごとのResourceClaimを生成および管理させるには、{{< glossary_tooltip text="ResourceClaimTemplate" term_id="resourceclaimtemplate" >}}を使用します。
  すべてのPodが、類似した設定を持つ個別のデバイスにアクセスできるようにしたい場合は、ResourceClaimTemplateを作成します。
  例えば、[並列実行](/docs/concepts/workloads/controllers/job/#parallel-jobs)を使用するJob内のPodから、デバイスへ同時にアクセスしたい場合があります。

Podから特定のResourceClaimを直接参照する場合、そのResourceClaimはクラスター内にすでに存在している必要があります。
参照しているResourceClaimが存在しない場合、そのResourceClaimが作成されるまで、PodはPending状態のままとなります。
自動生成されたResourceClaimをPodから参照することもできますが、自動生成されたResourceClaimは、生成のきっかけとなったPodの存続期間に結び付いているため、推奨されません。

リソースを要求するワークロードを作成するには、次のいずれかの方法を選択します:

{{< tabs name="claim-resources" >}}
{{% tab name="ResourceClaimTemplate" %}}

次のマニフェストの例を確認します:

{{% code_sample file="dra/resourceclaimtemplate.yaml" %}}

このマニフェストは、次の両方のパラメーターに一致するデバイスを、`example-device-class` DeviceClassから要求するResourceClaimTemplateを作成します:

  * `driver.example.com/type`属性の値が`gpu`であるデバイス。
  * `64Gi`の容量を持つデバイス。

ResourceClaimTemplateを作成するには、次のコマンドを実行します:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
```

{{% /tab %}}
{{% tab name="ResourceClaim" %}}

次のマニフェストの例を確認します:

{{% code_sample file="dra/resourceclaim.yaml" %}}

このマニフェストは、次の両方のパラメーターに一致するデバイスを、`example-device-class` DeviceClassから要求するResourceClaimを作成します:

  * `driver.example.com/type`属性の値が`gpu`であるデバイス。
  * `64Gi`の容量を持つデバイス。

ResourceClaimを作成するには、次のコマンドを実行します:

```shell
kubectl apply -f https://k8s.io/examples/dra/resourceclaim.yaml
```

{{% /tab %}}
{{< /tabs >}}

## DRAを使用してワークロードでデバイスを要求する {#request-devices-workloads}

デバイス割り当てを要求するには、Pod仕様の`resourceClaims`フィールドにResourceClaimまたはResourceClaimTemplateを指定します。
次に、そのPod内のコンテナの`resources.claims`フィールドで、名前を指定して特定のクレームを要求します。
`resourceClaims`フィールドには複数の項目を指定でき、異なるコンテナに対して個別のクレームが使用できます。

1. 次のJobの例を確認します:

   {{% code_sample file="dra/dra-example-job.yaml" %}}

   このJobの各Podには、次のプロパティがあります:

   * `separate-gpu-claim`という名前のResourceClaimTemplateと、`shared-gpu-claim`という名前のResourceClaimをコンテナが利用できるようにします。
   * 次のコンテナを実行します:
       * `container0`は、`separate-gpu-claim` ResourceClaimTemplateからデバイスを要求します。
       * `container1`と`container2`は、`shared-gpu-claim` ResourceClaimのデバイスへのアクセスを共有します。

1. Jobを作成します:

   ```shell
   kubectl apply -f https://k8s.io/examples/dra/dra-example-job.yaml
   ```

次のトラブルシューティング手順を試してください:

1. ワークロードが期待どおりに起動しない場合は、JobからPod、さらにResourceClaimへと順に掘り下げていき、各レベルのオブジェクトを`kubectl describe`で確認して、ワークロードが起動しない理由が記載されているstatusフィールドやイベントがあるかどうかを確認します。
1. Podの作成が`must specify one of: resourceClaimName, resourceClaimTemplateName`というエラーで失敗する場合は、`pod.spec.resourceClaims`のすべての項目で、これらのフィールドのいずれか一つだけが設定されていることを確認します。
   一つだけ設定されている場合は、Kubernetes 1.32より前のAPIを使用してビルドされたMutating Pod Webhookが、クラスターにインストールされている可能性があります。
   クラスター管理者と協力して、これを確認してください。

## クリーンアップ {#clean-up}

このタスクで作成したKubernetesオブジェクトを削除するには、次の手順に従います:

1.  サンプルのJobを削除します:

    ```shell
    kubectl delete -f https://k8s.io/examples/dra/dra-example-job.yaml
    ```

1.  リソースクレームを削除するには、次のいずれかのコマンドを実行します:

    * ResourceClaimTemplateを削除します:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaimtemplate.yaml
      ```
    * ResourceClaimを削除します:

      ```shell
      kubectl delete -f https://k8s.io/examples/dra/resourceclaim.yaml
      ```

## {{% heading "whatsnext" %}}

* [DRAについてさらに学ぶ](/docs/concepts/resource-management/dynamic-resource-allocation/)
