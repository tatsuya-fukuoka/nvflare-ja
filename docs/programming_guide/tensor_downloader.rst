.. _tensor_downloader:

##################################
FLARE Tensor Downloader
##################################

このガイドでは、フェデレーテッドラーニングのワークフローにおいて大規模な PyTorch モデルを
メモリ効率よく転送するための、NVIDIA FLARE の Tensor Downloader 機能について説明します。

.. contents:: 目次
   :local:
   :depth: 2

概要
====

Tensor Downloader とは何か
--------------------------

Tensor Downloader は、FL サーバーとクライアントの間で大規模な PyTorch テンソル (モデルパラメータ) を
効率的に転送できるようにするメモリ最適化機能です。送信前にモデル全体をメモリ上でシリアライズするのではなく、
テンソルを逐次的にストリーミングすることで、ピーク時のメモリ使用量を大幅に削減します。

なぜ必要なのか
--------------

従来のフェデレーテッドラーニングでは、サーバーがグローバルモデルをクライアントに送信するとき (あるいは
クライアントが更新を返送するとき)、モデル全体を次のように扱う必要があります。

1. **メモリ上でのシリアライズ** - モデルをバイト列に変換するには、モデルサイズと同等以上の追加メモリが必要になります
2. **送信中のメモリ保持** - シリアライズされたバイト列は、送信が完了するまでメモリ上に残しておく必要があります
3. **複数の受信者分だけ倍増** - N 個のクライアントに同時送信する場合、メモリ圧迫が劇的に増大します

大規模言語モデル (LLM) やその他の大規模モデルでは、これにより次のような問題が発生し得ます。

- 利用可能な RAM が不足した場合の **メモリ不足エラー**
- メモリが飽和した場合の **深刻な性能低下**
- 他のプロセスに影響を及ぼす **システムの不安定化**

Tensor Downloader は、**プルベースの逐次ストリーミング** 方式を用いることでこれらの問題を解決します。

主な利点
--------

- **メモリフットプリントの削減**: サーバー側・クライアント側ともにメモリ使用量が 20〜50% 削減されます
  (5GB のモデルと 4 クライアントで FedAvg を用いたテストに基づく)

- **コード変更が不要**: この最適化は PyTorch ワークフローに組み込まれており、既存の学習コードで
  自動的に動作します

- **複数クライアントへのスケーラビリティ**: 各クライアントは他のクライアントをブロックすることなく、
  自分のペースでダウンロードします

- **セキュアなシリアライズ**: pickle ベースのセキュリティ脆弱性を回避する `safetensors` 形式を使用します

- **信頼性の高い転送**: プルベースのアーキテクチャにより、多様なネットワーク状況にうまく対応します

制限事項
--------

- **PyTorch と NumPy のみ**: ストリーミングダウンロード機能は PyTorch テンソルと NumPy 配列をサポートします。
  TensorFlow モデルは現在サポートされておらず、従来のシリアライズが使用されます。

- **カスタムテンソル型**: カスタムテンソル型や非標準のモデル形式は直接サポートされていません。
  ストリーミングダウンロード機能の恩恵を受けるには、カスタムテンソルを PyTorch テンソル (``torch.Tensor``)
  または NumPy 配列 (``numpy.ndarray``) に変換してください。


使い方 (ユーザーの視点)
=======================

一般的なユーザーの場合
----------------------

**朗報です。何もする必要はありません!**

Tensor Downloader は FLARE 2.7.2 以降のすべての PyTorch ワークフローに組み込まれています。以下を使う場合が該当します。

- ``PTFedAvg`` コントローラー
- ``PTFileModelPersistor``
- ``PTClientAPILauncherExecutor``
- ``PTInProcessClientAPIExecutor``
- PyTorch ベースの任意の Recipe (``nvflare.app_opt.pt.recipes`` の ``FedAvgRecipe``)

TensorDecomposer は自動的に登録され、テンソルのストリーミングを透過的に処理します。

大規模モデルにおけるクライアントメモリに関する注意
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

TensorDownloader は転送時のメモリ圧迫を軽減します。また、``clear_cache=True`` (デフォルト) の場合、
クライアント側のパラメータ参照も ``flare.send()`` の後に解放されます。CPython では、テンソルは通常、
最後の参照が破棄されるとすぐに回収されます。

数 GB のペイロードでは、必要以上に長く余分な参照を保持しないようにしてください。

.. code-block:: python

    import nvflare.client as flare

    flare.init()
    while flare.is_running():
        input_model = flare.receive()
        output_model = train(input_model)
        flare.send(output_model)  # clear_cache=True by default

        # Optional: release script-local references promptly.
        del input_model
        del output_model

``gc.collect()`` は循環参照オブジェクトに対する補助的な安全策として引き続き有効ですが、
このフローにおいてテンソルメモリを解放する主要な仕組みではありません。

例: PyTorch FedAvg Recipe を使う
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from nvflare.app_opt.pt.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    # TensorDownloader is automatically used - no configuration needed
    # Model can be class instance or dict config
    # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
    recipe = FedAvgRecipe(
        name="my-fedavg-job",
        min_clients=2,
        num_rounds=10,
        model=MyLargeModel(),  # Even multi-GB models work efficiently
        train_script="client.py",
    )

    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

例: PTFedAvg コントローラーを直接使う
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    from nvflare import FedJob
    from nvflare.app_opt.pt.fedavg import PTFedAvg

    job = FedJob(name="pt-fedavg")

    # TensorDownloader is automatically enabled
    # Model can be class instance or dict config
    controller = PTFedAvg(
        num_clients=2,
        num_rounds=10,
        model=MyLargeModel(),
    )
    job.to(controller, "server")

設定
----

Tensor Downloader の動作は、ジョブ設定ファイル内のチャンクサイズ設定を通じて構成できます。

**設定パラメータ:**

- ``tensor_download_chunk_size``: PyTorch テンソルのダウンロードのチャンクサイズ (デフォルト: 2097152 = 2MB)
- ``np_download_chunk_size``: NumPy 配列のダウンロードのチャンクサイズ (デフォルト: 2097152 = 2MB)

Recipe API を使う (推奨)
^^^^^^^^^^^^^^^^^^^^^^^^

Recipe を使って作業しているユーザーは、``add_server_config()`` メソッドを使用してください。

.. code-block:: python

    from nvflare.recipe.fedavg import FedAvgRecipe

    recipe = FedAvgRecipe(
        name="my_job",
        num_rounds=10,
        min_clients=2,
        train_script="train.py",
    )

    # Configure chunk sizes and streaming timeout (server-side only)
    recipe.add_server_config({
        "np_download_chunk_size": 2097152,
        "tensor_download_chunk_size": 2097152,
        "streaming_per_request_timeout": 600
    })

Job API を使う
^^^^^^^^^^^^^^

Job API を直接使って作業しているユーザーの場合は次のとおりです。

.. code-block:: python

    from nvflare import FedJob

    job = FedJob(name="my_job")

    # Add config to server (these are server-side only settings)
    job.to_server({
        "np_download_chunk_size": 2097152,
        "tensor_download_chunk_size": 2097152,
        "streaming_per_request_timeout": 600
    })

大規模モデル向けのチューニング
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

非常に大規模なモデル (数 GB) では、最適な性能を得るためにチャンクサイズをチューニングしたい場合があります。
チャンクを大きくするとネットワークリクエスト数は減りますが、チャンクあたりのメモリ使用量は増加します。
チャンクを小さくするとメモリは削減されますが、ネットワークのオーバーヘッドが増加します。

サブプロセスモードの Client API ジョブからテンソルストリーミングを使用する場合は、タスクの読み取り、
結果の ACK、サーバー側のダウンロード完了を制御するサブプロセスのタイムアウト設定もチューニングしてください。
特に、``PEER_READ_TIMEOUT``、``download_complete_timeout``、``tensor_min_download_timeout`` を、
設定したストリーミングのリクエストごとのタイムアウトと整合させてください。:ref:`timeout_troubleshooting`
および :doc:`/programming_guide/timeouts` を参照してください。

**チャンクサイズをチューニングした config_fed_server.conf の例:**

.. code-block::

    format_version = 2

    # Chunk sizes for streaming large models (2MB default)
    np_download_chunk_size = 2097152
    tensor_download_chunk_size = 2097152
    streaming_per_request_timeout = 600

    task_data_filters = []
    task_result_filters = []

    components = [
      {
        id = "json_generator"
        path = "nvflare.app_common.widgets.validation_json_generator.ValidationJsonGenerator"
        args {}
      }
    ]

    workflows = [
      {
        id = "swarm_controller"
        path = "nvflare.app_common.ccwf.SwarmServerController"
        args {
          num_rounds = 3
          # Increased timeouts to accommodate large LLM payload init/broadcast
          start_task_timeout = 300
          progress_timeout = 7200
        }
      }
      {
        id = "cross_site_eval"
        path = "nvflare.app_common.ccwf.CrossSiteEvalServerController"
        args {
          eval_task_timeout = 1200
        }
      }
    ]

Tensor Downloader を無効化する
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ストリーミングダウンロード機能を無効にして、代わりに従来のシリアライズを使用したい場合は、
チャンクサイズをゼロに設定してください。

**Recipe API を使う場合:**

.. code-block:: python

    # Disable streaming (server-side setting)
    recipe.add_server_config({
        "np_download_chunk_size": 0,
        "tensor_download_chunk_size": 0
    })

**Job API を使う場合:**

.. code-block:: python

    job.to_server({"np_download_chunk_size": 0, "tensor_download_chunk_size": 0})

**設定ファイルを直接使う場合:**

.. code-block::

    format_version = 2

    # Set to 0 to disable streaming download (use native serialization)
    np_download_chunk_size = 0
    tensor_download_chunk_size = 0

    task_data_filters = []
    task_result_filters = []

    # ... rest of configuration


仕組み (上級ユーザー向け)
=========================

このセクションでは、Tensor Downloader の機能を理解または拡張したい開発者向けに、内部アーキテクチャを説明します。

アーキテクチャの概要
--------------------

Tensor Downloader は複数のコンポーネントで構成されています。

1. **TensorDecomposer**: PyTorch テンソルのシリアライズを扱う FOBS の decomposer
2. **TensorDownloadable**: 逐次ダウンロードの準備が整ったテンソルのコレクションを表現します
3. **TensorConsumer**: 受信側でダウンロードされたテンソルチャンクを処理します
4. **Download Service**: プルベースのダウンロードプロトコルを管理します

プルベース転送とプッシュベース転送
----------------------------------

従来型 (プッシュベース):

.. code-block:: text

    Server                           Client
      |                                 |
      |  [Serialize entire model]       |
      |  [Hold in memory]               |
      |-------- Full Model ------------>|
      |                                 |  [Deserialize]

Tensor Downloader (プルベース):

.. code-block:: text

    Server                           Client
      |                                 |
      |  [Prepare reference ID]         |
      |-------- Reference ID ---------->|
      |                                 |
      |<------- Request chunk 1 --------|
      |  [Serialize chunk 1 only]       |
      |-------- Chunk 1 --------------->|
      |                                 |
      |<------- Request chunk 2 --------|
      |  [Serialize chunk 2 only]       |
      |-------- Chunk 2 --------------->|
      |           ...                   |
      |                                 |  [Reassemble model]

シリアライズの流れ
------------------

1. **登録**: PyTorch コンポーネントが初期化されるとき、``TensorDecomposer`` を FOBS に登録します。

   .. code-block:: python

       from nvflare.app_opt.pt.decomposers import TensorDecomposer
       from nvflare.fuel.utils import fobs

       fobs.register(TensorDecomposer)

2. **テンソルの収集**: シリアライズ中に、FOBS がペイロード内のすべてのテンソルを辞書に収集します。

3. **Downloadable の生成**: テンソルは ``TensorDownloadable`` オブジェクトにラップされます。

   .. code-block:: python

       class TensorDownloadable(CacheableObject):
           def __init__(self, tensors: dict[str, torch.Tensor], max_chunk_size: int):
               self.keys = list(tensors.keys())
               super().__init__(tensors, max_chunk_size)

           def produce_item(self, index: int) -> bytes:
               key = self.keys[index]
               tensor_to_send = {key: self.base_obj[key]}
               return save_tensors(tensor_to_send)  # safetensors format

4. **参照 ID の生成**: 一意の参照 ID (RID) が生成され、実際のテンソルの代わりに受信者に送信されます。

5. **逐次ダウンロード**: 各受信者は RID を使って、テンソルを 1 つずつリクエストします。

TensorDecomposer
----------------

``TensorDecomposer`` は ``ViaDownloaderDecomposer`` を拡張したもので、次の機能を提供します。

.. code-block:: python

    class TensorDecomposer(ViaDownloaderDecomposer):

        def supported_type(self):
            return torch.Tensor

        def to_downloadable(self, items: dict, max_chunk_size: int, fobs_ctx: dict):
            return TensorDownloadable(items, max_chunk_size)

        def download(self, from_fqcn, ref_id, per_request_timeout, cell, ...):
            return download_tensors(from_fqcn, ref_id, per_request_timeout, cell, ...)

        def native_decompose(self, target: torch.Tensor, manager=None) -> bytes:
            # Fallback: serialize single tensor using safetensors
            return save({"t": target})

        def native_recompose(self, data: bytes, manager=None) -> torch.Tensor:
            # Fallback: deserialize single tensor
            return load(data).get("t")

低レベル API を使う
-------------------

高度なユースケースでは、テンソルダウンロード API を直接使用できます。

.. code-block:: python

    from nvflare.app_opt.pt.tensor_downloader import add_tensors, download_tensors

    # Server side: Register tensors for download
    ref_id = add_tensors(
        downloader=downloader,
        tensors=model.state_dict(),
        max_chunk_size=2 * 1024 * 1024,  # 2MB chunks
    )

    # Send ref_id to clients via your preferred mechanism
    # ...

    # Client side: Download tensors incrementally
    status, state_dict = download_tensors(
        from_fqcn=server_fqcn,
        ref_id=ref_id,
        per_request_timeout=30.0,
        cell=cell,
        secure=False,
        optional=False,
        abort_signal=abort_signal,
    )

    # Load into model
    model.load_state_dict(state_dict)

参考情報
========

- :ref:`decomposer_for_large_object` - FOBS の decomposer システムとファイルベースの decomposer の詳細
- :ref:`file_streaming` - その他の大規模データ型向けのファイルストリーミング
- :ref:`swarm_learning_large_models` - 大規模モデルのワークフロー向けのパラメータチューニング
- :ref:`timeout_troubleshooting` - 大規模な Client API サブプロセスジョブ向けのタイムアウトチューニング
