.. _containerized_deployment:

####################################################
コンテナ化デプロイメント
####################################################

Docker によるコンテナ化デプロイメント
================================================================

Docker サポートには 2 つの一般的な用途があります:

- シミュレーション、ノートブック、POCモード、または手動での実験のために、汎用の開発コンテナ内で
  FLARE を実行する。このパターンでは、Docker は Python 環境とファイルシステムを提供し、
  FLARE のプロセスはその 1 つのコンテナ内で通常どおり実行されます。
- プロビジョニング済みのスタートアップキットを Docker ランタイム実行用に準備する。これは、
  FLARE の親サーバー/クライアントプロセスを Docker 内で実行し、サーバー/クライアントのジョブを
  個別の Docker コンテナとして起動する場合に推奨されるパスです。

現行の Docker ランタイムのワークフローは次のとおりです:

1. NVFlare をインストールした親イメージをビルドする。
2. NVFlare とワークロードの依存関係をインストールしたジョブイメージをビルドする。
3. サーバー、クライアント、管理者のスタートアップキットをプロビジョニングする。
4. Docker で実行するサーバーまたはクライアントの各スタートアップキットに対して
   :ref:`deploy_prepare_command` を実行する。
5. 準備したサーバー/クライアントキットを ``startup/start_docker.sh`` で起動する。
6. ``meta.json`` に :ref:`launcher_spec` の Docker 設定を含むジョブを
   投入する。

実行可能なエンドツーエンドのワークフローについては、
:github_nvflare_link:`Docker ジョブランチャーの例 <examples/docker>` を参照してください。

前提条件
----------------
コンテナ化デプロイメントを始める前に、以下を確認してください:

1. システムに Docker がインストールされていること
2. GPU サポートのために NVIDIA Container Toolkit がインストールされていること
3. `NVIDIA Container Toolkit Install Guide <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html>`_ に従ってシステム要件を満たしていること
4. Docker ランタイムデプロイメントを準備する場合は、プロビジョニング済みの
   サーバー/クライアントのスタートアップキットがあること

NVIDIA FLARE を Docker コンテナ内で実行することには、いくつかの利点があります:

- 異なるシステム間で一貫した環境
- 容易な依存関係管理
- 再現可能なランタイム準備
- 分離された実行環境
- NVIDIA Container Toolkit による GPU サポート

親イメージとジョブイメージ
--------------------------------------------------

Docker ランタイムでは、親コンテナとジョブコンテナが分離されています:

- 親イメージは、準備されたスタートアップキットから、長時間稼働する FLARE サーバープロセス
  またはクライアントプロセスを実行します。
- ジョブイメージは、``DockerJobLauncher`` によって起動されるサーバージョブおよび
  クライアントジョブのプロセスを実行します。

親イメージは、``nvflare deploy prepare`` が使用するランタイム ``docker.yaml`` で設定します。
ジョブイメージは、投入するジョブの ``meta.json`` の ``launcher_spec`` で設定します。
両方の役割に同じイメージを使うこともできますが、分けておく方がすっきりすることが多いです。
親イメージには FLARE ランタイムと Docker SDK へのアクセスが必要であり、
ジョブイメージにはワークロードのフレームワークとトレーニングの依存関係が必要です。

Docker ランタイムワークフロー
------------------------------------------------------------

各サイトが使用するイメージをビルドまたは公開します。実行可能な
:github_nvflare_link:`Docker ジョブランチャーの例 <examples/docker>` は、
例として親イメージ ``nvflare-site:latest`` とジョブイメージ
``nvflare-job:latest`` をビルドします:

.. code-block:: shell

  cd examples/docker
  bash build_docker.sh

プロジェクトのプロビジョニング後:

.. code-block:: shell

  nvflare provision -p project.yml

``nvflare deploy prepare`` 用の Docker ランタイム設定を作成します:

.. code-block:: yaml

  runtime: docker

  parent:
    docker_image: nvflare-site:latest
    network: nvflare-network

  job_launcher:
    default_python_path: /usr/local/bin/python
    default_job_env:
      NCCL_P2P_DISABLE: "1"
    default_job_container_kwargs:
      shm_size: 8g
      ipc_mode: host

Docker で実行するサーバーまたはクライアントの各スタートアップキットを準備します:

.. code-block:: shell

  nvflare deploy prepare workspace/<project>/prod_00/server \
    --config docker.yaml \
    --output workspace/<project>/prepared/server

  nvflare deploy prepare workspace/<project>/prod_00/site-1 \
    --config docker.yaml \
    --output workspace/<project>/prepared/site-1

準備されたキットには、``startup/start_docker.sh``、パッチ適用済みのランチャー設定、
および ``local/study_runtime.yaml`` テンプレートが含まれます。管理者のスタートアップキットは、
親のサーバープロセスやクライアントプロセスを実行しないため、準備の対象外です。

準備した親プロセスは次のように起動します:

.. code-block:: shell

  cd workspace/<project>/prepared/server
  bash startup/start_docker.sh

準備した各クライアントキットからも同じコマンドを実行します。生成されたスクリプトは、
必要に応じて設定された Docker ネットワークを作成し、準備されたキットを
親コンテナにマウントします。

Docker モードのサイトに投入するジョブは、``launcher_spec`` でジョブイメージを
指定する必要があります:

.. code-block:: json

  {
    "launcher_spec": {
      "default": {
        "docker": {"image": "nvflare-job:latest"}
      },
      "site-1": {
        "docker": {"shm_size": "8g", "ipc_mode": "host"}
      }
    },
    "resource_spec": {
      "site-1": {"num_of_gpus": 1}
    }
  }

ランチャー固有のイメージとコンテナ設定には ``launcher_spec`` を使用してください。
``num_of_gpus`` のようなスケジューラー向けのリソース要求は ``resource_spec`` に
記述してください。Docker ジョブランチャーが設定されていないサイトは、
引き続き設定済みのランチャー(通常はプロセスモード)を使用します。

開発コンテナ
------------------------

単一コンテナのイメージは、ローカル開発やシミュレーターでの作業にも役立ちます。この場合、
Docker は NVFlare と開発用の依存関係をインストールした Python 環境を提供します。これは、
ベアメタルの Python 仮想環境の代替であり、Docker ジョブランチャーを使用する場合の
``nvflare deploy prepare`` の代替ではありません。

この単一コンテナのパターンを使えば、開発のためにすべての FLARE プロセスを 1 つのホストで
実行できます。シミュレーターの実行、POCモードの実行、または同じコンテナ内の別々のシェルからの
プロビジョニング済みサーバー/クライアントスクリプトの手動起動が可能です。
これは Docker ランタイムデプロイメントとは異なります。サーバーまたはクライアントの
スタートアップキットが ``nvflare deploy prepare`` で準備されていない限り、
ジョブランチャーはプロセスモードのままです。

環境に合わせたイメージをビルドして ``nvflare-dev:latest`` というタグを付けたら、
GPU サポートと永続的なワークスペースを有効にして実行します:

.. code-block:: shell

  mkdir my-workspace
  docker run --rm -it --gpus all \
      --ipc=host --ulimit memlock=-1 --ulimit stack=67108864 \
      -v $(pwd -P)/my-workspace:/workspace/my-workspace \
      nvflare-dev:latest

コンテナの実行後、たとえば追加の FLARE クライアントを起動するために別のターミナルが必要な場合など、
コンテナに exec で入ることもできます。まず ``docker ps`` で ``CONTAINER ID`` を確認し、
その ID を使ってコンテナに exec で入ります:

.. code-block:: shell

  docker ps  # use the CONTAINER ID in the output
  docker exec -it <CONTAINER ID> /bin/bash

ベストプラクティス
------------------------------------

1. 常に互換性のある最新の NVIDIA Container Toolkit バージョンを使用する
2. Docker ランタイム用のスタートアップキットには ``nvflare deploy prepare`` を使用する
3. 親イメージとジョブイメージの責務を明確に保つ
4. Docker のイメージおよびコンテナ設定は ``launcher_spec`` に記述する
5. スケジューラー向けのリソース要求は ``resource_spec`` に記述する
6. 永続的なデータ保存のためにボリュームをマウントする
7. セキュリティパッチのためにベースイメージを最新に保つ

.. note::

   注記: Docker Compose によるデプロイメントは非推奨です。現行の Docker ランタイム準備には
   ``nvflare deploy prepare`` を使用してください。

よくある問題と解決策
----------------------------------------

1. Docker デーモンへのアクセス: ``start_docker.sh`` を実行するユーザーが Docker デーモンに
   アクセスできることを確認してください。
2. ジョブイメージの欠如: Docker モードのすべてのジョブが
   ``launcher_spec[site]["docker"]["image"]`` または
   ``launcher_spec["default"]["docker"]["image"]`` を提供していることを確認してください。
3. GPU アクセスの問題: NVIDIA Container Toolkit が正しくインストールされており、
   ジョブの ``resource_spec`` が必要な GPU を要求していることを確認してください。
4. メモリまたは共有メモリの問題: ``shm_size`` や ``ipc_mode`` などのコンテナ kwargs を、
   ``launcher_spec`` または deploy prepare の
   ``default_job_container_kwargs`` で設定してください。
5. ネットワーク接続: 設定された Docker ネットワークとサーバーのホスト名が、
   管理者、親、ジョブの各コンテナから解決可能な状態を維持してください。
