.. _helm_chart:

##############################################
Kubernetes での FLARE の実行
##############################################

.. contents::
   :local:
   :depth: 2

NVIDIA FLARE は、まず通常のスタートアップキットをプロビジョニングし、次に各サーバーまたは
クライアントのキットを Kubernetes ランタイム向けに準備することで、Kubernetes にデプロイ
できます。準備されたキットには、参加者固有の Helm チャートと、Kubernetes ストレージへ
ステージングする必要のある ``startup/`` および ``local/`` フォルダが含まれます。

一時的な Kubernetes、OpenShift、マネージドクラウドクラスタのテストフローを自動化する
サンプルスクリプトについては、
:github_nvflare_link:`examples/devops <examples/devops>` を参照してください。これらの
スクリプトは開発、スモークテスト、デモ、学習のみを目的としたものであり、本番デプロイの
ガイダンスではありません。

前提条件
=============

作業を始める前に、以下が揃っていることを確認してください。

* プロビジョニングと ``nvflare deploy prepare`` を実行するワークステーションに
  ``nvflare`` がインストールされていること。
* 対象クラスタ向けに ``kubectl`` が設定されていること。Kubernetes API サーバーと
  互換性のある ``kubectl`` のバージョンを使用してください。
* ローカル環境、および ``kubectl cp`` で使用する一時的な Pod イメージの両方に ``tar``
  がインストールされていること。以下のステージング例では ``tar`` を含む ``busybox:1.36``
  を使用します。
* Helm 3。
* 標準の ``apps/v1`` Deployment、``rbac.authorization.k8s.io/v1``
  Role/RoleBinding、Service、Secret、PVC をサポートする Kubernetes クラスタ。
* 現在サポートされている Kubernetes リリース。生成されるチャートは安定した Kubernetes
  API を使用しており、プロバイダ固有の拡張には依存しません。
* デフォルトの ``StorageClass``、またはすべての PVC に対する明示的な
  ``storageClassName``。``kubectl get storageclass`` で確認してください。
* すべてのサーバークラスタおよびクライアントクラスタから pull できるコンテナレジストリ。
* ``resource_spec[site].num_of_gpus`` を指定したジョブを実行するクラスタには、NVIDIA GPU
  Operator または NVIDIA デバイスプラグインがインストールされていること。
  `クラウド GPU セットアップの参考資料`_ を参照してください。
* Kubernetes でのジョブ起動には、Python 3.13 以降の厳格な X.509 検証を通過する
  Kubernetes API サーバーの CA チェーンが必要です。CA 証明書には、証明書署名が許可された
  ``keyUsage`` など、RFC 5280 で必須とされる拡張が含まれている必要があります。

生成されるチャートは、Kubernetes クラスタ、ストレージクラス、GPU デバイスプラグイン、
Ingress コントローラ、レジストリ認証情報のインストールは行いません。

クラウド GPU セットアップの参考資料
-----------------------------------

マネージド Kubernetes サービスでは、GPU ドライバ、NVIDIA Container Toolkit、
NVIDIA GPU Operator、NVIDIA Kubernetes デバイスプラグインの扱い方が異なります。
GPU ジョブを実行する前に、GPU ノードが割り当て可能な ``nvidia.com/gpu`` リソースを
公開していることを確認してください。

お使いのクラスタについては、プロバイダの最新ドキュメントを参照してください。

* Amazon Elastic Kubernetes Service (EKS): `Amazon EKS での NVIDIA GPU デバイスの管理
  <https://docs.aws.amazon.com/eks/latest/userguide/device-management-nvidia.html>`__
  および `eksctl での GPU サポート
  <https://docs.aws.amazon.com/eks/latest/eksctl/gpu-support.html>`__。
* Google Kubernetes Engine (GKE): `GKE で NVIDIA GPU Operator を使って GPU スタックを
  管理する
  <https://cloud.google.com/kubernetes-engine/docs/how-to/gpu-operator>`__ および
  `GKE における GPU について
  <https://cloud.google.com/kubernetes-engine/docs/concepts/gpus>`__。
* Azure Kubernetes Service (AKS): `AKS で GPU を使用する
  <https://learn.microsoft.com/en-us/azure/aks/use-nvidia-gpu>`__ および `AKS での
  NVIDIA GPU Operator
  <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/microsoft-aks.html>`__。
* NVIDIA: `NVIDIA GPU Operator
  <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/>`__。

Kubernetes ランタイムモデル
============================

Kubernetes デプロイには 2 つのランタイムレイヤーがあります。

* **親 Pod** は、長時間稼働する FLARE サーバーまたはクライアントのプロセスを実行します。
  Helm は、``nvflare deploy prepare`` が生成した参加者ごとの ``helm_chart/`` から
  この Pod をインストールします。親 Pod は、設定されたワークスペース PVC を
  ``parent.workspace_mount_path`` にマウントし、その PVC から ``startup/`` と
  ``local/`` を読み込みます。Python 実行ファイルは ``parent.python_path`` で設定され、
  省略された場合は ``/usr/local/bin/python3`` がデフォルトになります。
* **ジョブ Pod** は、送信されたジョブごとに ``ServerK8sJobLauncher`` または
  ``ClientK8sJobLauncher`` によって動的に作成されます。ジョブ Pod のイメージ、Python
  パス、CPU、メモリ、GPU、エフェメラルストレージの設定は、送信されたジョブの
  ``launcher_spec`` と ``k8s.yaml`` 内の ``job_launcher`` デフォルト値から取得されます。

生成される Helm チャートは、送信されたジョブを直接実行するわけではありません。親となる
参加者プロセス、その Kubernetes Service、その ServiceAccount、そしてランチャーが
ジョブ Pod を作成できるようにする Role/RoleBinding をインストールします。

``job_launcher.config_file_path`` が省略されるか ``null`` に設定されている場合、
ランチャーは親 Pod の ServiceAccount によるクラスタ内 (in-cluster) の Kubernetes
設定を使用します。

親 Service は、動的に起動されるジョブ Pod にとってのクラスタ内の安定したアドレスです。
``nvflare deploy prepare`` は、準備されたキットの内部通信設定にパッチを当て、生成された
Service 名と ``parent_port`` を使用するようにします。``parent_port`` は、親／ジョブ間の
内部通信のためにジョブ Pod が使用する親プロセスのポートであり、リモートクライアントが
サーバーへ到達するために使用する連合学習用のポートではありません。Service の名前を変更
したり置き換えたりする場合は、Service 名、Service のポート、準備済みキットの通信設定を
一貫させてください。

ランタイムの構成は次のとおりです。

.. code-block:: text

   admin console
        |
        | FL/admin traffic to server fed_learn_port/admin_port
        v
   server cluster or namespace
     server parent pod
       | mounts workspace PVC: startup/, local/, transfer/
       | launches server job pods through Kubernetes API
       v
     server job pod emptyDir workspace
       | optional mounts: /data/<study>/<dataset> from study-data PVCs
       | workspace transfer over parent Service on parent_port

   client cluster or namespace
     client parent pod
       | outbound FL connection to server fed_learn_port
       | mounts workspace PVC: startup/, local/
       | launches client job pods through Kubernetes API
       v
     client job pod emptyDir workspace
       | optional mounts: /data/<study>/<dataset> from study-data PVCs
       | workspace transfer over client parent Service on parent_port

サーバーとクライアントの参加者は、同一の Kubernetes クラスタ内で実行することも、別々の
クラスタで実行することもできます。各サイトが自身の計算資源とデータを管理するため、
別々のクラスタを使用するのが一般的です。参加者が別々のクラスタで実行される場合、各クラスタで
同じネームスペース名と PVC 名を使用しても問題ありません。複数の参加者を 1 つのクラスタで
実行する場合は、参加者ごとに専用のネームスペースまたは専用のワークスペース PVC を割り当てて
ください。サーバーとクライアントの ``startup/`` と ``local/`` の内容は異なるため、両者を
同じワークスペース PVC に向けてはいけません。

クライアントサイトには、プロビジョニング時に設定されたサーバーエンドポイント (通常は
``<server-host>:<fed_learn_port>``) への外向きのネットワークアクセスが必要です。
クライアントサイトには、受信用の FL ポートも外部公開された Service も必要ありません。
クライアントのチャートがクラスタ内 Service を作成するのは、動的に起動されるクライアント
ジョブ Pod が自身のクライアント親 Pod に到達できるようにするためだけです。

準備された各参加者フォルダには、それぞれ専用のチャートが含まれます。

.. code-block:: text

   server-k8s/
     helm_chart/
     local/
     startup/
     transfer/

   site-1-k8s/
     helm_chart/
     local/
     startup/
     transfer/

``transfer/`` ディレクトリは、通常の FLARE 管理者ファイル転送ディレクトリです。サーバーの
場合、管理者ストレージが ``transfer`` として設定されているときに、マウントされたワーク
スペース配下で使用されます。これは Kubernetes のジョブワークスペース転送の仕組みではなく、
ジョブ Pod はこれをマウントしません。``startup/`` と ``local/`` をステージングする際に、
サーバーのワークスペース PVC 上にこのディレクトリをステージングまたは作成してください。

FLARE イメージのビルドとプッシュ
=================================

Helm チャートには、すべての参加クラスタが pull できる FLARE ランタイムイメージが必要です。
イメージのビルドとレジストリへのプッシュのワークフローについては、
:ref:`brev_build_push_flare_image` を参照してください。

NVIDIA は、``nvcr.io`` の NGC コンテナレジストリで公式の NVFlare Docker イメージを公開
しています。スタートアップキットのプロビジョニングと準備に使用した NVFlare のバージョンに
一致するタグを使用し、そのイメージを ``k8s.yaml`` の ``parent.docker_image`` に設定して
ください。

ユーザーは、``docker/Dockerfile.parent`` を変更して独自の親ランタイムイメージをこの
リポジトリからビルドし、すべての参加クラスタが pull できるレジストリにプッシュすることも
できます。親サーバーまたは親クライアントがジョブ Pod を作成できるように、NVFlare の
``K8S`` エクストラを維持するか、Kubernetes Python クライアントを明示的にインストール
してください。

親イメージは ``k8s.yaml`` の ``parent.docker_image`` から取得され、
``helm_chart/values.yaml`` に反映されます。送信されるジョブでも、``meta.json`` の
``launcher_spec[site][k8s].image`` または ``launcher_spec.default.k8s.image`` で
ジョブイメージを指定する必要があります。親イメージとジョブイメージは同じイメージでも
構いませんが、同じである必要はありません。

スタートアップキットの準備
===========================

アイデンティティ情報、証明書、サーバーのホスト名、FL ポート、FLARE の設定については、
引き続きプロビジョニングのステップが担当します。

.. code-block:: bash

   nvflare provision -p project.yml -w workspace

``project.yml`` 内のサーバーの ``default_host`` と ``host_names`` は、クライアントや
管理コンソールがサーバーへ到達するために使用する外部エンドポイントと一致している必要が
あります。これらの値が変更された場合は、再プロビジョニングしたうえで
``nvflare deploy prepare`` を再実行してください。

プロビジョニング後、``nvflare deploy prepare`` で各サーバーまたはクライアントの
スタートアップキットを準備します。

.. code-block:: bash

   nvflare deploy prepare workspace/<project>/prod_00/server \
       --output server-k8s \
       --config k8s.yaml

   nvflare deploy prepare workspace/<project>/prod_00/site-1 \
       --output site-1-k8s \
       --config k8s.yaml

``k8s.yaml`` の例を示します。

.. code-block:: yaml

   runtime: k8s
   namespace: nvflare
   parent:
     docker_image: registry.example.com/nvflare:dev
     image_pull_secrets:
       - registry-credentials
     parent_port: 8102
     workspace_pvc: nvflws
     workspace_mount_path: /var/tmp/nvflare/workspace
     python_path: /usr/local/bin/python3
     resources:
       requests:
         cpu: "2"
         memory: 8Gi
   job_launcher:
     config_file_path:
     default_python_path: /usr/local/bin/python3
     image_pull_secrets:
       - job-registry-credentials
     pending_timeout: 300

このランタイム設定は、サイトレベルの Kubernetes 設定を制御します。

* ``namespace`` は、親 Pod と動的に起動されるジョブ Pod が実行されるネームスペースです。
* ``server_service_name`` は、FL サーバーの Kubernetes Service 名を設定します。
  デフォルトは ``nvflare-server`` です。
* ``parent`` の値は Helm チャートに反映されます。これらは、親イメージ、Python 実行ファイル、
  ワークスペース PVC、親サービスのポート、親 Pod のリソース、任意の親 Pod セキュリティ
  コンテキスト、任意のイメージ pull Secret 参照を設定します。
  ``parent.image_pull_secrets`` には、対象ネームスペースに既に存在する Kubernetes Secret
  を指定する必要があります。NVFLARE はレジストリ認証情報を作成しません。この設定は生成
  される親 Pod のチャートに適用されます。動的に起動されるジョブ Pod には
  ``job_launcher.image_pull_secrets`` を使用してください。
  ``parent.python_path`` は、長時間稼働する SP/CP 親 Pod のコマンドを制御します。
  ``parent.workspace_mount_path`` は K8s ランチャーの設定にも書き込まれ、起動される
  SJ/CJ ジョブ Pod がジョブワークスペースとスタートアップキットをコンテナ内の同じパスに
  マウントするようにします。
* ``job_launcher`` の値は、親プロセスがジョブ Pod を作成できるように、参加者の
  ``local/resources.json.default`` に書き込まれます。``config_file_path`` はクラスタ内
  設定の場合は空でも構いません。``default_python_path`` は、ジョブが
  ``launcher_spec[site][k8s].python_path`` を上書きしない場合に SJ/CJ ジョブ Pod を
  制御します。これは SP/CP 親 Pod の Python パスは制御しません。そのコマンドには
  ``parent.python_path`` を使用してください。
  ``image_pull_secrets`` は、この準備済みサイトで動的に起動されるすべてのジョブ Pod に
  付与される、既存の Kubernetes イメージ pull Secret を指定します。ジョブイメージが
  プライベートレジストリにある場合は、デプロイ準備の段階でこれを設定してください。
  ジョブの作成者は引き続き ``meta.json`` でジョブイメージを指定するだけで済みます。
  動的に起動されるジョブ Pod のスタディ固有の Pod テンプレートは、
  ``local/study_runtime.yaml`` の ``pod_template`` によってスタディごとに設定します。
  ``pending_timeout`` は秒単位です。これは、動的に起動されたジョブ Pod が ``Pending``
  または ``Unknown`` の状態にとどまることのできる時間を制御し、この時間を超えるとランチャーは
  その Pod を削除し、実行を実行例外として報告します。その結果、管理者の ``list_jobs``
  コマンドは、このタイムアウトをユーザーによる中断として扱うのではなく、
  ``FINISHED:EXECUTION_EXCEPTION`` と表示します。

親 Pod とジョブ Pod では、異なる Python 設定が使用されます。

.. list-table::
   :header-rows: 1

   * - 設定
     - 適用対象
     - 備考
   * - ``parent.python_path``
     - 親サーバー Pod または親クライアント Pod
     - ``server_train`` または ``client_train`` 用の Helm コンテナコマンドとして
       反映されます。
   * - ``job_launcher.default_python_path``
     - 動的に起動されるジョブ Pod
     - ジョブが ``launcher_spec[site][k8s].python_path`` を設定していない場合に
       使用されます。
   * - ``launcher_spec[site][k8s].python_path``
     - 動的に起動されるジョブ Pod
     - ``meta.json`` におけるジョブ単位の上書き設定です。

クラスタストレージの準備
=========================

参加者を起動する前に、クラスタで必要となるワークスペース PVC やスタディデータ PVC を
作成してバインドしてください。

ネームスペース付きの PVC マニフェストを適用したり、Helm チャートをインストールしたりする
前に、ネームスペースを作成します。

.. code-block:: bash

   export NAMESPACE=nvflare
   kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

ワークスペース PVC
-------------------

ワークスペース PVC は、親サーバー Pod または親クライアント Pod のためのものです。生成
されるチャートは ``parent.workspace_pvc`` を ``parent.workspace_mount_path`` に
マウントしますが、PVC へファイルをアップロードすることはありません。チャートをインストール
する前に、親 Pod の ``startup/`` および ``local/`` フォルダについて、サポートされている
2 つのステージング方法のいずれかを選択してください。

- 準備済みキットの ``startup/`` および ``local/`` ディレクトリを、ワークスペース PVC の
  ルートへコピーする。
- ``nvflare deploy k8s stage`` を実行して、``local/`` 用の ConfigMap と ``startup/``
  用の Secret を作成し、生成されたチャートの値にパッチを当てる。

PVC コピー方式を使用するサーバーキットでは、管理者ファイル転送用ストレージとして
``transfer/`` もワークスペースのルートに作成またはコピーしてください。以下に示すように
``kubectl cp`` を使用する場合、``kubectl cp`` は対象コンテナ内に ``tar`` を必要とする
ため、一時的なコピー用 Pod のイメージには ``tar`` が含まれている必要があります。

いずれのステージング方法の後でも、生成されたチャートに対して ``helm upgrade --install``
を実行し、長時間稼働する親サーバー Pod または親クライアント Pod を起動します。

``workspace-pvc.yaml`` の例を示します。

.. code-block:: yaml

   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nvflws
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
     # If your cluster has no default StorageClass, uncomment and set this.
     # storageClassName: <storage-class-name>

サーバーのジョブ履歴、スナップショット、ログにより多くの容量が必要な場合は、より大きな
サイズを指定してください。複数の参加者が同じネームスペースで実行される場合は、参加者ごとに
別々のワークスペースクレームを使用してください。

方法 1: ワークスペース PVC へコピーする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

例として、``server-k8s`` という名前の準備済みフォルダと ``nvflws`` という名前の
ワークスペース PVC を使い、``startup/`` と ``local/`` を PVC のルートへ直接コピーします。

.. code-block:: bash

   export NAMESPACE=nvflare
   export PREPARED_KIT=server-k8s
   export WORKSPACE_PVC=nvflws

   kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -
   kubectl -n "$NAMESPACE" apply -f workspace-pvc.yaml
   kubectl -n "$NAMESPACE" get pvc "$WORKSPACE_PVC"

   kubectl -n "$NAMESPACE" delete pod nvflare-pvc-copy --ignore-not-found=true
   cat >/tmp/nvflare-pvc-copy.json <<EOF
   {
     "spec": {
       "restartPolicy": "Never",
       "volumes": [
         {"name": "ws", "persistentVolumeClaim": {"claimName": "${WORKSPACE_PVC}"}}
       ],
       "containers": [
         {
           "name": "nvflare-pvc-copy",
           "image": "busybox:1.36",
           "command": ["sleep", "600"],
           "volumeMounts": [{"name": "ws", "mountPath": "/mnt/nvflws"}]
         }
       ]
     }
   }
   EOF
   kubectl -n "$NAMESPACE" run nvflare-pvc-copy \
       --image=busybox:1.36 \
       --restart=Never \
       --overrides="$(cat /tmp/nvflare-pvc-copy.json)"
   kubectl -n "$NAMESPACE" wait --for=condition=Ready pod/nvflare-pvc-copy --timeout=120s
   kubectl -n "$NAMESPACE" exec nvflare-pvc-copy -- rm -rf /mnt/nvflws/startup /mnt/nvflws/local
   kubectl -n "$NAMESPACE" cp "$PREPARED_KIT/startup" nvflare-pvc-copy:/mnt/nvflws/startup
   kubectl -n "$NAMESPACE" cp "$PREPARED_KIT/local" nvflare-pvc-copy:/mnt/nvflws/local
   kubectl -n "$NAMESPACE" exec nvflare-pvc-copy -- mkdir -p /mnt/nvflws/transfer
   kubectl -n "$NAMESPACE" exec nvflare-pvc-copy -- ls -la /mnt/nvflws
   kubectl -n "$NAMESPACE" delete pod nvflare-pvc-copy

OpenShift 用のヘルパースクリプト
:github_nvflare_link:`examples/devops/openshift/scripts/k8s_deploy.sh <examples/devops/openshift/scripts/k8s_deploy.sh>`
は、この PVC コピー方式を一連の流れとして示しています。
:github_nvflare_link:`examples/devops/openshift/scripts/k8s_common.sh <examples/devops/openshift/scripts/k8s_common.sh>`
内の ``stage_workspace_pvc`` ヘルパーが一時的なコピー用 Pod を作成し、``startup/`` と
``local/`` を PVC へコピーします。その後、スクリプトが参加者ごとに Helm を実行します。

PVC のルートには ``startup/`` と ``local/`` が直接含まれている必要があります。実行時、
これらのフォルダは設定されたワークスペースのマウントパス (``parent.workspace_mount_path``。
``persistence.workspace.mountPath`` として反映されます) の配下に現れます。例のデフォルト
設定では、親は ``/var/tmp/nvflare/workspace/startup`` と
``/var/tmp/nvflare/workspace/local`` を想定します。PVC のルートに代わりに
``server-k8s/`` や ``site-1-k8s/`` のような入れ子のフォルダが含まれている場合、親 Pod は
設定されたマウントパス配下でこれらのフォルダを見つけられません。

方法 2: ConfigMap と Secret をステージングする
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``startup/`` と ``local/`` を PVC にコピーする代わりに、``nvflare deploy k8s stage``
を実行して、これらのフォルダ用の読み取り専用の Kubernetes リソースを作成することもできます。
``nvflare deploy k8 stage`` もエイリアスとして受け付けられます。

.. code-block:: bash

   nvflare deploy k8s stage "$PREPARED_KIT" --namespace "$NAMESPACE"

``kubectl`` ではなく ``oc`` を使って OpenShift へステージングする場合は、
``--kubectl oc`` を使用してください。このコマンドは ``local/`` 用の ConfigMap と
``startup/`` 用の Secret を作成し、その後 ``helm_chart/values.yaml`` にパッチを当てて、
親 Pod がそれらを ``/var/tmp/nvflare/workspace/local`` と
``/var/tmp/nvflare/workspace/startup`` にマウントするようにします。ワークスペース PVC は
引き続きワークスペースのルートにマウントされ、ジョブ、スナップショット、ログ、
``transfer/`` などの書き込み可能なランタイム状態に使用されます。このステージングコマンドが
成功したら、表示された ``helm_command``、または準備済みチャートに対する同等の
``helm upgrade --install`` コマンドを実行してください。ステージングされた ConfigMap と
Secret は Helm リリースとは独立しています。リリースをアンインストールした後、表示された
``cleanup_command`` または以下の同等のコマンドを実行してこれらを削除してください。

.. code-block:: bash

   helm uninstall "$RELEASE_NAME" --namespace "$NAMESPACE"
   nvflare deploy k8s unstage "$PREPARED_KIT"

stage コマンドは、ネームスペースと正確なリソース名を準備済みチャートの値に記録するため、
それらを繰り返し指定する必要はありません。記録機能のない古いバージョンの NVFlare で
ステージングされたキットをクリーンアップする場合は、``--namespace`` を指定してください。
親 Pod はインストールされている間ステージングされたボリュームに依存するため、unstage は
Helm のアンインストール後に実行してください。

動的に起動されるジョブ Pod は、このワークスペース PVC を **マウントしません** 。各ジョブ
Pod には、設定されたワークスペースのマウントパスにマウントされた、書き込み可能な専用の
``emptyDir`` が与えられます。ランチャーは、Pod の起動時に必要な ``local/`` とジョブ
ワークスペースの内容をその ``emptyDir`` へ転送し、ジョブの終了時にジョブの結果を親プロセス
へアップロードします。ジョブ Pod のワークスペースサイズは、設定されている場合は
``launcher_spec[site][k8s].ephemeral_storage`` によって、設定されていない場合は
ランチャーのデフォルト値によって制御されます。同じ値がコンテナの ``ephemeral-storage``
のリクエストおよびリミットにも使用されます。

スタディデータ PVC
-------------------

スタディデータ PVC は、親のワークスペース PVC とは別のものです。``local/`` をワークスペース
PVC へコピーする前に、準備済みキット内の ``local/study_runtime.yaml`` で任意のスタディ
データマッピングを設定してください (``nvflare deploy prepare`` はコメント付きのテンプレート
を書き出します。ランチャーはこのファイルを自動的に検出するため、ランチャーへの引数は不要
です)。キットが既にステージング済みの場合は、PVC 上のファイルを編集するか、``local/`` を
再ステージングしてください。

``study_runtime.yaml`` の例を示します。

.. code-block:: yaml

   format_version: 2
   studies:
     default:
       datasets:
         data:
           source: nvfldata
           mode: ro

Kubernetes の場合、各データセットの ``source`` の値は PVC のクレーム名です。ジョブ Pod は
データセットを ``/data/<study>/<dataset>`` (例: ``/data/default/data``) にマウントします。
``mode`` は ``ro`` または ``rw`` でなければなりません。ジョブのスタディに対応するエントリ
がない場合、そのジョブにはスタディデータ PVC はマウントされません。同じファイルでは、
スタディごとの環境変数、Secret に基づく環境変数とマウント、Pod テンプレートも設定できます。
ランチャーはジョブ起動のたびにこのファイルを読み直します。``pod_template`` から参照される
テンプレートファイルは、``local/`` と一緒にステージングする必要があります。

``local/study_data.yaml`` を引き続き使用する従来の v1 キットも動作します。その場合、
``nvflare deploy prepare`` はそのファイルを指すランチャーの
``study_data_pvc_file_path`` を出力し、``study_runtime.yaml`` のテンプレートは書き出し
ません。この 2 つのファイルを共存させてはいけません。移行するには、すべてのスタディを
``study_runtime.yaml`` へ移し、``study_data.yaml`` を削除してください。

あるスタディの ``local/study_runtime.yaml`` エントリで ``pod_template`` が設定されている
場合、該当するジョブはそのスタディ固有の Pod テンプレートを使用し、同じスタディの
``datasets`` エントリが PVC ボリュームマウントとして追加されます。ランチャーは常に、
テンプレートの ``workspace-job`` および ``startup-kit`` ボリュームとジョブコンテナの
マウントを、自身が生成したワークスペース ``emptyDir`` とスタートアップキットの Secret
マウントで置き換えます。

最小構成のスタディジョブ Pod テンプレート
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

``local/study_runtime.yaml`` でスタディごとにテンプレートを参照します (パスは ``local/``
からの相対パスとして解決されます。インラインの pod マッピングも受け付けられます)。

.. code-block:: yaml

   format_version: 2
   studies:
     study-a:
       pod_template: pod_specs/default-job-pod.yaml

次の ``pod_specs/default-job-pod.yaml`` は、最小構成の Pod テンプレートを出発点として、
NVIDIA GPU Operator または NVIDIA GPU Feature Discovery (GFD) によってラベル付けされた
H100 ノード向けのノードセレクターを含む、よく使われる任意指定のフィールドを示しています。
セレクターを設定する前に、お使いのクラスタでの正確なラベル値を確認してください。

.. code-block:: console

   $ kubectl get nodes -L nvidia.com/gpu.product,nvidia.com/gpu.count,nvidia.com/gpu.present

任意指定のフィールドを省略して ``nvflare_job`` コンテナだけを残した場合、スタディは自身の
``pod_template`` を使用しつつ、組み込みのランチャーの動作と同じ実効的なジョブ Pod
マニフェストを維持します。

.. code-block:: yaml

   apiVersion: v1
   kind: Pod
   metadata:
     labels:
       nvflare.io/study: study-a
       workload: h100-training
     annotations:
       nvflare.io/study-owner: research-team-a
       cluster-autoscaler.kubernetes.io/safe-to-evict: "false"
   spec:
     serviceAccountName: study-a-job
     nodeSelector:
       nvidia.com/gpu.product: NVIDIA-H100-80GB-HBM3
     tolerations:
       - key: nvidia.com/gpu
         operator: Exists
         effect: NoSchedule
     containers:
       - name: nvflare_job

起動時、NVFLARE は ``nvflare_job`` コンテナを選択し、組み込みマニフェストで使用されるのと
同じランチャー管理のフィールド、すなわち Pod 名、ジョブコンテナ名、イメージ、コマンド、
引数、リソース、ワークスペース ``emptyDir``、スタートアップキットの Secret、ボリューム
マウント、転送用環境変数、イメージ pull Secret、``restartPolicy: Never`` を上書きします。
``serviceAccountName``、``nodeSelector``、``affinity``、``tolerations``、サイドカー
コンテナ、追加のボリュームといったテンプレートフィールドは、そのスタディが元のランチャー
マニフェストとは異なる動作を必要とする場合にのみ追加してください。

Kubernetes のノードラベルを使用して、スタディのジョブ Pod を特定のノードへ誘導できます。
NVIDIA GPU Operator または GFD を導入したクラスタでは、GPU ノードには通常
``nvidia.com/gpu.product`` や ``nvidia.com/gpu.count`` などのラベルが付与されています。
上記の H100 セレクターは、NVIDIA がフル構成の H100 80GB HBM3 ノードについて文書化して
いる製品ラベルの値と一致します。クラスタで MIG や GPU 共有、あるいは異なるフォーム
ファクタの H100 を使用している場合は、``kubectl get nodes`` から正確な
``nvidia.com/gpu.product`` の値をコピーしてください。複数の H100 製品ラベルを許容する
といった、より複雑な配置ルールが必要な場合は、``nodeSelector`` の代わりに、あるいはそれに
加えて ``spec.affinity.nodeAffinity`` を使用してください。

.. code-block:: yaml

   spec:
     affinity:
       nodeAffinity:
         requiredDuringSchedulingIgnoredDuringExecution:
           nodeSelectorTerms:
             - matchExpressions:
                 - key: nvidia.com/gpu.product
                   operator: In
                   values:
                     - NVIDIA-H100-80GB-HBM3
                     - NVIDIA-H100-NVL

特定の名前のノードを対象にするには、標準の ``kubernetes.io/hostname`` ラベルなど、その
ノードを識別できるラベルを使用するか、独自の運用ラベルを追加してテンプレートから選択して
ください。厳密なノード選択を行うと、選択したノードに空き容量がない場合にジョブ Pod が
``Pending`` のままになる可能性がある点に注意してください。
``metadata.annotations`` の Pod アノテーションは保持され、アドミッションコントローラ、
スケジューラ、監視連携から利用できますが、Kubernetes はアノテーションだけでノードを選択
することはありません。GPU のリソースリクエストとリミットは引き続きランチャーの管理下に
あります。これらは Pod テンプレートではなく、送信するジョブの
``launcher_spec[site][k8s].num_of_gpus`` で設定してください。

``nvfldata-pvc.yaml`` の例を示します。

.. code-block:: yaml

   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: nvfldata
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 50Gi
     # If your cluster has no default StorageClass, uncomment and set this.
     # storageClassName: <storage-class-name>

お使いのストレージバックエンドがサポートするアクセスモードを使用してください。多くの
シングルノードまたは単一ジョブのケースでは ``ReadWriteOnce`` で十分です。異なるノード上の
複数のジョブ Pod が同じデータセットへ同時にアクセスする必要がある場合は、``ReadOnlyMany``
または ``ReadWriteMany`` のストレージ、あるいはサイトごとに分離したクレームを使用して
ください。

参加者のジョブ Pod が実行されるのと同じネームスペースにスタディデータ PVC を適用します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" apply -f nvfldata-pvc.yaml
   kubectl -n "$NAMESPACE" get pvc nvfldata

チャートのインストール
=======================

各サーバーまたはクライアントのキットは、その参加者が実行される Kubernetes クラスタまたは
ネームスペースにおいて、準備、ステージング、インストールを行ってください。上記のいずれかの
ステージング方法を実施した後、生成された Helm チャートをインストールして、長時間稼働する
親 Pod を起動します。

サーバーのチャートをインストールします。

.. code-block:: bash

   export NAMESPACE=nvflare

   helm upgrade --install server server-k8s/helm_chart \
       --namespace "$NAMESPACE"

同じパターンでクライアントのチャートをインストールします。

.. code-block:: bash

   helm upgrade --install site-1 site-1-k8s/helm_chart \
       --namespace "$NAMESPACE"

``nvflare deploy prepare`` は、``k8s.yaml`` の ``parent.docker_image`` を基に
``image.repository`` と ``image.tag`` を ``helm_chart/values.yaml`` へ書き込みます。
別の親イメージを使用する場合は、更新した ``k8s.yaml`` で ``nvflare deploy prepare`` を
再実行してください。Helm のインストール時またはアップグレード時にイメージを上書きする必要が
ある場合は、values ファイルを使用し、関連するすべての ``helm upgrade`` コマンドにそれを
渡すことを推奨します。

.. code-block:: bash

   cat > server-values.yaml <<'EOF'
   image:
     repository: registry.example.com/nvflare
     tag: dev
   EOF

   helm upgrade --install server server-k8s/helm_chart \
       --namespace "$NAMESPACE" \
       -f server-values.yaml

イメージ変更の正となる情報源として、その場限りの ``--set image.repository=...`` や
``--set image.tag=...`` フラグを使用することは避けてください。同じ上書き設定を含まない
後続のアップグレードコマンドを実行すると、生成されたチャートのデフォルト値でリリースが
レンダリングされてしまう可能性があります。

サーバーとクライアントが同じネームスペースで実行される場合は、異なるワークスペース PVC を
使用するか、いずれかのリリースで ``persistence.workspace.claimName`` を上書きしてください。

.. code-block:: bash

   helm upgrade --install site-1 site-1-k8s/helm_chart \
       --namespace "$NAMESPACE" \
       --set persistence.workspace.claimName=nvflws-site-1

ネームスペース付きの ``kubectl`` コマンドを実行したり、チャートをインストールしたりする
前に、ネームスペースが既に存在している必要があります。上記のストレージ準備のステップで
明示的に作成しています。その手順を省略する場合は、先にネームスペースを作成してください。

.. code-block:: bash

   kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

FL トラフィックの公開
======================

生成されるサーバーチャートは、FL サーバー用の Kubernetes Service を作成します。この
Service のデフォルトは ``ClusterIP`` であり、クラスタ内からのみ到達できます。クライアント
や管理コンソールがクラスタ外部から接続する場合は、お使いの Kubernetes 環境に合った仕組みで
FL サーバーのポートを公開してください。

サーバーリリースに上書き用の values ファイルを使用している場合は、以下の ``helm upgrade``
コマンドにも同じ ``-f`` ファイルを含めてください。

* 利用できる場合はクラウドのロードバランサーを使用します。

  .. code-block:: bash

     helm upgrade --install server server-k8s/helm_chart \
         --namespace "$NAMESPACE" \
         --set service.type=LoadBalancer
     kubectl -n "$NAMESPACE" get svc nvflare-server

* 同一マシン上でのローカルテストには、ポートフォワーディングを使用します。

  .. code-block:: bash

     kubectl -n "$NAMESPACE" port-forward svc/nvflare-server 8002:8002 8003:8003

* シングルノードクラスタや Ingress ベースのクラスタでは、``project.yml`` の FL ポートと
  管理ポートが ``nvflare-server`` Service に到達するように、クラスタの TCP ルーティング、
  ファイアウォールルール、ホストポートを設定してください。シングルノードのデプロイでは、
  サーバーチャートに対して ``--set hostPortEnabled=true`` を使用する場合もあります。

プロビジョニング時に使用したサーバーのホスト名が、公開されたアドレスに解決されることを
確認してください。たとえば、管理コンソールおよびすべてのリモートクライアントサイトで DNS
や ``/etc/hosts`` を更新します。

デプロイメントの検証
=====================

チャートをインストールした後、Deployment、Pod、Service、PVC が正常であることを確認します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" rollout status deployment/server --timeout=300s
   kubectl -n "$NAMESPACE" rollout status deployment/site-1 --timeout=300s
   kubectl -n "$NAMESPACE" get pods,svc,pvc
   kubectl -n "$NAMESPACE" logs deploy/server --tail=200
   kubectl -n "$NAMESPACE" logs deploy/site-1 --tail=200

Pod が Ready にならない場合は、その Pod と最近のイベントを確認します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" describe pod -l app.kubernetes.io/instance=server
   kubectl -n "$NAMESPACE" get events --sort-by=.lastTimestamp

Pod のログは、その Pod が存在する間しか保持されません。親 Pod が再起動したり、Helm の
アップグレードによって再作成されたりすると、それ以前のログは失われます。ログを保持する
必要がある場合は、クラスタのログ集約機能を使用するか、外部でログを取得してください。

管理コンソールでのログイン
===========================

``nvflare provision`` が生成した管理者スタートアップキットを使用します。管理コンソールは、
プロビジョニングされたプロジェクトに書き込まれたサーバーのホストとポートに接続するため、
ログインする前に、それらの名前が公開された Kubernetes エンドポイントに解決されることを
確認してください。

.. code-block:: bash

   cd workspace/<project>/prod_00/admin@nvidia.com/startup
   bash fl_admin.sh

``User Name`` の入力を求められたら、``admin@nvidia.com`` のような ``project.yml`` の
管理者アイデンティティを入力してください。

プライベートレジストリとイメージ pull Secret
=============================================

生成されるチャートは、``helm_chart/values.yaml`` の ``imagePullSecrets`` を通じて親 Pod
のイメージ pull Secret をサポートします。``nvflare deploy prepare`` は、``k8s.yaml`` の
``parent.image_pull_secrets`` からこの値を設定します。対象の Kubernetes Secret は参加者の
ネームスペースに既に存在している必要があります。NVFLARE はレジストリ認証情報を作成しません。

例を示します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" create secret docker-registry registry-credentials \
       --docker-server=registry.example.com \
       --docker-username="$REGISTRY_USERNAME" \
       --docker-password="$REGISTRY_PASSWORD"

.. code-block:: yaml

   parent:
     docker_image: registry.example.com/nvflare:dev
     image_pull_secrets:
       - registry-credentials

これにより、親チャートの値は次のようにレンダリングされます。

.. code-block:: yaml

   imagePullSecrets:
     - name: registry-credentials

動的に起動されるジョブ Pod は、インストール後は Helm チャートの制御下にありません。
プライベートなジョブイメージを使用する場合は、``nvflare deploy prepare`` を実行する前に
``k8s.yaml`` で ``job_launcher.image_pull_secrets`` を設定してください。K8s ランチャーは、
作成する各ジョブ Pod の ``spec.imagePullSecrets`` にこれらの Secret 参照を書き込みます。

クラスタがノードレベルのレジストリ認証情報をサポートしている場合、またはネームスペースの
デフォルト ServiceAccount が既に適切なイメージ pull Secret を持っている場合は、明示的な
``image_pull_secrets`` 設定の代わりにそれを利用できます。

親 Pod またはジョブ Pod が ``ImagePullBackOff`` になった場合は、``kubectl describe pod``
で Pod のイベントを確認し、イメージ名、タグ、レジストリ認証情報、イメージ pull ポリシーが
正しいことを確かめてください。

Helm values リファレンス
=========================

``nvflare deploy prepare`` は、各参加者について生成したデフォルト値を
``helm_chart/values.yaml`` に書き込みます。最も頻繁に上書きされる値は、イメージ、
Service の公開設定、リソース、永続化に関するものです。

.. list-table::
   :header-rows: 1

   * - 値
     - スコープ
     - デフォルト値の由来
     - 用途
   * - ``name``
     - サーバーおよびクライアント
     - 参加者名
     - Deployment 名およびチャートのラベル。ただしチャートのヘルパーが別の名前を
       導出する場合を除きます。
   * - ``siteName``
     - クライアント
     - 参加者名
     - ``client_train`` に渡されるクライアント UID。
   * - ``serviceName``
     - サーバーおよびクライアント
     - サーバーの場合は ``server_service_name``、クライアントの場合は安定したサイト名
     - ジョブ Pod が親 Pod へ到達するために使用する Kubernetes Service 名。
   * - ``image.repository``
     - サーバーおよびクライアント
     - ``parent.docker_image`` のリポジトリ部分
     - 親 Pod のイメージリポジトリ。
   * - ``image.tag``
     - サーバーおよびクライアント
     - ``parent.docker_image`` のタグ部分
     - 親 Pod のイメージタグ。空の場合、リポジトリの値がそのまま使用されます。
   * - ``image.pullPolicy``
     - サーバーおよびクライアント
     - サーバーは ``IfNotPresent``、クライアントは ``Always``
     - 親 Pod のイメージ pull ポリシー。
   * - ``imagePullSecrets``
     - サーバーおよびクライアント
     - ``parent.image_pull_secrets`` を ``[{name: ...}]`` としてレンダリングしたもの
     - 親 Pod のイメージ pull Secret 参照。対象の Secret はリリースのネームスペースに
       既に存在している必要があります。
   * - ``serviceAccount.create``
     - サーバーおよびクライアント
     - ``true``
     - 親 Pod 用の ServiceAccount を作成します。
   * - ``serviceAccount.annotations``
     - サーバーおよびクライアント
     - ``{}``
     - 生成される ServiceAccount にアノテーションを追加します。
   * - ``serviceAccount.automountServiceAccountToken``
     - サーバーおよびクライアント
     - ``true``
     - 親ランチャーがクラスタ内の Kubernetes API アクセスを使用する場合は、有効の
       ままにしておく必要があります。
   * - ``rbac.create``
     - サーバーおよびクライアント
     - ``true``
     - ジョブ Pod とスタートアップ Secret の作成に必要な Role と RoleBinding を
       作成します。
   * - ``podAnnotations``
     - サーバーおよびクライアント
     - ``{}``
     - 親 Pod テンプレートにアノテーションを追加します。
   * - ``securityContext``
     - サーバーおよびクライアント
     - ``parent.pod_security_context`` または ``{}``
     - 親 Pod のセキュリティコンテキスト。
   * - ``resources``
     - サーバーおよびクライアント
     - ``parent.resources``、または CPU ``2`` とメモリ ``8Gi`` のリクエスト
     - 親 Pod のリソースリクエストおよびリミット。
   * - ``persistence.workspace.claimName``
     - サーバーおよびクライアント
     - ``parent.workspace_pvc`` または ``nvflws``
     - 親 Pod がマウントするワークスペース PVC。
   * - ``persistence.workspace.volumeName``
     - サーバーおよびクライアント
     - ``workspace``
     - 親 Pod マニフェスト内の内部ボリューム名。
   * - ``persistence.workspace.mountPath``
     - サーバーおよびクライアント
     - ``parent.workspace_mount_path``
     - コンテナ内のワークスペースマウントパス。
   * - ``fedLearnPort``
     - サーバー
     - プロビジョニングで設定されたサーバーの ``fed_learn_port``、または ``8002``
     - サーバー Service と親コンテナが公開する FL サーバーのポート。
   * - ``adminPort``
     - サーバー
     - ``fedLearnPort`` と異なる場合はサーバーの ``admin_port``、それ以外は
       ``null``
     - サーバー Service と親コンテナが公開する管理ポート。
   * - ``parentPort``
     - サーバー
     - ``parent.parent_port`` または ``8102``
     - サーバーのジョブ Pod 向けの内部親 Service ポート。
   * - ``port``
     - クライアント
     - ``parent.parent_port`` または ``8102``
     - クライアントのジョブ Pod 向けの内部親 Service ポート。
   * - ``hostPortEnabled``
     - サーバー
     - ``false``
     - サーバーの親 Pod に ``fedLearnPort`` と ``adminPort`` の ``hostPort`` を
       追加します。一部のシングルノードクラスタで有用です。
   * - ``tcpConfigMapEnabled``
     - サーバー
     - ``false``
     - FL ポートをサーバー Service にマッピングする MicroK8s nginx ingress の
       TCP-services ConfigMap を出力します。nginx ingress アドオンを使用する
       MicroK8s クラスタでのみ有用です。
   * - ``service.type``
     - サーバー
     - ``ClusterIP``
     - サーバー Service のタイプ (例: ``LoadBalancer``)。
   * - ``service.loadBalancerIP``
     - サーバー
     - ``null``
     - クラスタがサポートしている場合の、任意指定の静的ロードバランサー IP。
   * - ``service.annotations``
     - サーバーおよびクライアント
     - ``{}``
     - 生成される Service にアノテーションを追加します。
   * - ``command``
     - サーバーおよびクライアント
     - ``parent.python_path``
     - 親コンテナのコマンド。
   * - ``args``
     - サーバーおよびクライアント
     - ``nvflare deploy prepare`` によって生成されます
     - 親プロセスのモジュールおよびランタイム引数。FLARE の親プロセスがどのように
       起動されるかを理解している場合にのみ上書きしてください。

ランチャーの RBAC
==================

生成されるチャートは、デフォルトで ServiceAccount とネームスペーススコープの
Role/RoleBinding を作成します。ランチャーには次の権限が必要です。

* Pod の create、delete、get、list、watch。
* Secret の create、get、update、patch、delete。

Secret に関する権限が必要なのは、ランチャーが動的に起動されるジョブ Pod 用にサイトごとの
スタートアップキット Secret を作成または更新し、さらに ``secretKeyRef`` 経由でジョブの
ブートストラップ認証情報を環境変数として提供するジョブごとの認証情報 Secret
(``nvflare-cred-<pod-name>``) を作成するためです。認証情報 Secret には、その Pod への
ownerReference がパッチとして付与され、ジョブ終了時に削除されます。ジョブ Pod は
スタートアップキット Secret を ``<workspace_mount_path>/startup`` に読み取り専用で
マウントします。スタートアップキット Secret の名前は、次のパターンに従います。

.. code-block:: text

   nvflare-startup-<rfc1123-site-name>-<8-char-sha256-prefix>

``<rfc1123-site-name>`` は、RFC1123 に準拠しない文字を置き換えたサイト名です。8 文字の
SHA256 サフィックスは、既に RFC1123 に準拠しているサイト名の場合でも常に付加されるため、
Secret 名は自分で組み立てるのではなく、以下の ``grep`` の例で検索してください。一方、
Service 名と Deployment 名は、サイト名が既に DNS ラベルに準拠している場合 (英小文字、
数字、ハイフンからなり、先頭と末尾が英数字で、最大 63 文字) はサイト名をそのまま反映します。

たとえば、次のコマンドでスタートアップキットの Secret を確認できます。

.. code-block:: bash

   kubectl -n "$NAMESPACE" get secret | grep nvflare-startup

クラスタの運用者がチャートの値で ``serviceAccount.create`` または ``rbac.create`` を
無効にしている場合は、ジョブを送信する前に同じネームスペース内で同等の API アクセス権を
用意してください。親 Pod は、ジョブ Pod を作成でき、かつスタートアップキット Secret と
ジョブごとの認証情報 Secret を create、update、patch、delete できる ServiceAccount で
実行される必要があります。

Kubernetes ジョブ Pod の設定
=============================

ジョブ Pod の設定は、送信されるジョブの ``meta.json`` の ``launcher_spec`` 配下にあります。
``default`` ブロックはすべてのサイトに適用され、サイト固有のブロックがそれを上書きします。

.. code-block:: json

   {
     "launcher_spec": {
       "default": {
         "k8s": {
           "image": "registry.example.com/nvflare-job:latest",
           "python_path": "/usr/local/bin/python3",
           "cpu": "2",
           "memory": "8Gi",
           "ephemeral_storage": "8Gi"
         }
       },
       "site-1": {
         "k8s": {
           "image": "registry.example.com/site-1-job:latest",
           "cpu_request": "1",
           "memory_request": "4Gi"
         }
       }
     },
     "resource_spec": {
       "site-1": {
         "num_of_gpus": 1
       }
     }
   }

サポートされている ``launcher_spec[site][k8s]`` のキーには、次のものがあります。

* ``image``: ジョブ Pod のコンテナイメージ。``launcher_spec.default.k8s`` または
  サイト固有の ``k8s`` ブロックのいずれかで必ず指定する必要があります。
* ``python_path``: ジョブイメージ内の Python 実行ファイル。省略した場合、ランチャーは
  準備済みサイトのランタイム設定から ``job_launcher.default_python_path`` を使用します。
* ``cpu`` と ``memory``: コンテナのリミット。``cpu_request`` または ``memory_request``
  が省略された場合、リクエストは対応するリミットと同じ値になります。
* ``cpu_request`` と ``memory_request``: リクエストをリミットより小さくしたい場合に
  指定する任意の値です。
* ``ephemeral_storage``: ジョブワークスペースの ``emptyDir.sizeLimit`` と、コンテナの
  ``ephemeral-storage`` のリクエストおよびリミットに使用される Kubernetes の数量文字列
  です。``launcher_spec.default.k8s`` またはサイト固有の ``launcher_spec[site].k8s``
  ブロックで設定してください。省略した場合は、組み込みのランチャーのデフォルト値が使用
  されます。現在の ``deploy prepare`` のランタイム設定では、
  ``job_launcher.ephemeral_storage`` は ``k8s.yaml`` の設定項目として公開されていません。

ジョブ Pod は ``imagePullPolicy: Always`` で作成されます。タグの変更は即座に反映されますが、
送信されるジョブごとに、サイト単位で 1 回イメージが pull されます。プライベートレジストリを
使用する場合は、レート制限とレジストリ認証情報の受け渡しにこの点を織り込んでください。
動的に起動されるジョブ Pod に明示的なイメージ pull Secret が必要な場合は、
``job_launcher.image_pull_secrets`` を使用してください。

``resource_spec`` は引き続きスケジューラ向けの設定です。新しいジョブでは、K8s ランチャーの
設定は ``launcher_spec`` に、``num_of_gpus`` などのリソースリクエストは ``resource_spec``
に配置してください。ランチャーは、``resource_spec[site].num_of_gpus`` を
``nvidia.com/gpu`` のリクエストとリミットの両方として書き込みます。

GPU のリクエストには、対象クラスタ上に NVIDIA GPU Operator または NVIDIA デバイス
プラグインが必要です。MIG を使用する場合は、デバイスプラグインがランチャーの要求する
リソースを公開していることを確認してください。組み込みのランチャーは ``num_of_gpus`` に
対して ``nvidia.com/gpu`` を書き込みます。``nvidia.com/mig-1g.5gb`` のようなプロファイル
固有のリソースのみを公開するクラスタでは、それらのリソース名を要求するために、クラスタ側の
設定またはランチャーのカスタマイズが必要です。

再プロビジョニングとアップグレード
===================================

プロビジョニングされた証明書、ローカル設定、サーバーの通信設定、および準備済みの
Kubernetes 親 Service の設定は、プロビジョニングされたプロジェクトの状態と結び付いて
います。``project.yml``、サーバーのホスト名、ポート、参加者、または ``k8s.yaml`` の設定を
変更する場合は、ConfigMap/Secret 方式を使用しているときはまずそのステージングを
クリーンアップしてください。

.. code-block:: bash

   helm uninstall "$RELEASE_NAME" --namespace "$NAMESPACE"
   nvflare deploy k8s unstage "$PREPARED_KIT"

記録されたネームスペースと正確なクリーンアップ用の名前を利用できる状態に保つため、準備済み
の出力を置き換える前に unstage を実行してください。``deploy prepare`` は、ステージング
されたリソースをまだ参照しているチャートの上書きを拒否します。

その後、次の手順を実行します。

#. ``nvflare provision`` を再実行します。
#. 影響を受けるすべての参加者について ``nvflare deploy prepare`` を再実行します。
#. 再ステージングの前に、保持したい PVC の内容をバックアップします。サーバーのワークスペース
   PVC では、通常 ``transfer/`` (管理者のアップロード)、ジョブ履歴とスナップショットを
   保持するサイトディレクトリ、およびワークスペースのルートにあるログファイルが対象と
   なります。クライアントのワークスペース PVC では、通常は任意のログ以外に保持すべきものは
   ほとんどありません。
#. 選択した方法で ``startup/`` と ``local/`` を再ステージングします。PVC 方式の場合は、
   参加者のワークスペース PVC 上でこれらのフォルダを置き換え、先に古いコピーを削除して
   ください。ConfigMap/Secret 方式の場合は、新しい準備済みキットに対して
   ``nvflare deploy k8s stage`` を実行します。
#. 影響を受けるリリースに対して ``helm upgrade --install`` を実行します。

再プロビジョニング後に、古いステージング済みの ``startup/`` または ``local/`` フォルダを
再利用しないでください。

トラブルシューティング
=======================

PVC が ``Pending`` のままになる
--------------------------------

クラスタにデフォルトのストレージクラスが存在することを確認するか、各 PVC の ``spec``
配下に明示的な ``storageClassName`` を追加してください。

.. code-block:: bash

   kubectl get storageclass
   kubectl -n "$NAMESPACE" describe pvc nvflws
   kubectl -n "$NAMESPACE" describe pvc nvfldata

``storageClassName: ""`` は、動的なストレージクラスを使わずに事前作成された
PersistentVolume にバインドする場合にのみ使用してください。

親 Pod が ``ImagePullBackOff`` になる
--------------------------------------

親イメージが存在し、クラスタがそれを pull できることを確認します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" describe pod -l app.kubernetes.io/instance=server
   kubectl -n "$NAMESPACE" describe pod -l app.kubernetes.io/instance=site-1

レンダリングされたイメージを確認します。

.. code-block:: bash

   helm -n "$NAMESPACE" get values server --all
   helm -n "$NAMESPACE" get values site-1 --all

プライベートレジストリを使用する場合は、`プライベートレジストリとイメージ pull Secret`_
で説明しているとおり、ノードの認証情報を設定するか、イメージ pull Secret を追加してください。

親 Pod が ``startup`` または ``local`` を見つけられない
--------------------------------------------------------

準備済みキットが PVC 内の誤った階層にコピーされたか、誤った PVC がマウントされています。
設定されたワークスペースのマウントパスには、次のディレクトリが含まれている必要があります。

.. code-block:: text

   <workspace_mount_path>/startup
   <workspace_mount_path>/local

例のデフォルトの ``workspace_mount_path`` を使用した場合、それらのパスは次のようになります。

.. code-block:: text

   /var/tmp/nvflare/workspace/startup
   /var/tmp/nvflare/workspace/local

`ワークスペース PVC`_ で紹介したヘルパー Pod を使って ``/mnt/nvflws`` を確認し、準備済み
フォルダから ``startup/`` と ``local/`` を再ステージングしてください。

親は起動するがジョブ Pod を起動できない
----------------------------------------

親のログに Kubernetes のインポートエラーや認可の失敗が出ていないか確認します。

.. code-block:: bash

   kubectl -n "$NAMESPACE" logs deploy/server --tail=200
   kubectl -n "$NAMESPACE" auth can-i create pods \
       --as=system:serviceaccount:"$NAMESPACE":server
   kubectl -n "$NAMESPACE" auth can-i create secrets \
       --as=system:serviceaccount:"$NAMESPACE":server

ログに ``kubernetes`` Python パッケージが見つからない旨が出力されている場合は、NVFlare の
``K8S`` エクストラを含めて親イメージを再ビルドするか、
``pip install "kubernetes!=36.0.0"`` を実行してください。

ログに ``CA cert does not include key usage extension`` を伴う
``SSLCertVerificationError`` が出力されている場合は、親の Kubernetes クライアントが
クラスタ API サーバーの CA を拒否しています。これは、X.509 の ``keyUsage`` 拡張を省略して
いる一部の MicroK8s CA 証明書で発生することが知られています。
`canonical/microk8s#4864 <https://github.com/canonical/microk8s/issues/4864>`__ を
参照してください。クラスタの CA を RFC 5280 準拠の CA で再生成または置き換えてください。
開発用クラスタにおける一時的な互換性回避策としては、Python 3.12 以前をベースとしたカスタム
親イメージを使用してください。本番環境で Kubernetes API の TLS 検証を無効にしないで
ください。

ジョブ Pod が ``Pending`` または ``Unknown`` のままになる
----------------------------------------------------------

SJ または CJ のジョブ Pod が ``job_launcher.pending_timeout`` 秒を超えて ``Pending``
または ``Unknown`` のままとなり、送信されたジョブを開始できない場合、NVFLARE は停滞して
いる Pod を削除し、そのジョブを ``FINISHED:EXECUTION_EXCEPTION`` としてマークします。
クラスタのスケジューリングイベントを確認してください。

.. code-block:: bash

   kubectl -n "$NAMESPACE" get pods
   kubectl -n "$NAMESPACE" describe pod <job-pod-name>
   kubectl -n "$NAMESPACE" get events --sort-by=.lastTimestamp

Common causes include insufficient CPU, memory, GPU, or ephemeral storage;
missing study-data PVCs; image pull failures; and missing GPU device-plugin
resources.

Job pod cannot pull its image
-----------------------------

Job pods use the image from the submitted job's ``launcher_spec`` and set
``imagePullPolicy: Always``. Confirm the job image name and configure registry
credentials for dynamically launched pods. Use
``job_launcher.image_pull_secrets`` in ``k8s.yaml`` for explicit Secret
references, or rely on node-level credentials or the namespace default
ServiceAccount if your cluster is configured that way.

Client cannot connect to the server
-----------------------------------

Verify these items:

* ``default_host`` in ``project.yml`` matches the DNS name used by the client.
* The DNS name resolves from the client cluster.
* The server cluster exposes ``fed_learn_port``.
* The server certificate includes the DNS name in ``host_names``.
* Network policy and firewalls allow outbound client traffic to the server.

Run a DNS check from the client cluster:

.. code-block:: bash

   kubectl -n "$NAMESPACE" run dns-test --rm -it \
       --image=busybox:1.36 -- \
       nslookup server1.example.com

If you change ``default_host`` or ``host_names``, reprovision, restage the
updated folders, and redeploy the charts.

Uninstall
=========

To stop a participant installed by Helm:

.. code-block:: bash

   helm uninstall server -n "$NAMESPACE"
   helm uninstall site-1 -n "$NAMESPACE"

If the participants used ConfigMap/Secret staging, remove those objects after
uninstalling the Helm releases. They are not owned by Helm, and the startup
Secret contains the participant's identity keys and certificates:

.. code-block:: bash

   nvflare deploy k8s unstage ./server-k8s --namespace "$NAMESPACE"
   nvflare deploy k8s unstage ./site-1-k8s --namespace "$NAMESPACE"

Delete the namespace only if it is dedicated to this deployment:

.. code-block:: bash

   kubectl delete namespace "$NAMESPACE"

Depending on the storage class reclaim policy, PVC-backed volumes may remain
after deleting Helm releases or namespaces. Remove retained volumes only after
confirming that the startup kits, logs, snapshots, job history, and study data
no longer need to be preserved.
