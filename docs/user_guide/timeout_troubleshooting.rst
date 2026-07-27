.. _timeout_troubleshooting:

##################################################
タイムアウトのトラブルシューティングガイド
##################################################

本ガイドでは、最もよく発生するタイムアウト関連のジョブ失敗と、その解決方法について説明します。
すべてのタイムアウトの包括的なリファレンスについては、 :ref:`timeouts_programming_guide` を参照してください。

.. contents:: 目次
   :local:
   :depth: 2

よくあるジョブ失敗のシナリオ
============================

タスク取得タイムアウト
----------------------

**症状** : クライアントがサーバーからタスクを受信できず、タスク取得中に "timeout" がログに出力されます。

**よくある原因** :

- 大きなモデルの重みの転送に時間がかかりすぎている
- ネットワークのレイテンシがデフォルトのタイムアウトを超えている
- テンソルストリーミングのタイムアウトがタスク取得タイムアウトを超えている

**解決策** : クライアント設定で ``get_task_timeout`` を設定します。

.. code-block:: python

   recipe.add_client_config({
       "get_task_timeout": 300,  # 5 minutes
   })


外部プロセスの初期化前タイムアウト (Client API のみ)
------------------------------------------------------

**対象** : サブプロセスランチャーを使う Client API ( ``ScriptRunner`` 、 ``ClientAPILauncherExecutor`` )

**症状** : 学習が始まる前に "external_pre_init_timeout" エラーでジョブが失敗します。

このタイムアウトは、外部の学習スクリプトが ``flare.init()`` を呼び出すのを NVFlare がどれだけ待つかを制御します。
Client API を使用する場合、NVFlare はスクリプトをサブプロセスとして起動し、接続が戻ってくるのを待ちます。

**よくある原因** :

- 大規模モデル (LLM) では ``flare.init()`` が呼ばれる前のロードに時間がかかる
- 重いライブラリのインポート (PyTorch、TensorFlow、transformers)
- モデルの重みを読み込むディスク I/O が遅い

**解決策** : エグゼキューターの設定で ``external_pre_init_timeout`` を増やします。

.. code-block:: python

   from nvflare.app_common.executors.client_api_launcher_executor import ClientAPILauncherExecutor

   executor = ClientAPILauncherExecutor(
       external_pre_init_timeout=600,  # 10 minutes for LLMs
       ...
   )


ハートビートタイムアウト
------------------------

**症状** : クライアントが停止したとみなされ、ログに "heartbeat timeout" または "client not responding" が出力されます。

**よくある原因** :

- 長時間実行される学習がハートビートスレッドをブロックしている
- ネットワークの問題によりハートビートが欠落している
- クライアントが計算処理で過負荷になっている

**解決策** : ハートビートの設定を調整します。

.. code-block:: python

   # In executor configuration
   heartbeat_timeout = 300.0   # 5 minutes
   heartbeat_interval = 10.0   # Send every 10 seconds

**ルール** : ``heartbeat_interval`` は ``heartbeat_timeout`` より小さくする必要があります。


学習タスクのタイムアウト
------------------------

**症状** : 学習が完了する前に中断され、ログにタスクのタイムアウトが出力されます。

**よくある原因** :

- 学習ラウンドが想定より長くかかっている
- データのロードが遅い
- ハードウェアが想定より遅い

**解決策** : コントローラーで適切なタスクタイムアウトを設定します。

.. code-block:: python

   # ScatterAndGather controller
   controller = ScatterAndGather(
       train_timeout=7200,  # 2 hours per round
       wait_time_after_min_received=60,
   )

   # Or via ModelController
   controller = FedAvg(
       num_rounds=100,
       timeout=7200,  # 2 hours per round
   )


結果送信タイムアウト
--------------------

**症状** : 学習は完了するものの、結果の送信に失敗します。

**よくある原因** :

- 大きなモデルの結果の転送に時間がかかる
- ネットワークの輻輳

**解決策** : ``submit_task_result_timeout`` を設定します。

.. code-block:: python

   recipe.add_client_config({
       "submit_task_result_timeout": 300,  # 5 minutes
   })


サブプロセスでの大規模モデル結果送信タイムアウト
------------------------------------------------

**対象** : 大規模モデルを扱うサブプロセスモードのクライアント ( ``launch_external_process=True`` )

**症状** : サブプロセス内では学習が完了するものの、その直後にジョブがハングするか失敗し、
結果の受領確認が受信されません。ペイロードが非常に大きく、クライアント数が多い場合には、
遅延したリトライの後に ``DownloadService`` から ``no ref found`` メッセージが繰り返し
ログに出力されることもあります。

**原因** : ``submit_result_timeout`` は、学習サブプロセスがクライアントジョブプロセスによる
結果の受領確認を待つ時間です。 ``PEER_READ_TIMEOUT`` は、親クライアントジョブがサブプロセスによる
タスクの読み取りを待つ、対応する待機時間のためのクライアント設定キーです。大規模モデル (5 GB 以上)
かつクライアント数が多い場合、ストリーミングのリクエストタイムアウトがパイプのタイムアウトより
大きく設定されていると、どちらの側も短いデフォルト値を超える可能性があります。また、結果の ACK 後に
サーバーが ``DownloadService`` からテンソルを取得し終えるまで、サブプロセスは生存し続ける必要があります。

**解決策** :

.. code-block:: python

   recipe.add_client_config({
       "submit_result_timeout": 1800,      # 30 min for LLM-scale results
       "download_complete_timeout": 1800,  # keep subprocess alive for server tensor download
       "PEER_READ_TIMEOUT": 600,           # parent CJ read budget; match configured streaming timeout
       "tensor_min_download_timeout": 600, # PyTorch: increase if inter-chunk gaps exceed 300s default
       # "np_min_download_timeout": 600,   # NumPy/sklearn: same, use instead of tensor variant
       "max_resends": 3,                   # finite value; 0 disables retries, None is rejected
   })

.. note::
   ``submit_result_timeout`` は、サブプロセス側での受領確認の待機時間です。
   これは、クライアントが結果を配信するのをサーバー側で待つ ``submit_task_result_timeout`` とは
   区別されます。大規模モデルの場合、サブプロセスが送信を終えた時点でもサーバーがまだ待ち受けているように、
   ``submit_task_result_timeout`` (サーバー側) を ``submit_result_timeout`` (サブプロセス側) と
   同等以上に設定してください。

.. note::
   FLARE 2.8.0 では、 ``ClientAPILauncherExecutor`` はジョブの初期化時に
   ``download_complete_timeout=None`` および ``max_resends=None`` を拒否します。
   正の ``download_complete_timeout`` と、有限で非負の ``max_resends`` の値を使用してください。
   Recipe ベースの外部プロセスジョブは、エグゼキューターの引数にデフォルトの ``max_resends=3`` を
   シリアライズします。 ``recipe.add_client_config({"max_resends": N})`` は、そのデフォルト値を
   上書きする場合にのみ使用してください。

Swarm Learning の P2P 転送タイムアウト
----------------------------------------

**対象** : 大規模モデルを扱う ``SwarmLearningRecipe``

**症状** : ピア間のモデルスキャッター中に P2P ACK タイムアウトが発生し、Swarm Learning ジョブが失敗します。

**原因** : ``round_timeout`` (ピア間の P2P モデル転送における ACK の許容時間を設定します) の
デフォルトは 3600 秒です。輻輳したネットワーク上で非常に大きなモデル (7B 以上) を扱う場合、
ピアツーピアのテンソルストリーミングがこの上限に近づくことがあります。

**解決策** : recipe に直接 ``round_timeout`` を設定します。

.. code-block:: python

   recipe = SwarmLearningRecipe(
       name="swarm",
       model=MyModel(),
       min_clients=3,
       num_rounds=5,
       train_script="client.py",
       round_timeout=7200,  # 2 hours for 70B+ models
   )

サイト横断評価のタイムアウト
----------------------------

**症状** : サイト横断検証中にモデルの評価が失敗するか、タイムアウトします。

**解決策** : 評価のタイムアウトを調整します。

.. code-block:: python

   from nvflare.app_common.np.recipes import NumpyCrossSiteEvalRecipe

   recipe = NumpyCrossSiteEvalRecipe(
       submit_model_timeout=900,      # 15 min for model submission
       validation_timeout=7200,       # 2 hours for validation
   )


クイックリファレンス表
======================

最も頻繁に調整されるタイムアウト
--------------------------------

.. list-table::
   :header-rows: 1
   :widths: 30 15 55

   * - タイムアウト
     - デフォルト
     - 増やすべき状況
   * - get_task_timeout
     - None
     - 大規模モデル、低速なネットワーク、テンソルストリーミング
   * - submit_task_result_timeout
     - None
     - 大きな結果ペイロード
   * - submit_result_timeout (サブプロセスモードのみ)
     - Client API のジョブ設定経由では 300 秒、素の ``FlareAgent`` では 60 秒
     - サブプロセスからの大規模モデル結果の転送。LLM の場合は 1800 秒に設定する
   * - tensor_min_download_timeout / np_min_download_timeout (サブプロセスモードのみ)
     - 300 秒
     - 輻輳したネットワーク上の 70B 以上のモデル。600 秒まで増やす (tensor = PyTorch、np = NumPy/sklearn)
   * - PEER_READ_TIMEOUT (Client API のサブプロセスのみ)
     - 300 秒
     - ストリーミングのリクエストごとのタイムアウトを明示的に増やしている場合の、大きなタスクペイロード
   * - download_complete_timeout (サブプロセスモードのみ)
     - 1800 秒
     - サーバーが大きなテンソル結果をダウンロードする間、サブプロセスを生存させ続ける
   * - max_resends (サブプロセスモードのみ)
     - 3
     - 継続的なネットワーク障害。有限値を保つこと。0 でリトライを無効化
   * - round_timeout (Swarm Learning のみ)
     - 3600 秒
     - Swarm のピア間での 7B 以上のモデルの P2P 転送
   * - external_pre_init_timeout (Client API のサブプロセスのみ)
     - 60-300 秒
     - LLM、 ``flare.init()`` 前の重いインポート
   * - heartbeat_timeout
     - 60-300 秒
     - 長い学習イテレーション、低速なネットワーク
   * - train_timeout
     - 0
     - 長い学習ラウンド
   * - validation_timeout
     - 6000 秒
     - 大規模な検証データセット
   * - progress_timeout
     - 3600 秒
     - 複雑なマルチラウンドのワークフロー


設定方法
========

Recipe API を使う方法
----------------------

.. code-block:: python

   # Client-side timeouts (applies to all clients)
   recipe.add_client_config({
       "get_task_timeout": 300,
       "submit_task_result_timeout": 300,
   })

   # Or for specific clients
   recipe.add_client_config({
       "get_task_timeout": 600,
   }, clients=["site-1", "site-2"])


設定ファイルを使う方法
----------------------

**application.conf** (ジョブレベル):

.. code-block::

   get_task_timeout = 300.0
   submit_task_result_timeout = 300.0

   # Server startup/dead-job safety flags
   strict_start_job_reply_check = false
   sync_client_jobs_require_previous_report = true

サーバー側の安全フラグに関するガイダンス (詳細は :ref:`server_startup_dead_job_safety_flags` を参照してください):

- ``strict_start_job_reply_check`` (デフォルト ``false`` ): 非厳格モードでは、ジョブ開始時のタイムアウトは
  ``min_sites`` / ``required_sites`` の強制なしにアクティブセットから暗黙的に除外されます。タイムアウトを可視化し、
  起動時に ``min_sites`` / ``required_sites`` の制約を強制するには ``true`` に設定してください。
- ``sync_client_jobs_require_previous_report`` (デフォルト ``true`` ): 起動時や同期時の一時的な競合状態による
  誤ったデッドジョブ報告を避けるため、有効のままにしてください。

**comm_config.json** (システムレベル、スタートアップキット内):

.. code-block:: json

   {
     "heartbeat_interval": 10,
     "streaming_read_timeout": 600
   }


シナリオ別の推奨設定
====================

標準的な学習
------------

.. code-block:: python

   recipe.add_client_config({
       "get_task_timeout": 120,
   })


大規模モデルの学習 (100M+ パラメータ)
--------------------------------------

.. code-block:: python

   recipe.add_client_config({
       "get_task_timeout": 600,
       "submit_task_result_timeout": 600,
       "submit_result_timeout": 600,        # subprocess mode only
       "download_complete_timeout": 1800,   # subprocess mode only
       "tensor_min_download_timeout": 300,  # subprocess mode only; use np_min_download_timeout for NumPy
       "PEER_READ_TIMEOUT": 600,            # subprocess mode only
       "max_resends": 3,                    # subprocess mode only; finite default
   })


LLM/基盤モデルの学習
---------------------

.. code-block:: python

   recipe.add_client_config({
       "get_task_timeout": 1200,
       "submit_task_result_timeout": 1800,  # server-side; must be >= submit_result_timeout
       "submit_result_timeout": 1800,       # subprocess mode only
       "download_complete_timeout": 1800,   # subprocess mode only
       "tensor_min_download_timeout": 600,  # PyTorch; use np_min_download_timeout for NumPy
       "PEER_READ_TIMEOUT": 600,            # subprocess mode only
       "max_resends": 5,                    # subprocess mode only
   })


高レイテンシネットワーク
------------------------

.. code-block:: python

   # Longer communication timeouts
   recipe.add_client_config({
       "get_task_timeout": 600,
       "submit_task_result_timeout": 600,
   })

システムレベル (スタートアップキット内の ``comm_config.json`` ):

.. code-block:: json

   {
     "heartbeat_interval": 15,
     "streaming_read_timeout": 600
   }


ストリーミング停止のガードレール (``comm_config.json``)
--------------------------------------------------------

大きなペイロードやモデルの転送では、 ``comm_config.json`` (サーバーおよびクライアントのスタートアップキット) で
F3 のストリーム停止検出を設定してください。

**ランタイムのデフォルト値** (明示的に設定されていない場合):

- ``streaming_send_timeout`` : ``30.0`` 秒
- ``streaming_ack_progress_timeout`` : ``60.0`` 秒
- ``streaming_ack_progress_check_interval`` : ``5.0`` 秒
- ``sfm_send_stall_timeout`` : ``45.0`` 秒
- ``sfm_close_stalled_connection`` : ``false`` (警告のみ)
- ``sfm_send_stall_consecutive_checks`` : ``3``

**推奨されるデプロイのガイドライン** :

1. まずは **警告のみ** から始めて、安全に挙動を観察します。
2. 大規模モデルのストリーミング中に停止の警告が繰り返し観測される場合は、自動クローズを有効にします。
3. 誤検知を減らすため、連続チェック付きでガードを有効なままにします。

警告のみのベースライン:

.. code-block:: json

   {
     "sfm_close_stalled_connection": false,
     "sfm_send_stall_timeout": 75,
     "sfm_send_stall_consecutive_checks": 3
   }

自動復旧モード (必要な場合):

.. code-block:: json

   {
     "sfm_close_stalled_connection": true,
     "sfm_send_stall_timeout": 75,
     "sfm_send_stall_consecutive_checks": 3
   }

**タイミングの関係 (重要)** :

- ``sfm_send_stall_timeout`` は、送信がブロックされ続けた合計の連続時間と比較されます。
- ``sfm_send_stall_consecutive_checks`` は、ハートビート監視の連続したティック (5 秒ごと) の回数を数えるものであり、
  ``sfm_send_stall_timeout`` の倍数ではありません。

おおよその自動クローズの時間枠 ( ``sfm_close_stalled_connection=true`` の場合):

.. code-block:: text

   close_lower_bound ~= sfm_send_stall_timeout + (HEARTBEAT_TICK * (sfm_send_stall_consecutive_checks - 1))
   close_upper_bound ~= sfm_send_stall_timeout + (HEARTBEAT_TICK * sfm_send_stall_consecutive_checks)

``sfm_send_stall_timeout=75`` かつ ``sfm_send_stall_consecutive_checks=3`` の場合、クローズは通常、
連続した停止が ``85`` - ``90`` 秒程度に達した時点で発生します (225 秒ではありません)。

**外側のタイムアウトに関するガイドライン** :

上位レイヤーのタイムアウト (たとえば ``communication_timeout`` や、メッセージ転送時間を含むタスク/リクエストの
タイムアウト) は、 ``close_upper_bound`` に安全マージンを加えた値より大きく設定してください。

例: ``communication_timeout=300`` は、約 ``90`` 秒の停止自動クローズの時間枠より十分に大きい値です。

**ログの解釈方法** :

- 実際に停止が発生した場合に想定される警告:
  ``Detected stalled send on ... (N/3)``
- 正常なストリーミングでは、停止の警告は出力されないはずです。
- 断続的な停止では、連続したチェックでしきい値に達しない限り、接続はクローズされないはずです。


大規模な階層型 / HPC デプロイメント (Slurm、Lustre)
----------------------------------------------------

共有ファイルシステム (Lustre、GPFS) を備えた HPC システム上で、100 個以上の FL クライアントを
階層型トポロジーで実行する場合、2 つの設定が起動時の信頼性を大きく向上させます。

**1.** ``config_fed_server.json`` **で最小クライアント数の許容度を設定する**

ジョブを中断させることなく、起動時に少数のクライアントが遅れたり利用できなかったりすることを許容します。
144 クライアントのジョブでは、最大 4% 程度の遅延クライアントを許容するのが安全です。

.. code-block:: json

   {
     "workflows": [{
       "id": "controller",
       "path": "nvflare.app_common.workflows.fedavg.FedAvg",
       "args": {
         "num_clients": 144,
         "min_clients": 138
       }
     }]
   }

**2.** ``config_fed_client.json`` **でランナー同期のタイムアウトを延長する**

ランナー同期のデフォルト設定 (リクエストごとのタイムアウトが 2.0 秒で、全体の同期は
``max_runner_sync_timeout`` によって制限される) では、ジョブ起動時に多数のクライアントが Lustre の I/O を
奪い合うことで、初期化が完了する前にタイムアウトする可能性があります。各クライアントの起動により多くの時間を
与えるため、これらの値を増やしてください。

.. code-block:: json

   {
     "runner_sync_timeout": 120,
     "max_runner_sync_timeout": 7200
   }

これら 2 つの変更は、大規模な階層型デプロイメントで最もよく発生する起動時の競合状態に対処するものであり、
FLARE 2.7.2 の起動安定性の修正と互換性があります。


タイムアウト問題のデバッグ
==========================

1. **ログを確認する** — "timeout" メッセージを探し、どのタイムアウトが発生したかを特定します
2. **デバッグログを有効にする** — 詳細なタイミング情報を確認します
3. **ハートビートの状態を監視する** — 管理コンソールで確認します
4. **開発中は長めのタイムアウトから始める** — その後に最適化します

タイムアウトの階層関係や、利用可能なすべてのタイムアウトパラメータについては、
包括的な :ref:`timeouts_programming_guide` を参照してください。
