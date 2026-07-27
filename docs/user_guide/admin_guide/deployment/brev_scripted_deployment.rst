.. _brev_scripted_deployment:

#########################################
Brev スクリプトデプロイのクイックスタート
#########################################

このガイドでは、Brev のヘルパースクリプトを使用して、既存の 3 つの Brev シングルノード
Kubernetes 環境上に 1 つの FLARE サーバーと 2 つの FLARE クライアントをデプロイするための
最短の手順を示します。

``project.yml`` の詳細、PVC のステージング、Helm チャートの挙動、トラブルシューティングを
含む完全な技術ワークフローについては、:ref:`brev_deployment` を参照してください。

スクリプト:

* :download:`prepare_brev_startup_kits.sh <brev_scripts/prepare_brev_startup_kits.sh>`
* :download:`launch_brev_nvflare.sh <brev_scripts/launch_brev_nvflare.sh>`

スクリプトの動作内容
====================

``prepare_brev_startup_kits.sh`` はローカルのワークステーション上で実行します。このスクリプトは
次の処理を行います。

* ``project.yml`` を指定しない限り、シンプルな 3 参加者構成の ``project.yml`` を作成します。
* ``nvflare provision`` を実行します。
* 生成された ``server``、``site-1``、``site-2`` のスタートアップキットに対して、K8s 向けの
  ``nvflare deploy prepare`` を実行します。
* 準備済みの参加者キットをパッケージ化します。
* ``brev copy`` を使用して、対応するアーカイブと起動スクリプトを各 Brev 環境にコピーします。

``launch_brev_nvflare.sh`` は各 Brev 環境の内部で実行します。このスクリプトは次の処理を行います。

* コピーされた参加者アーカイブを展開します。
* ``nvflare deploy prepare`` が Kubernetes ランチャーを設定済みであることを検証します
  (起動先と同じ ``namespace`` を持つ、想定される ``ServerK8sJobLauncher`` または
  ``ClientK8sJobLauncher`` コンポーネント)。
* 準備済みチャートから ``persistence.workspace.claimName`` を読み取り、起動時に指定された
  ``WORKSPACE_PVC`` が矛盾する場合はそれを拒否します。
* ``nvflare`` ネームスペースと、ワークスペース/データ用の PVC を作成します。
* チャートが ``workspace_mount_path`` でそれらを見つけられるように、準備済みの ``startup/`` と
  ``local/`` ディレクトリをワークスペース PVC のルートにコピーします。
* クライアント参加者については、ネームスペース内の一時的な Pod から ``SERVER_HOST`` の DNS
  ルックアップを実行します。
* 生成された Helm チャートをインストールまたはアップグレードします。
* 参加者のデプロイメントを待機し、直近の Pod ログを出力します。

想定される結果
==============

prepare スクリプトが完了すると、各 Brev 環境には次のファイルが存在するはずです。

* ``/home/ubuntu/nvflare-server.tgz``、``/home/ubuntu/nvflare-site-1.tgz``、または
  ``/home/ubuntu/nvflare-site-2.tgz``
* ``/home/ubuntu/launch_brev_nvflare.sh``

各環境で launch スクリプトが完了すると、次のようになります。

* ``kubectl -n nvflare get pvc`` で PVC が ``Bound`` と表示されるはずです。
* ``helm list -n nvflare`` でその参加者のリリースが 1 つ表示されるはずです。
* ``kubectl -n nvflare get pods`` で参加者の Pod が実行中と表示されるはずです。
* ``site-1`` と ``site-2`` のログには、サーバーのエンドポイントへ接続していることが表示される
  はずです。

必要なもの
==========

スクリプトを実行する前に、次のものを用意してください。

* ``server``、``site-1``、``site-2`` 用に稼働中の 3 つの Brev Kubernetes 環境
* 両方のサイトから到達可能な、サーバーの DNS 名または IP アドレス
* 3 つの環境すべてが pull できる NVFlare の親コンテナイメージ
* 3 つの環境すべてが pull できる PyTorch NVFlare ジョブイメージ
* ローカルのワークステーションにインストール済みで、ログイン済みの Brev CLI

キットの準備とコピー
====================

ローカルの NVFlare チェックアウトから、サーバーホスト、親イメージ、PyTorch ワークロード
イメージを設定します。メンテナンスされている Dockerfile で 2 つのイメージをビルドできます。
3 つの環境すべてが pull できるレジストリにプッシュしてください。

.. code-block:: shell

   export SERVER_HOST=server1.example.com
   export IMAGE=registry.example.com/nvflare-parent:dev
   export JOB_IMAGE=registry.example.com/nvflare-job:dev

   docker build -t "$IMAGE" -f docker/Dockerfile.parent .
   docker build -t "$JOB_IMAGE" -f docker/Dockerfile.job .
   docker push "$IMAGE"
   docker push "$JOB_IMAGE"

   bash docs/user_guide/admin_guide/deployment/brev_scripts/prepare_brev_startup_kits.sh

``IMAGE`` は長時間稼働する親プロセスを実行し、Kubernetes ランチャーの依存関係を含みます。
``JOB_IMAGE`` は投入された CIFAR-10 のワークロードを実行し、PyTorch と torchvision を
含みます。両方の要件を満たすカスタムイメージである場合に限り、これらは同一のイメージでも
構いません。

デフォルトでは、スクリプトは ``server``、``site-1``、``site-2`` という名前の Brev 環境に
キットをコピーします。

Brev 環境の名前が異なる場合は、実行前に次のように設定してください。

.. code-block:: shell

   export SERVER_BREV=nvflare-server-k8s
   export SITE_1_BREV=nvflare-site-1-k8s
   export SITE_2_BREV=nvflare-site-2-k8s

   bash docs/user_guide/admin_guide/deployment/brev_scripts/prepare_brev_startup_kits.sh

あるいは、スクリプトに Brev 環境名を対話的に入力させることもできます。

.. code-block:: shell

   bash docs/user_guide/admin_guide/deployment/brev_scripts/prepare_brev_startup_kits.sh \
     --prompt-brev-names

サーバーポートの公開
====================

Brev コンソールでサーバー環境の Access ページを開き、TCP ポート ``8002`` を公開します。
``SERVER_HOST`` が公開されたサーバーホストに解決されることを確認してください。
``SERVER_HOST`` に ``:8002`` を含めないでください。

各環境の起動
============

各 Brev 環境の内部で launch スクリプトを実行します。

サーバー:

.. code-block:: shell

   brev shell "${SERVER_BREV:-server}"
   IMAGE="$IMAGE" bash /home/ubuntu/launch_brev_nvflare.sh server

サイト 1:

.. code-block:: shell

   brev shell "${SITE_1_BREV:-site-1}"
   IMAGE="$IMAGE" SERVER_HOST="$SERVER_HOST" bash /home/ubuntu/launch_brev_nvflare.sh site-1

サイト 2:

.. code-block:: shell

   brev shell "${SITE_2_BREV:-site-2}"
   IMAGE="$IMAGE" SERVER_HOST="$SERVER_HOST" bash /home/ubuntu/launch_brev_nvflare.sh site-2

launch スクリプトは、準備済みの Kubernetes ランチャー設定と準備済みチャートのワークスペース
PVC 名を検証し、ネームスペースと PVC を作成し、チャートが ``workspace_mount_path`` で見つけ
られるように ``startup/`` と ``local/`` をワークスペース PVC のルートにステージングし、Helm
チャートをインストールし、直近の Pod ログを出力します。

Recipe ジョブの投入
===================

3 つの参加者すべてが稼働したら、スタートアップキットの準備に使用したのと同じローカルの
NVFlare チェックアウトから、PyTorch CIFAR-10 の Recipe サンプルを投入します。最新の管理者用
startup ディレクトリを探し、3 つの Brev クラスタすべてが pull できるワークロードイメージを
渡してください。

.. code-block:: shell

   ADMIN_KIT="$(find /tmp/nvflare/brev-provision/brev_nvflare_project \
     -path '*/admin@nvidia.com/startup' -type d | sort | tail -n 1)"

   cd examples/advanced/recipe-k8s
   python3 job.py \
     --startup-kit "$ADMIN_KIT" \
     --image "$JOB_IMAGE" \
     --server-image "$JOB_IMAGE"

このクイックスタートでは、両方のクライアントを ``ClientK8sJobLauncher`` で準備するのに加えて、
サーバーも ``ServerK8sJobLauncher`` で準備するため、ここでは ``--server-image`` 引数が必要です。
このサンプルは 2 つのクライアントを明示的に対象とし、``set_recipe_meta`` を使用して、
スケジューラ向けの ``resource_spec`` と Kubernetes の ``launcher_spec`` エントリを追加します。
:github_nvflare_link:`完全なサンプルとメタデータの説明 <examples/advanced/recipe-k8s>` を参照して
ください。

便利なオーバーライド
====================

よく使われるオプションの環境変数は次のとおりです。

``NAMESPACE``、``PARENT_PORT``、``WORKSPACE_PVC``、``WORKSPACE_MOUNT_PATH`` は deploy prepare の
入力であり、準備されるリソースおよび ``helm_chart/`` に埋め込まれます。これらは
``prepare_brev_startup_kits.sh`` の実行時に設定してください。起動時にのみ変更してはいけません。
準備済みキットが起動環境と一致するように、まず prepare スクリプトを再実行してください。

* ``NAMESPACE``: Kubernetes のネームスペース。デフォルトは ``nvflare`` です。両方のスクリプトで
  同じ値を使用してください。launch スクリプトは準備済みの ``resources.json.default`` を読み取り、
  起動時の ``NAMESPACE`` が準備済みランチャーと一致しない場合は起動を拒否します。
* ``DATA_PVC``: オプションのジョブデータ PVC 名。デフォルトは ``nvfldata`` です。スタディデータ用
  の追加 PVC を作成するために、起動時にのみ使用されます。
* ``CLEAN_WORKSPACE_PVC=true``: 新しいキットをステージングする前に、古いワークスペース PVC の
  内容を消去します。

prepare 専用のオーバーライド:

* ``WORKSPACE_PVC``: ワークスペース PVC 名。デフォルトは ``nvflws`` です。prepare スクリプトは
  この値を生成される Helm チャートの ``persistence.workspace.claimName`` に書き込みます。
  launch スクリプトは準備済みチャートからその値を読み取り、PVC の作成とステージングに使用します。
  起動時にも ``WORKSPACE_PVC`` を設定し、それが準備済みチャートと一致しない場合、チャート、PVC、
  ステージングされた内容の整合性を保つために launch スクリプトは失敗します。
* ``PARENT_PORT``: ``nvflare deploy prepare`` によって書き込まれる、クラスタ内の親サービスポート。
  デフォルトは ``8102`` です。launch スクリプトは準備済みの Helm チャートとランチャー設定を読み
  取ります。この変数は読み取りません。
* ``WORKSPACE_MOUNT_PATH``: ``nvflare deploy prepare`` によって書き込まれる、親 Pod およびジョブ
  Pod のワークスペースパス。デフォルトは ``/var/tmp/nvflare/workspace`` です。launch スクリプトは
  ``resources.json.default`` から準備済みの値を検証します。この変数は読み取りません。

現在のオプションを確認するには、いずれかのスクリプトを ``--help`` 付きで実行してください。
