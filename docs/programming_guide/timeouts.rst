.. _timeouts_programming_guide:

##################################################
NVIDIA FLARE におけるタイムアウト(リファレンス)
##################################################

本ドキュメントは、NVIDIA FLARE におけるすべてのタイムアウト設定について、機能カテゴリごとに
整理し、相互の関係、影響、および使用例とあわせて包括的に解説します。

.. contents:: 目次
   :local:
   :depth: 2

ネットワーク通信のタイムアウト
==================================

このセクションでは、F3/CellNet 通信レイヤー、サーバー設定、クライアント通信設定を含む、
ネットワーク関連のすべてのタイムアウトを扱います。

F3/CellNet レイヤー
------------------------

F3 (Flare-Friendly Framework) と CellNet は、中核となる通信インフラストラクチャを提供します。
これらのタイムアウトは ``comm_config.json`` で設定します。

CommConfigurator の設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

低レベルの通信設定 (comm_config.py):

.. list-table::
   :header-rows: 1
   :widths: 32 10 58

   * - パラメータ
     - デフォルト
     - 目的
   * - heartbeat_interval
     - 可変
     - ハートビートメッセージの送信間隔
   * - subnet_heartbeat_interval
     - 5.0
     - サブネットのハートビートチェックの間隔
   * - streaming_read_timeout
     - 300
     - ストリーミングデータの読み取りタイムアウト
   * - streaming_ack_interval
     - 4MB
     - ストリーミング中の ACK メッセージ間のバイト数
   * - streaming_ack_wait
     - 可変
     - ストリーミングの ACK を待機する時間
   * - streaming_reliable
     - false
     - ストリーミングされたチャンクを、確認応答が返るまで再試行するかどうか
   * - streaming_retry_wait
     - 5.0
     - 確認応答のない reliable ストリーミングチャンクを再試行するまでの待機時間
   * - streaming_retry_timeout
     - 60.0
     - 確認応答のない reliable ストリーミングチャンクを再試行する最大時間
   * - streaming_retry_max_pending_bytes
     - 2 * streaming_window_size
     - reliable ストリーミングの再試行のためにメモリ上に保持するペイロードの最大バイト数


CoreCell の設定
^^^^^^^^^^^^^^^^^^^^

コアセルの通信パラメータ (core_cell.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - max_timeout
     - 3600
     - send_and_receive のデフォルトタイムアウト(1 時間)
   * - bulk_check_interval
     - 0.5
     - 一括メッセージのチェック間隔
   * - bulk_process_interval
     - 0.5
     - 一括メッセージの処理間隔


Cell リクエストのタイムアウト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

セルレベルのリクエストタイムアウト (cell.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - 10.0
     - send_request/broadcast_request のデフォルトタイムアウト

**タイムアウトのフェーズ**: リクエストは 3 つのタイムアウトフェーズを経ます。

1. **送信タイムアウト**: メッセージ送信を完了するまでの時間
2. **リモート処理タイムアウト**: リモート側がリクエストを処理する時間
3. **受信タイムアウト**: レスポンスを受信するまでの時間


``comm_config.json`` の例:

.. code-block:: json

   {
     "heartbeat_interval": 10,
     "subnet_heartbeat_interval": 5,
     "streaming_read_timeout": 300,
     "streaming_ack_interval": 4194304,
     "max_message_size": 1048576
   }


サーバー設定
--------------------

これらのタイムアウトは ``fed_server.json`` またはサーバー設定で構成します。

FedServer のタイムアウト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

サーバーのハートビートと接続管理 (fed_server.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - heart_beat_timeout
     - 600
     - クライアントが停止しているとみなされるまでの、ハートビートがない時間
   * - remove_interval
     - 5.0
     - 停止したクライアントのチェック/削除を行う間隔
   * - check_interval
     - 0.2
     - 接続チェックループの間隔


ServerRunner のタイムアウト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

サーバーランナーの設定 (server_runner.py, server_json_config.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - heartbeat_timeout
     - 60
     - クライアントのハートビートタイムアウト(秒)
   * - task_request_interval
     - 2
     - タスクリクエストの間隔(秒)


管理サーバーのタイムアウト
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

管理サーバーのコマンドタイムアウト (admin.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - 10.0
     - 管理コマンドのタイムアウト
   * - timeout_secs
     - 2.0
     - クライアントへの send_requests のタイムアウト

**例** (fed_server.json):

.. code-block:: json

   {
     "heart_beat_timeout": 600,
     "admin_timeout": 10.0
   }


クライアント設定
----------------------

クライアントのハートビートおよびリトライ設定 (client_train.py, base_client_deployer.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - heart_beat_interval
     - 10.0
     - サーバーへハートビートを送信する間隔
   * - retry_timeout
     - 30
     - リトライ操作のタイムアウト

**注意**: クライアントのステータスを正しく追跡するためには、``heart_beat_interval`` は
サーバーの ``heart_beat_timeout`` より小さくする必要があります。

クライアントからサーバーへの通信
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

低レベルのクライアント通信タイムアウト (communicator.py, fed_client_base.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - communication_timeout
     - 300.0
     - 一般的な通信タイムアウト
   * - maint_msg_timeout
     - 30.0
     - メンテナンスメッセージのタイムアウト
   * - engine_create_timeout
     - 30.0
     - エンジン生成のタイムアウト
   * - retry_timeout
     - 30.0
     - 操作のリトライタイムアウト

Flare Agent
^^^^^^^^^^^^^^^^

外部プロセス連携のための FlareAgent (flare_agent.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - heartbeat_timeout
     - 60.0
     - ピアが停止しているとみなされるまでの、ハートビートがない時間
   * - submit_result_timeout
     - 60.0
     - クライアントの学習プロセスへタスク結果を送信する際のタイムアウト。大規模モデルには
       60 秒では短すぎます。``add_client_config({"submit_result_timeout": 1800})`` で設定してください。
   * - max_resends
     - 生の ``FlareAgent`` では None、Client API のジョブ設定経由では 3
     - 失敗時の最大送信リトライ回数。``ClientAPILauncherExecutor`` のジョブでは、
       デフォルトは有限値の ``3`` であり、``None`` はジョブの初期化時に拒否されます。
       ``add_client_config({"max_resends": N})`` で上書きできます。
   * - download_complete_timeout
     - 1800.0
     - 結果の ACK 後、サーバーがサブプロセスの ``DownloadService`` からテンソルの
       ダウンロードを完了するまでサブプロセスが待機する時間。
       ``ClientAPILauncherExecutor`` のジョブでは ``None`` にしてはいけません。

**注意**: 生の ``FlareAgentWithCellPipe`` は ``submit_result_timeout`` のデフォルトが 60.0 秒で、
``max_resends`` は無制限です。``ClientAPILauncherExecutor`` 経由で起動した場合、生成される
Client API の設定が上記のより安全なジョブデフォルトを提供します。レシピベースの外部プロセス
ジョブも Executor の引数に ``max_resends=3`` をシリアライズするため、再読み込みされたジョブが
生の無制限リトライのデフォルトに戻ることはありません。異なる有限のリトライ回数が必要な
ジョブの場合にのみ ``recipe.add_client_config({"max_resends": N})`` を使用してください。

IPC Agent
^^^^^^^^^^^^^^

プロセス間通信のための IPC Agent (ipc_agent.py):

.. list-table::
   :header-rows: 1
   :widths: 32 10 58

   * - パラメータ
     - デフォルト
     - 目的
   * - submit_result_timeout
     - 30.0
     - 結果送信のタイムアウト
   * - flare_site_connection_timeout
     - 60.0
     - CJ の切断に関するタイムアウト
   * - flare_site_heartbeat_timeout
     - None
     - CJ のハートビート欠落に関するタイムアウト


gRPC ユーティリティのタイムアウト
--------------------------------------

gRPC 接続の確立 (grpc_utils.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - ready_timeout
     - 可変
     - gRPC サーバーが準備完了になるまで待機する時間


Reliable Message
----------------------

Reliable Message は、リトライロジックによって配信を保証します (reliable_message.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - per_msg_timeout
     - 可変
     - 個々のメッセージ試行ごとのタイムアウト
   * - tx_timeout
     - 可変
     - すべてのリトライを含むトランザクション全体のタイムアウト

**動作**:

- ``tx_timeout <= per_msg_timeout`` の場合、リクエストはリトライされず 1 回だけ送信されます
- ``tx_timeout`` に達するまでメッセージはリトライされます
- 遅延した重複を処理するため、完了したリクエストは ``2 × tx_timeout`` の間追跡されます

**例**:

.. code-block:: python

   from nvflare.apis.utils.reliable_message import ReliableMessage

   ReliableMessage.send_request(
       target="site-1",
       topic="my_topic",
       request=shareable,
       per_msg_timeout=30.0,   # Each attempt times out after 30s
       tx_timeout=300.0,       # Total transaction timeout 5 minutes
       abort_signal=abort_signal,
       fl_ctx=fl_ctx,
   )


連合イベントのタイムアウト
================================

連合イベントランナーの間隔 (fed_event.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - regular_interval
     - 0.01
     - 通常の処理間隔
   * - grace_period
     - 2.0
     - シャットダウン前の猶予期間
   * - queue_empty_period
     - 2.0
     - キューが空のときに待機する期間


シミュレーターのタイムアウト
====================================

シミュレーター固有のタイムアウト (simulator_runner.py, simulator_worker.py):

.. list-table::
   :header-rows: 1
   :widths: 30 10 60

   * - パラメータ
     - デフォルト
     - 目的
   * - simulator_worker_timeout
     - 60.0
     - シミュレーターワーカーのタイムアウト
   * - app_runner_timeout
     - 60.0
     - アプリランナーのタイムアウト
   * - CELL_CONNECT_CHECK_TIMEOUT
     - 10.0
     - セル接続チェックのタイムアウト
   * - FETCH_TASK_RUN_RETRY
     - 3
     - タスク取得のリトライ試行回数


Flare API セッションのタイムアウト
========================================

プログラム的な API のためのセッション管理 (flare_api.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout (new_session)
     - 10.0
     - セッションを確立するためのタイムアウト
   * - poll_interval
     - 2.0
     - ジョブステータスをポーリングする間隔
   * - set_timeout()
     - 可変
     - セッション固有のコマンドタイムアウト

**例**:

.. code-block:: python

   from nvflare.fuel.flare_api.flare_api import new_secure_session

   # Create session with timeout
   sess = new_secure_session(
       username="admin@nvidia.com",
       startup_kit_location="/path/to/startup",
       timeout=30.0,
   )

   # Set command timeout
   sess.set_timeout(60.0)

   # Monitor job with timeout and poll interval
   rc = sess.monitor_job(job_id, timeout=3600, poll_interval=5.0)


ハートビートのタイムアウト
================================

Executor のハートビート
----------------------------

ハートビートの仕組みは、コンポーネント間の接続性を保証します。

.. list-table::
   :header-rows: 1
   :widths: 25 10 35 30

   * - タイムアウト
     - デフォルト
     - 場所
     - 目的
   * - heartbeat_interval
     - 5.0
     - ``LauncherExecutor`` launcher_executor.py:49
     - ハートビートメッセージを送信する間隔
   * - heartbeat_timeout
     - 60.0
     - ``LauncherExecutor`` launcher_executor.py:50
     - ピアからのハートビートを待機するタイムアウト
   * - peer_read_timeout
     - 60.0
     - ``LauncherExecutor`` launcher_executor.py:46
     - 送信したメッセージをピアが受け取るまで待機する時間

Client API のハートビート
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Client API は、タスク交換の設定からハートビート設定を継承します (config.py:154-159):

.. code-block:: python

   def get_heartbeat_timeout(self):
       return self.config.get(ConfigKey.TASK_EXCHANGE, {}).get(
           ConfigKey.HEARTBEAT_TIMEOUT,
           self.config.get(ConfigKey.METRICS_EXCHANGE, {}).get(ConfigKey.HEARTBEAT_TIMEOUT, 60),
       )

Executor と Launcher のタイムアウト
==========================================

LauncherExecutor 基底クラス
--------------------------------

``LauncherExecutor`` クラスは、外部プロセス管理のための中核となるタイムアウトパラメータを
定義します (launcher_executor.py:38-58):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - launch_timeout
     - None
     - ランチャーの "launch_task" メソッド完了のタイムアウト
   * - task_wait_timeout
     - None
     - タスク結果を取得するためのタイムアウト
   * - last_result_transfer_timeout
     - 300.0
     - 外部プロセスから最終結果を転送するためのタイムアウト
   * - external_pre_init_timeout
     - 60.0
     - ``flare.init()`` の呼び出し前に外部プロセスを待機する時間

ClientAPILauncherExecutor
------------------------------

Client API の Executor は、基底クラスのタイムアウトをより保守的なデフォルト値で拡張します
(client_api_launcher_executor.py:29-53):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - external_pre_init_timeout
     - 300.0
     - 重いライブラリのインポートに対応するために延長されたタイムアウト
   * - peer_read_timeout
     - 300.0
     - ピアがメッセージを受け取るまでのタイムアウト
   * - heartbeat_timeout
     - 300.0
     - Client API 向けに延長されたハートビートタイムアウト
   * - submit_result_timeout
     - 300.0
     - 各結果メッセージを CJ が確認応答するまでのサブプロセス側の待機時間
   * - max_resends
     - 3
     - 最初の結果送信後の最大リトライ回数。``None`` は拒否されます
   * - download_complete_timeout
     - 1800.0
     - サーバー側のテンソルダウンロードが完了するまでサブプロセスが生存し続ける時間

大きなペイロードを扱うサブプロセスモードの Client API ジョブでは、FLARE はジョブ開始時に
以下を検証します。

- ``download_complete_timeout`` は ``None`` であってはなりません。
- ``max_resends`` は有限の非負整数でなければなりません。レシピベースのジョブは、
  Executor の引数にデフォルト値 ``3`` をシリアライズします。リトライを無効化するには ``0``
  を使用してください。無制限のリトライのために ``None`` を使用してはいけません。

``recipe.add_client_config()`` を通じて渡された値は、``config_fed_client.json`` の
トップレベルのエントリになります。サブプロセスモードの Client API ジョブでは、
``ClientAPILauncherExecutor`` がサブプロセス用の ``client_api_config.json`` を書き出す前に
これらの上書きを適用するため、``submit_result_timeout``、``download_complete_timeout``、
``max_resends`` は親のクライアントジョブプロセスと外部の学習プロセスの両方から参照されます。

``tensor_streaming_per_request_timeout`` または ``np_streaming_per_request_timeout`` が
明示的に設定されている場合、FLARE は ``PEER_READ_TIMEOUT`` または
``download_complete_timeout`` がそのストリーミングタイムアウトより短いときにも警告します。
親のクライアントジョブにより大きなパイプ読み取りの猶予が必要な場合は、
``add_client_config`` を通じて ``PEER_READ_TIMEOUT`` を設定してください。

.. code-block:: python

   recipe.add_client_config({
       "tensor_streaming_per_request_timeout": 600,
       "tensor_min_download_timeout": 600,
       "PEER_READ_TIMEOUT": 600,
       "download_complete_timeout": 1800,
       "max_resends": 3,
   })

外部プロセスの事前初期化タイムアウトの上書き
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ジョブは、クライアント設定を通じて外部プロセスの事前初期化タイムアウトを上書きできます
(constants.py:20-22):

.. code-block:: python

   # Configuration key for overriding external_pre_init_timeout in ClientAPILauncherExecutor
   EXTERNAL_PRE_INIT_TIMEOUT = "EXTERNAL_PRE_INIT_TIMEOUT"


TaskExchanger
------------------

``TaskExchanger`` 基底クラスは、外部プロセスとのパイプベースのタスク交換を管理します
(task_exchanger.py:38-68):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - read_interval
     - 0.5
     - パイプから読み取る頻度
   * - heartbeat_interval
     - 5.0
     - ピアへハートビートを送信する頻度
   * - heartbeat_timeout
     - 60.0
     - ピアからのハートビートを待機する時間(None = 無効化)
   * - resend_interval
     - 2.0
     - 送信に失敗した場合にメッセージを再送する頻度
   * - peer_read_timeout
     - 60.0
     - 送信したメッセージをピアが受け取るまで待機する時間
   * - result_poll_interval
     - 0.5
     - タスク結果をポーリングする頻度


IPCExchanger
-----------------

``IPCExchanger`` は、Flare Agent との IPC ベースの通信を管理します
(ipc_exchanger.py:50-82):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - send_task_timeout
     - 5.0
     - Agent へタスクを送信した際にレスポンスを待つ時間
   * - resend_task_interval
     - 2.0
     - 失敗した場合にタスクを再送する頻度
   * - agent_connection_timeout
     - 60.0
     - Agent が切断されたとみなすまでにハートビートの欠落を許容する時間
   * - agent_heartbeat_timeout
     - None
     - 停止するまでにハートビートの欠落を許容する時間(None = 無効)
   * - agent_heartbeat_interval
     - 5.0
     - Agent へハートビートを送信する頻度
   * - agent_ack_timeout
     - 5.0
     - Agent の ACK(ハートビートおよび bye メッセージ)を待つ時間


InProcessClientAPIExecutor
-------------------------------

Client API のためのインプロセス Executor (in_process_client_api_executor.py:50-70):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - result_pull_interval
     - 0.5
     - タスク結果をポーリングする頻度
   * - log_pull_interval
     - None
     - ログを取得する頻度(None = result_pull_interval と同じ)

Pipe Handler
-----------------

Client API のためのプロセス間通信パイプのタイムアウト (pipe_handler.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - heartbeat_interval
     - 5.0
     - ハートビートを送信する間隔
   * - heartbeat_timeout
     - 30.0
     - ピアが停止しているとみなすまでの、ハートビートがない最大時間
   * - default_request_timeout
     - 5.0
     - リクエストのデフォルトタイムアウト
   * - resend_interval
     - 2.0
     - メッセージ再送の間隔

**重要**: ``heartbeat_interval`` は ``heartbeat_timeout`` より小さくする必要があります。


P2P Executor
-----------------

ピアツーピアの同期 Executor (sync_executor.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - sync_timeout
     - 10
     - 隣接ノードからの値を待機するタイムアウト


管理クライアントのタイムアウト
====================================

管理クライアントのタイムアウトは、セッション管理とコマンド実行を制御します。

.. list-table::
   :header-rows: 1
   :widths: 28 10 32 30

   * - タイムアウト
     - デフォルト
     - 場所
     - 目的
   * - idle_timeout
     - 900.0
     - 管理設定
     - アイドル期間後の自動シャットダウン
   * - login_timeout
     - 10.0
     - 管理設定
     - ログインを試行する最大時間
   * - authenticate_msg_timeout
     - 2.0
     - 管理設定
     - 認証メッセージのタイムアウト
   * - Command timeout
     - 5.0
     - FLARE API セッション
     - 管理コマンドのデフォルトタイムアウト

セッション固有のタイムアウト
--------------------------------

管理 API は、セッション固有のコマンドタイムアウトをサポートします (api_spec.py:305-318):

.. code-block:: python

   def set_timeout(self, value: float):
       """Set a session-specific command timeout. This is the amount of time the server
       will wait for responses after sending commands to FL clients.
       Note that this value is only effective for the current API session."""


タスク通信とメッセージング
================================

これらのタイムアウトは、サーバーとクライアント間のタスク割り当てと結果収集を制御します。

WfCommServer (ワークフロー通信サーバー)
--------------------------------------------

サーバー側のワークフロー通信 (wf_comm_server.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - task.timeout
     - 可変
     - タスク全体のタイムアウト
   * - task_assignment_timeout
     - 0
     - クライアントがタスクを取得するまで待機する時間
   * - task_result_timeout
     - 0
     - クライアントが結果を返すまで待機する時間
   * - task_check_period
     - 0.2
     - タスクステータスをチェックする間隔

**検証ルール**:

- ``task_assignment_timeout`` は ``task.timeout`` 以下でなければなりません
- ``task_result_timeout`` は ``task.timeout`` 以下でなければなりません


WfCommClient (ワークフロー通信クライアント)
--------------------------------------------

クライアント側のワークフロー通信 (wf_comm_client.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - max_task_timeout
     - 3600
     - 単一タスクの最大実行時間。Controller が task.timeout = 0(つまり「タイムアウトなし」)を設定した場合に、実効的なタイムアウトとして使用されます


タスクの取得(Pull/Fetch)のタイムアウト
------------------------------------------

サーバーからのクライアント側タスク取得 (client_runner.py, communicator.py, fed_client_base.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - get_task_timeout
     - None
     - クライアントがサーバーからタスクを取得する際のタイムアウト
   * - submit_task_result_timeout
     - None
     - クライアントがサーバーへ結果を送信する際のタイムアウト
   * - timeout (pull_task)
     - None
     - pull_task 通信のタイムアウト

**設定方法**: クライアント設定の ``ConfigVarName.GET_TASK_TIMEOUT`` および ``ConfigVarName.SUBMIT_TASK_RESULT_TIMEOUT`` で設定します。

**例** (ジョブ内のクライアントパラメータ):

.. code-block:: python

   recipe.add_client_config({
       "get_task_timeout": 300,  # 5 minutes
   })


タスクマネージャーのタイムアウト
------------------------------------

タスクマネージャーは、逐次およびリレー方式のタスク配布を制御します (send_manager.py, seq_relay_manager.py, any_relay_manager.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - task_assignment_timeout
     - 0
     - クライアントがタスクを要求できる時間枠
   * - task_result_timeout
     - 0
     - 次へ進む前にクライアントの結果を待機する時間

**動作**:

- SendOrder.SEQUENTIAL の場合: クライアントはスライディングタイムウィンドウを用いて順番に割り当てられます
- SendOrder.ANY の場合: 最初に利用可能になったクライアントがタスクを取得します
- タイムアウトが 0 の場合はタイムアウトなし(無期限に待機)を意味します


ワークフローと Controller のタイムアウト
================================================

クライアント制御ワークフロー(サーバー側)
--------------------------------------------

ワークフロー管理のためのサーバー側 Controller のタイムアウト (common.py:79-92):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - タイムアウト
     - デフォルト
     - 目的
   * - configure_task_timeout
     - 300
     - クライアントが config タスクに応答するまでの時間
   * - start_task_timeout
     - 10
     - 開始側クライアントがワークフローを開始するまでの時間
   * - end_workflow_timeout
     - 2.0
     - ワークフロー終了メッセージのタイムアウト
   * - progress_timeout
     - 3600.0
     - ワークフローの進捗がない状態を許容する最大時間
   * - max_status_report_interval
     - 90.0
     - クライアントがステータス報告を欠落できる最大時間

クライアント制御ワークフロー(クライアント側)
------------------------------------------------

タスク調整のためのクライアント側タイムアウト (common.py:87-92):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - タイムアウト
     - デフォルト
     - 目的
   * - learn_task_check_interval
     - 1.0
     - 新しい学習タスクをチェックする間隔
   * - learn_task_ack_timeout
     - 10
     - P2P モデル転送の ACK 猶予時間(秒)。2 GB を超えるモデルには 10 秒では短すぎます。
       ``learn_task_ack_timeout`` と ``final_result_ack_timeout`` の両方を設定する
       ``SwarmLearningRecipe(round_timeout=3600)`` で指定してください。
   * - learn_task_abort_timeout
     - 5.0
     - タスク中断のタイムアウト
   * - final_result_ack_timeout
     - 10
     - 最終結果の確認応答のタイムアウト。上記の ``learn_task_ack_timeout`` の注記を参照してください。
   * - get_model_timeout
     - 10
     - ピアからモデルを取得する際のタイムアウト
   * - max_task_timeout
     - 3600
     - 単一タスクの最大実行時間

ScatterAndGather Controller
--------------------------------

SAG Controller は集約のタイミングを管理します (scatter_and_gather.py:37-67):

.. list-table::
   :header-rows: 1
   :widths: 35 12 53

   * - パラメータ
     - デフォルト
     - 目的
   * - train_timeout
     - 0
     - クライアントがローカル学習を行うのを待機する時間(0 = タイムアウトなし)
   * - wait_time_after_min_received
     - 10
     - min_clients に達した後、追加の応答を待機する時間
   * - task_check_interval
     - 0.5
     - タスク完了をチェックする間隔


ModelController ベースのワークフロー
----------------------------------------

FedAvg、Cyclic、Scaffold、およびその他の ModelController ベースのワークフロー (model_controller.py, base_model_controller.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - 0
     - クライアントがタスクを実行するのを待機する時間(0 = タイムアウトなし)

**注意**: FedAvg、Scaffold、Cyclic はすべて ModelController を継承しており、同じ ``timeout`` パラメータを使用します。


CyclicController
---------------------

Cyclic ワークフローの Controller (cyclic_ctl.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - task_assignment_timeout
     - 10
     - クライアントが割り当てられたタスクを要求するまでのタイムアウト


CrossSiteModelEval / CrossSiteEval
--------------------------------------

サイト横断のモデル評価ワークフロー (cross_site_model_eval.py, cross_site_eval.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - submit_model_timeout
     - 600
     - submit_model_task のタイムアウト(10 分)
   * - validation_timeout
     - 6000
     - validate_model タスクのタイムアウト(100 分)
   * - wait_for_clients_timeout
     - 300
     - クライアントが現れるまでのタイムアウト(5 分)
   * - eval_task_timeout (CCWF)
     - 1200+
     - クライアントによるモデル評価の時間
   * - configure_task_timeout (CCWF)
     - 300
     - 構成タスクのタイムアウト
   * - progress_timeout (CCWF)
     - 7200+
     - ワークフロー全体の進捗タイムアウト

設定例:

.. code-block:: python

   from nvflare.app_common.np.recipes import NumpyCrossSiteEvalRecipe

   recipe = NumpyCrossSiteEvalRecipe(
       submit_model_timeout=600,
       validation_timeout=6000,
   )


GlobalModelEval
--------------------

グローバルモデル評価の Controller (global_model_eval.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - validation_timeout
     - 6000
     - validate_model タスクのタイムアウト
   * - wait_for_clients_timeout
     - 300
     - クライアントが現れるまでのタイムアウト


BroadcastAndProcess / InitializeGlobalWeights
------------------------------------------------

ブロードキャスト系ワークフロー (broadcast_and_process.py, initialize_global_weights.py):

.. list-table::
   :header-rows: 1
   :widths: 30 10 60

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout / task_timeout
     - 0
     - タスクのタイムアウト(0 = タイムアウトなし)
   * - wait_time_after_min_received
     - 0-10
     - 最小数の応答を受信した後の待機時間


StatisticsController / HierarchicalStatisticsController
----------------------------------------------------------

統計ワークフローの Controller (statistics_controller.py, hierarchical_statistics_controller.py):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - result_wait_timeout
     - 10
     - 統計量ごとに結果を待機する秒数
   * - wait_time_after_min_received
     - 1
     - 最小数のクライアントから受信した後に待機する秒数

**注意**: ``result_wait_timeout`` は統計量ごとにリセットされるものであり、全体のタイムアウトではありません。


SplitNNController
----------------------

Split Learning の Controller (splitnn_workflow.py:47-79):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - task_timeout
     - 10
     - クライアントが割り当てられたタスクを要求するまでのタイムアウト
   * - TIMEOUT (クラス定数)
     - 60.0
     - 補助メッセージリクエストのタイムアウト


TIE Controller (サードパーティ統合)
----------------------------------------

サードパーティ統合のための基底 Controller (tie/controller.py, tie/defs.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - configure_task_timeout
     - 10
     - クライアントが config タスクを完了するまで待機する時間
   * - start_task_timeout
     - 10
     - クライアントが start タスクを完了するまで待機する時間
   * - job_status_check_interval
     - 2.0
     - クライアントのジョブステータスをチェックする頻度
   * - max_client_op_interval
     - 90.0
     - クライアントからのアプリ操作の間隔として許容される最大時間
   * - progress_timeout
     - 3600.0
     - ワークフローの進捗がない状態を許容する最大時間

**注意**: TIE は XGBoost、Flower、およびその他のサードパーティフレームワーク統合で使用されます。


Flower 統合のタイムアウト
------------------------------

Flower 固有の Controller および Executor のタイムアウト (flower/controller.py, flower/executor.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - superlink_ready_timeout
     - 10.0
     - Flower の superlink が準備完了になるまで待機する時間
   * - superlink_min_query_interval
     - 10.0
     - superlink のステータスを問い合わせる最小間隔
   * - monitor_interval
     - 0.5
     - Flower の実行ステータスをチェックする頻度
   * - per_msg_timeout
     - 10.0
     - ReliableMessage のメッセージ単位のタイムアウト
   * - tx_timeout
     - 100.0
     - ReliableMessage のトランザクションタイムアウト
   * - client_shutdown_timeout
     - 5.0
     - クライアントのグレースフルシャットダウンの最大時間


Private Set Intersection (PSI)
----------------------------------

PSI ワークフローには、PSI Controller レベルでの明示的なタイムアウトパラメータはありません。
PSI は、基盤となるタスクシステムから一般的なワークフローのタイムアウトを継承します。

PSI の処理では、タイムアウトはより低いレベルで制御されます。

- **タスクレベルのタイムアウト**: Controller の一般的な ``timeout`` パラメータを使用します
- **通信のタイムアウト**: システムの ``heartbeat_timeout`` と ``peer_read_timeout`` を継承します

**注意**: 大規模な PSI 処理では、反復的な Diffie-Hellman プロトコルのやり取りに対応できるよう、
``application.conf`` でシステムレベルのタイムアウトを十分に確保してください。


Aggregator のタイムアウト
------------------------------

LazyAggregator
^^^^^^^^^^^^^^^^^^

非同期集約のための Lazy Aggregator (lazy.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - accept_timeout
     - 600.0
     - accept の完了を待機する最大時間

ジョブスケジューラーのタイムアウト
--------------------------------------

``DefaultJobScheduler`` はジョブのスケジューリング頻度を制御します (job.rst:255-270):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - min_schedule_interval
     - 10.0
     - スケジューリング試行間の最小間隔
   * - max_schedule_interval
     - 600.0
     - スケジューリング試行間の最大間隔
   * - max_schedule_count
     - 10
     - ジョブのスケジューリングを試行する最大回数

**スケジューリング戦略**: スケジューラーは適応的な頻度を使用し、失敗するたびに間隔を
最大値まで倍増させます。

Recipe のタイムアウト
==========================

標準 Recipe のタイムアウト
------------------------------

すべての標準 Recipe は、これらのタイムアウトパラメータをサポートします (fedavg.py, cyclic.py):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - shutdown_timeout
     - 0.0
     - クリーンアップのためにシャットダウン前に待機する時間
   * - task_assignment_timeout
     - 10
     - Cyclic のタスク割り当てのタイムアウト(CyclicRecipe のみ)

``CyclicRecipe`` は両方のパラメータを直接公開しています。その高度な設定である
``server_config_overrides`` および ``client_config_overrides`` の辞書は、それぞれ
``CyclicController`` と ``ScriptRunner`` を対象とします。これらは浅いマージ(shallow merge)を
使用し、重複する名前付きパラメータより優先されます。

評価 Recipe のタイムアウト
------------------------------

評価用の Recipe には固有のタイムアウト要件があります (fedeval.py, cross_site_eval.py):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - validation_timeout
     - 6000
     - モデル検証に許容される時間
   * - submit_model_timeout
     - 600
     - クライアントが評価用のモデルを提出するための時間


大規模モデルとストリーミングのタイムアウト
==============================================

ファイルストリーミングのタイムアウト
----------------------------------------

大きなファイルのファイルストリーミング (file_streamer.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - chunk_timeout
     - 5.0
     - ターゲットへ送信される各チャンクのタイムアウト
   * - chunk_size
     - 1M bytes
     - ストリーミングされる各チャンクのサイズ

**例**:

.. code-block:: python

   from nvflare.app_common.streamers.file_streamer import FileStreamer

   FileStreamer.stream_file(
       targets=["site-1", "site-2"],
       file_name="/path/to/large_file.bin",
       fl_ctx=fl_ctx,
       chunk_size=1024 * 1024,  # 1MB chunks
       chunk_timeout=10.0,      # 10 seconds per chunk
   )


コンテナストリーミングのタイムアウト
----------------------------------------

コンテナ/オブジェクトのストリーミング (container_streamer.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - entry_timeout
     - 60.0
     - ターゲットへ送信される各エントリのタイムアウト

**例**:

.. code-block:: python

   from nvflare.app_common.streamers.container_streamer import ContainerStreamer

   ContainerStreamer.stream_container(
       targets=["site-1"],
       container=my_large_container,
       fl_ctx=fl_ctx,
       entry_timeout=120.0,  # 2 minutes per entry
   )


オブジェクト取得のタイムアウト
------------------------------------

リモートサイトからのファイル/コンテナの取得 (file_retriever.py, container_retriever.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - 可変
     - データ取得を待機する最大秒数
   * - chunk_timeout
     - 可変
     - ファイル取得中のチャンクごとのタイムアウト


バイトストリーミングのタイムアウト
--------------------------------------

バイトストリーミングのタイムアウトと間隔 (byte_receiver.py, byte_streamer.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - streaming_read_timeout
     - 300
     - ストリーミングされたデータの読み取りタイムアウト
   * - ack_interval
     - 4MB
     - 確認応答メッセージ間のバイト数
   * - ack_wait
     - 可変
     - タイムアウトするまで ACK を待機する時間

**注意**: ACK のタイムアウトは ``StreamError`` を発生させ、ストリームを停止します。


ダウンロードトランザクションのタイムアウト
----------------------------------------------

オブジェクトのダウンロードトランザクションのタイムアウト (download_service.py, obj_downloader.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - 可変
     - トランザクションのタイムアウト(最後のアクティビティからの時間)
   * - per_request_timeout
     - 可変
     - オブジェクト所有者への各リクエストのタイムアウト

**注意**: 指定された期間、いずれの受信側からもアクティビティがない場合、トランザクションは
タイムアウトします。正常に完了したダウンロードの参照は一時的に tombstone 化されるため、
同じ受信側からの遅延したリトライは、致命的な「参照が存在しない」レスポンスではなく、
元の EOF またはエラーステータスを受け取ることができます。タイムアウトしたトランザクションと
削除されたトランザクションは tombstone 化されません。


テンソルストリーミングのタイムアウト
----------------------------------------

テンソルストリーミングは、大きなモデル重みの効率的な転送を提供します。これらのタイムアウトは
ストリーミングの動作を制御します (tensor_stream/server.py, client.py):

.. list-table::
   :header-rows: 1
   :widths: 38 12 50

   * - パラメータ
     - デフォルト
     - 目的
   * - tensor_send_timeout
     - 30.0
     - 各テンソルエントリの転送操作のタイムアウト
   * - wait_send_task_data_all_clients_timeout
     - 300.0
     - すべてのクライアントへテンソルを送信する際のタイムアウト
   * - wait_for_tensors timeout
     - 5.0
     - テンソルが受信されるまで待機する時間

**サーバー側の設定** (TensorServerStreamer):

.. code-block:: python

   from nvflare.app_opt.tensor_stream.server import TensorServerStreamer

   streamer = TensorServerStreamer(
       format="pytorch",
       tensor_send_timeout=60.0,  # Per-tensor timeout
       wait_send_task_data_all_clients_timeout=600.0,  # All clients timeout
   )

**クライアント側の設定** (TensorClientStreamer):

.. code-block:: python

   from nvflare.app_opt.tensor_stream.client import TensorClientStreamer

   streamer = TensorClientStreamer(
       format="pytorch",
       tensor_send_timeout=60.0,  # Per-tensor timeout
   )

.. warning::

   **テンソルストリーミングにおける重要なタイムアウトの関係**

   テンソルストリーミングを使用する場合、``get_task_timeout`` を設定し、その値が
   ``wait_send_task_data_all_clients_timeout`` 以上であることを **必ず** 確認してください。
   ``get_task_timeout`` が設定されていない場合、communicator のタイムアウトがデフォルトとして
   使用されますが、これはテンソルストリーミングのタイムアウトより短い可能性があります。

   **問題**: ストリーミングのタイムアウトが communicator のタイムアウトより長く、かつ
   ``get_task_timeout`` が設定されていない場合、一部のクライアントは重みを受信する一方で、
   他のクライアントは待機し続ける可能性があります。サーバーがタスクを時間内に送信できず、
   タイムアウトが発生してテンソルストリーミングの処理が最初からやり直される場合があります。
   これにより、クライアントが空のテンソルを受け取り、ジョブが失敗する可能性があります。

   **解決策**: テンソルストリーミングを使用する場合は、常に ``get_task_timeout`` を
   設定してください。

   .. code-block:: python

      # Ensure get_task_timeout >= wait_send_task_data_all_clients_timeout
      recipe.add_client_config({
          "get_task_timeout": 600,  # Must be >= streaming timeout
      })

ストリーミングダウンロードのタイムアウト
--------------------------------------------

大きなペイロード転送のためのフレームワークレベルの設定 (fl_constant.py:553, comm_config.py:41):

.. list-table::
   :header-rows: 1
   :widths: 35 15 50

   * - パラメータ
     - デフォルト
     - 目的
   * - streaming_per_request_timeout
     - 600
     - ストリーミングチャンクのリクエスト単位のタイムアウト
   * - streaming_read_timeout
     - 300
     - ストリーミングデータの読み取りタイムアウト
   * - np_min_download_timeout
     - 300
     - 非アクティブな NumPy 配列のダウンロードトランザクションが停止していると
       判定されるまでの最小アイドル時間(秒)。NumPy/scikit-learn ベースのモデルに適用されます。
       混雑したネットワーク上で 70B 以上のモデルを扱う場合は 600 秒に増やしてください。
       ``add_client_config({"np_min_download_timeout": 600})`` で設定します。
   * - tensor_min_download_timeout
     - 300
     - 非アクティブな PyTorch テンソルのダウンロードトランザクションが停止していると
       判定されるまでの最小アイドル時間(秒)。PyTorch ベースのモデルに適用されます。
       混雑したネットワーク上で 70B 以上のモデルを扱う場合は 600 秒に増やしてください。
       ``add_client_config({"tensor_min_download_timeout": 600})`` で設定します。
   * - np_download_chunk_size
     - 2097152
     - NumPy 配列ダウンロードのチャンクサイズ(バイト)
   * - tensor_download_chunk_size
     - 2097152
     - PyTorch テンソルダウンロードのチャンクサイズ(バイト)

Client API のサブプロセスジョブでは、これらのダウンロード設定をサブプロセスのパイプ設定と
整合させてください。

- ``tensor_min_download_timeout`` / ``np_min_download_timeout`` は、少なくとも
  ``tensor_streaming_per_request_timeout`` /
  ``np_streaming_per_request_timeout`` 以上にしてください。
- ``PEER_READ_TIMEOUT`` は、少なくとも設定されたストリーミングのリクエスト単位の
  タイムアウト以上にし、サブプロセスが大きなペイロードをダウンロードしている最中に
  親のクライアントジョブがタスクを再送しないようにしてください。
- ``download_complete_timeout`` は、少なくとも設定されたストリーミングのリクエスト単位の
  タイムアウト以上で、かつ結果の ACK 後にサーバーがサブプロセスから大きなテンソルの結果を
  取得するのに十分な長さにしてください。
- ``max_resends`` は有限のままにしてください。レシピのデフォルトは ``3`` です。
  結果の確認応答が数回遅延した後にネットワークが回復すると見込まれる場合にのみ
  値を増やしてください。

Swarm Learning の大規模モデル設定
--------------------------------------

Swarm Learning における大規模モデル向けの推奨タイムアウト:

.. code-block:: python

   recipe = SwarmLearningRecipe(
       name="swarm",
       model=MyModel(),
       min_clients=3,
       num_rounds=5,
       train_script="client.py",
       round_timeout=7200,   # P2P ACK budget; covers learn_task_ack_timeout + final_result_ack_timeout
       progress_timeout=7200,
       start_task_timeout=300,
   )

   # Server-side streaming configuration
   recipe.add_server_config({
       "np_download_chunk_size": 2097152,
       "streaming_per_request_timeout": 600,
   })

   # Subprocess-mode timeouts (when launch_external_process=True)
   recipe.add_client_config({
       "submit_result_timeout": 1800,
       "download_complete_timeout": 1800,
       "tensor_min_download_timeout": 600,
       "PEER_READ_TIMEOUT": 600,
       "max_resends": 5,
   })


XGBoost 固有のタイムアウト
================================

XGBoost ヒストグラムベースの Controller
--------------------------------------------

XGBoost のヒストグラムベース Controller のタイムアウト (histogram_based_v2/controller.py):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - configure_task_timeout
     - 300
     - 構成タスクのタイムアウト
   * - start_task_timeout
     - 10
     - start タスクのタイムアウト
   * - progress_timeout
     - 3600.0
     - ワークフロー全体の進捗タイムアウト

**注意**: XGBoost はセキュアな学習のために Reliable Message を使用します。``per_msg_timeout``
および ``tx_timeout`` の設定については `Reliable Message`_ のセクションを参照してください。

XGBoost gRPC クライアント
------------------------------

XGBoost 通信のための gRPC クライアント (grpc_client.py, grpc_server_adaptor.py):

.. list-table::
   :header-rows: 1
   :widths: 28 12 60

   * - パラメータ
     - デフォルト
     - 目的
   * - ready_timeout
     - 10
     - gRPC サーバーが準備完了になるまでのタイムアウト
   * - xgb_server_ready_timeout
     - 可変
     - XGBoost サーバーの準備完了に関するタイムアウト
   * - aggr_timeout
     - 10.0
     - モックサービサーの集約タイムアウト

大規模データセット向けの設定例:

.. code-block:: python

   "per_msg_timeout": 300.0,
   "tx_timeout": 900.0,


Confidential Computing のタイムアウト
==========================================

SNP Authorizer のタイムアウト
----------------------------------

AMD SEV-SNP のアテステーションに関するタイムアウト (snp_authorizer.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - cmd_timeout
     - 60
     - SNPGuest コマンド実行のタイムアウト
   * - retry_interval
     - 10
     - リトライ試行間の待機時間
   * - max_retries
     - 5
     - 最大リトライ試行回数

CC Manager のタイムアウト
------------------------------

サイト横断の CC 検証に関するタイムアウト (cc_manager.py):

.. list-table::
   :header-rows: 1
   :widths: 30 12 58

   * - パラメータ
     - デフォルト
     - 目的
   * - get_site_request_timeout
     - 10.0
     - サイト取得リクエストのタイムアウト
   * - get_token_request_timeout
     - 10.0
     - トークン取得リクエストのタイムアウト
   * - verify_frequency
     - 600
     - CC トークンの検証間隔(秒)
   * - cross_validation_interval
     - 可変
     - サイト横断の検証サイクル間の間隔

**注意**: その他の CC Authorizer (ACI、TDX、GPU、Azure CVM) には明示的なタイムアウト
パラメータはなく、システムのデフォルト値に依存します。


ジョブランチャーのタイムアウト
====================================

Kubernetes ランチャー
--------------------------

K8s ジョブランチャーのタイムアウト (k8s_launcher.py):

.. list-table::
   :header-rows: 1
   :widths: 20 12 68

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - None
     - Pod が RUNNING/TERMINATED 状態になるまでのタイムアウト

Docker ランチャー
----------------------

Docker コンテナランチャーのタイムアウト (docker_launcher.py):

.. list-table::
   :header-rows: 1
   :widths: 20 12 68

   * - パラメータ
     - デフォルト
     - 目的
   * - timeout
     - None
     - コンテナが目標の状態になるまでのタイムアウト


エッジデバイスのタイムアウト
====================================

このセクションでは、エッジデバイス、モバイルクライアント、および階層型 FL の
すべてのタイムアウトを扱います。

エッジデバイス全般
------------------------

エッジデバイスには固有のタイムアウト要件があります。

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - update_timeout
     - 5
     - デバイスからのモデル更新のタイムアウト
   * - device_wait_timeout
     - None
     - 十分な数のデバイスが参加するのを待機する時間
   * - job_timeout
     - 60.0
     - エッジジョブ実行全体のタイムアウト

例:

.. code-block:: python

   from nvflare.edge.tools.edge_fed_buff_recipe import EdgeFedBuffRecipe

   recipe = EdgeFedBuffRecipe(
       model=MyModel(),
       update_timeout=10,
       job_timeout=120.0,
   )


階層型 FL
--------------

階層型 FL は、ツリー構造に編成されたエッジデバイスによる多階層のフェデレーションを実現します。

ScatterAndGatherForEdge (SAGE) Controller
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

階層型エッジ FL のためのサーバー側 Controller (edge/controllers/sage.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - assess_interval
     - 0.5
     - タスク実行中に assessor を呼び出す間隔
   * - update_interval
     - 1.0
     - 子ノードが更新を送信する間隔
   * - task_check_period
     - 0.5
     - タスクのステータスをチェックする間隔

HierarchicalUpdateGatherer (HUG) Executor
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

階層的な更新収集のための Executor (edge/executors/hug.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - update_timeout
     - 必須
     - 親へ送信する更新メッセージのタイムアウト

EdgeTaskExecutor (ETE)
^^^^^^^^^^^^^^^^^^^^^^^^^^

リーフノード向けのエッジタスク Executor (edge/executors/ete.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - update_timeout
     - 必須
     - 親へ送信する更新メッセージのタイムアウト

**例**:

.. code-block:: python

   from nvflare.edge.controllers.sage import ScatterAndGatherForEdge
   from nvflare.edge.executors.hug import HierarchicalUpdateGatherer

   # Server-side controller
   sage = ScatterAndGatherForEdge(
       num_rounds=5,
       assess_interval=0.5,
       update_interval=1.0,
       task_check_period=0.5,
   )

   # Client-side executor
   hug = HierarchicalUpdateGatherer(
       learner_id="learner",
       updater_id="updater",
       update_timeout=30.0,
   )


モバイルクライアント
------------------------

Android SDK にはジョブ操作のタイムアウトが含まれます (mobile_android.rst:43-58):

.. code-block:: kotlin

   AndroidFlareRunner(
       // ... other parameters
       jobTimeout: Float,  // Timeout in seconds for job operations
   )


SubprocessLauncher のタイムアウト
======================================

サブプロセスランチャーのタイムアウト (subprocess_launcher.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - shutdown_timeout
     - 0.0
     - サブプロセスを強制停止するまでの待機時間


実験トラッキングのタイムアウト
====================================

WandB Receiver
-------------------

Weights & Biases 連携のタイムアウト (wandb_receiver.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - process_timeout
     - 10.0
     - シャットダウン時に WandB プロセスを join する際のタイムアウト
   * - login timeout
     - 1.0
     - WandB のログイン検証に関する内部タイムアウト


MLflow Receiver
--------------------

MLflow 連携のタイミング (mlflow_receiver.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - buffer_flush_time
     - 1
     - MLflow トラッキングサーバーへの送信間隔(秒)

**注意**: ``buffer_flush_time`` を小さくすると MLflow サーバーへのトラフィックが増加し、
レイテンシの原因になる可能性があります。


TensorBoard Receiver
-------------------------

TensorBoard Receiver (tb_receiver.py) には明示的なタイムアウトパラメータはありません。
イベントはバッファリングされることなく、直接ディスクへ書き込まれます。


Metrics Relay と Sender
----------------------------

実験トラッキングのためのメトリクス交換のタイムアウト (metric_relay.py, metrics_sender.py):

.. list-table::
   :header-rows: 1
   :widths: 25 12 63

   * - パラメータ
     - デフォルト
     - 目的
   * - heartbeat_timeout
     - 30.0-60.0
     - ピアのハートビートのタイムアウト(MetricRelay: 60 秒、MetricsSender: 30 秒)
   * - heartbeat_interval
     - 5.0
     - ハートビートの間隔
   * - read_interval
     - 0.1
     - パイプから読み取る間隔

**例**:

.. code-block:: python

   from nvflare.app_common.widgets.metric_relay import MetricRelay

   metric_relay = MetricRelay(
       heartbeat_interval=5.0,
       heartbeat_timeout=60.0,
       read_interval=0.1,
   )


タイムアウトの関係と依存関係
====================================

階層的な関係
------------------

.. code-block:: text

   ┌─────────────────────────────────────────────────────────────────┐
   │                    SYSTEM-LEVEL TIMEOUTS                        │
   ├─────────────────────────────────────────────────────────────────┤
   │  Server Configuration (fed_server.json)                        │
   │  ├── heart_beat_timeout (600s) - Client liveness detection     │
   │  ├── admin_timeout (10s) - Admin command processing            │
   │  └── task_request_interval (2s) - Task polling rate            │
   │                                                                 │
   │  Client Configuration                                           │
   │  ├── heart_beat_interval (10s) - Keep-alive to server          │
   │  ├── retry_timeout (30s) - Operation retry                      │
   │  └── communication_timeout (300s) - Network operations          │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │                    F3/CELLNET LAYER                             │
   ├─────────────────────────────────────────────────────────────────┤
   │  CommConfigurator (comm_config.json)                            │
   │  ├── heartbeat_interval < heartbeat_timeout (REQUIRED)          │
   │  ├── subnet_heartbeat_interval (5s)                             │
   │  ├── streaming_read_timeout (300s)                              │
   │  └── max_timeout (3600s) - CoreCell default                     │
   │                                                                 │
   │  Cell Requests                                                  │
   │  └── timeout (10s) → Sending → Processing → Receiving           │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │                    TASK COMMUNICATION                           │
   ├─────────────────────────────────────────────────────────────────┤
   │  Task Lifecycle                                                 │
   │  ├── task_assignment_timeout ≤ task.timeout (REQUIRED)          │
   │  ├── task_result_timeout ≤ task.timeout (REQUIRED)              │
   │  ├── get_task_timeout - Client fetching task                    │
   │  └── submit_task_result_timeout - Client submitting result      │
   │                                                                 │
   │  max_task_timeout (3600s) - Applied when task.timeout = 0       │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │                    WORKFLOW LAYER                               │
   ├─────────────────────────────────────────────────────────────────┤
   │  ModelController-Based (FedAvg, Cyclic, Scaffold, etc.)         │
   │  └── timeout (0 = no timeout) - Per-task timeout                │
   │                                                                 │
   │  ScatterAndGather / ScatterAndGatherScaffold                    │
   │  ├── train_timeout (0 = no timeout)                             │
   │  └── wait_time_after_min_received (10s)                         │
   │                                                                 │
   │  CyclicController                                               │
   │  └── task_assignment_timeout (10s)                              │
   │                                                                 │
   │  CrossSiteModelEval / CrossSiteEval                             │
   │  ├── submit_model_timeout (600s)                                │
   │  ├── validation_timeout (6000s)                                 │
   │  └── wait_for_clients_timeout (300s)                            │
   │                                                                 │
   │  GlobalModelEval                                                │
   │  ├── validation_timeout (6000s)                                 │
   │  └── wait_for_clients_timeout (300s)                            │
   │                                                                 │
   │  BroadcastAndProcess / InitializeGlobalWeights                  │
   │  ├── timeout / task_timeout (0 = no timeout)                    │
   │  └── wait_time_after_min_received (0-10s)                       │
   │                                                                 │
   │  StatisticsController / HierarchicalStatisticsController        │
   │  └── result_wait_timeout (10s) - Per-statistic timeout          │
   │                                                                 │
   │  SplitNNController                                              │
   │  └── task_timeout (10s)                                         │
   │                                                                 │
   │  TIE Controller (XGBoost, Flower, etc.)                         │
   │  ├── configure_task_timeout (10s)                               │
   │  ├── start_task_timeout (10s)                                   │
   │  ├── job_status_check_interval (2s)                             │
   │  ├── max_client_op_interval (90s)                               │
   │  └── progress_timeout (3600s)                                   │
   │                                                                 │
   │  Flower-Specific                                                │
   │  ├── superlink_ready_timeout (10s)                              │
   │  ├── per_msg_timeout (10s)                                      │
   │  ├── tx_timeout (100s)                                          │
   │  └── client_shutdown_timeout (5s)                               │
   │                                                                 │
   │  CCWF Server-Side                                               │
   │  ├── configure_task_timeout (300s)                              │
   │  ├── start_task_timeout (10s)                                   │
   │  └── progress_timeout (3600s) - Overall workflow                │
   │                                                                 │
   │  CCWF Client-Side (Swarm Learning)                              │
   │  ├── learn_task_ack_timeout (10s)                               │
   │  ├── learn_task_abort_timeout (5s)                              │
   │  └── final_result_ack_timeout (10s)                             │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │                    EXECUTOR LAYER                               │
   ├─────────────────────────────────────────────────────────────────┤
   │  LauncherExecutor / ClientAPILauncherExecutor                   │
   │  ├── launch_timeout                                             │
   │  ├── external_pre_init_timeout (60-300s)                        │
   │  ├── task_wait_timeout                                          │
   │  ├── last_result_transfer_timeout (300s)                        │
   │  └── heartbeat_timeout (60-300s)                                │
   │                                                                 │
   │  TaskExchanger (Pipe Handler)                                   │
   │  ├── heartbeat_interval < heartbeat_timeout (REQUIRED)          │
   │  ├── read_interval (0.5s)                                       │
   │  ├── resend_interval (2s)                                       │
   │  ├── peer_read_timeout (60s)                                    │
   │  └── result_poll_interval (0.5s)                                │
   │                                                                 │
   │  IPCExchanger (Agent-based)                                     │
   │  ├── send_task_timeout (5s)                                     │
   │  ├── resend_task_interval (2s)                                  │
   │  ├── agent_connection_timeout (60s)                             │
   │  ├── agent_heartbeat_timeout (None)                             │
   │  └── agent_ack_timeout (5s)                                     │
   │                                                                 │
   │  InProcessClientAPIExecutor                                     │
   │  ├── result_pull_interval (0.5s)                                │
   │  └── log_pull_interval (None)                                   │
   └─────────────────────────────────────────────────────────────────┘

   ┌─────────────────────────────────────────────────────────────────┐
   │                    STREAMING LAYER                              │
   ├─────────────────────────────────────────────────────────────────┤
   │  Reliable Message                                               │
   │  └── per_msg_timeout ≤ tx_timeout (for retries to work)         │
   │                                                                 │
   │  File/Container Streaming                                       │
   │  ├── chunk_timeout (5s per chunk)                               │
   │  └── entry_timeout (60s per entry)                              │
   │                                                                 │
   │  Tensor Streaming (CRITICAL RELATIONSHIP)                       │
   │  ├── tensor_send_timeout (30s)                                  │
   │  ├── wait_send_task_data_all_clients_timeout (300s)             │
   │  └── get_task_timeout >= wait_send_task_data_all_clients_timeout│
   │      (REQUIRED to prevent task fetch timeout during streaming)  │
   └─────────────────────────────────────────────────────────────────┘


影響の分析
----------------

**タイムアウトが短すぎる場合:**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - タイムアウトのカテゴリ
     - 値が短すぎる場合の影響
   * - heart_beat_timeout
     - クライアントが誤って停止扱いされ、再接続が頻発します
   * - task.timeout / train_timeout
     - 学習が完了前に中断され、作業が失われます
   * - external_pre_init_timeout
     - 大規模モデルの読み込みが失敗し、外部プロセスが強制終了されます
   * - streaming_read_timeout
     - 大きなファイルの転送がストリーミング途中で失敗します
   * - per_msg_timeout
     - 低速なネットワークで Reliable Message が失敗します
   * - get_task_timeout
     - クライアントがタスクを受信できず、ジョブが停滞します
   * - admin_timeout
     - 管理コマンドが失敗し、CLI の使用感が悪化します
   * - task_assignment_timeout (Cyclic)
     - クライアントが時間内にタスクを取得できず、ジョブが中断されます
   * - submit_model_timeout (CrossSiteEval)
     - モデルの提出が失敗し、評価が不完全になります
   * - validation_timeout (CrossSiteEval)
     - 検証タスクが早すぎるタイミングで失敗します
   * - result_wait_timeout (Statistics)
     - すべてのクライアントが応答する前に統計収集が中断されます
   * - agent_connection_timeout (IPC)
     - 外部エージェントが誤って切断扱いされます
   * - send_task_timeout (IPC)
     - エージェントへのタスク配信が失敗し、再送が発生します
   * - superlink_ready_timeout (Flower)
     - Flower 連携の初期化が失敗します
   * - configure_task_timeout (TIE)
     - サードパーティフレームワークの構成が失敗します
   * - max_client_op_interval (TIE)
     - 正常なクライアントがスタックしているとみなされます

**タイムアウトが長すぎる場合:**

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - タイムアウトのカテゴリ
     - 値が長すぎる場合の影響
   * - heart_beat_timeout
     - 停止したクライアントが検出されず、リソースが無駄になります
   * - task_assignment_timeout
     - バックアップのクライアントへのフェイルオーバーが遅くなります
   * - progress_timeout
     - ハングしたワークフローが数時間検出されません
   * - retry_timeout
     - リトライ試行までの遅延が長くなります
   * - shutdown_timeout
     - ジョブの終了が遅くなり、リソースのクリーンアップが遅延します
   * - wait_for_clients_timeout (CrossSiteEval)
     - 参加しないクライアントを長時間待ち続けます
   * - agent_heartbeat_timeout (IPC)
     - ハングしたエージェントが検出されず、ジョブが停滞します
   * - resend_task_interval (IPC/TaskExchanger)
     - 一時的な障害からの回復が遅くなります
   * - result_poll_interval (Executor)
     - 結果の検出が遅れ、ジョブの完了が遅くなります
   * - job_status_check_interval (TIE)
     - ジョブの完了または失敗の検出が遅れます
   * - tx_timeout (ReliableMessage)
     - 失敗したトランザクションを長時間待つことになります


ユースケース別の推奨設定
============================

開発環境
--------------

素早いフィードバックによる高速なイテレーション:

.. code-block:: python

   # Server (fed_server.json)
   heart_beat_timeout = 60        # Quick dead client detection
   admin_timeout = 5.0            # Fast admin commands

   # Client parameters
   heartbeat_timeout = 30.0
   task_wait_timeout = 60.0
   external_pre_init_timeout = 60.0

   # Flare API
   login_timeout = 5.0
   poll_interval = 1.0


本番環境 - 標準的な学習
----------------------------

一般的な連合学習向けのバランスの取れた設定:

.. code-block:: python

   # Server (fed_server.json)
   heart_beat_timeout = 600       # 10 min before client considered dead
   admin_timeout = 10.0
   task_request_interval = 2.0

   # comm_config.json
   heartbeat_interval = 10
   subnet_heartbeat_interval = 5
   streaming_read_timeout = 300

   # Executor
   external_pre_init_timeout = 300.0
   heartbeat_timeout = 300.0
   last_result_transfer_timeout = 300.0


本番環境 - 大規模モデル(1 億パラメータ以上)
------------------------------------------------

大規模モデルの学習のために延長したタイムアウト:

.. code-block:: python

   # Server
   heart_beat_timeout = 1200      # 20 min for large model operations

   # Executor/Launcher
   external_pre_init_timeout = 600.0   # 10 min for model loading
   task_wait_timeout = 3600.0          # 1 hour for training

   # Streaming
   streaming_per_request_timeout = 900  # 15 min per chunk
   tensor_send_timeout = 120.0

   # CCWF
   progress_timeout = 14400       # 4 hours
   learn_task_timeout = 7200      # 2 hours


LLM/基盤モデルの学習
--------------------------

数十億パラメータのモデル向け (examples/advanced/llm_hf):

.. code-block:: python

   # Recipe configuration
   recipe = FedAvgRecipe(
       name="llm_training",
       model=None,  # Use dict config for large models
       shutdown_timeout=120.0,
   )

   # Client parameters - CRITICAL for LLM
   recipe.add_client_config({
       "get_task_timeout": 600,            # 10 min to receive task
       "submit_task_result_timeout": 600,  # 10 min to submit results
       "external_pre_init_timeout": 900,   # 15 min for model init
   })


不安定/高レイテンシのネットワーク
--------------------------------------

厳しいネットワーク条件向けの保守的な設定:

.. code-block:: python

   # More frequent heartbeats with longer tolerance
   heartbeat_interval = 15.0      # Less frequent to reduce traffic
   heartbeat_timeout = 180.0      # 3 min tolerance

   # Extended communication timeouts
   communication_timeout = 600.0
   peer_read_timeout = 180.0
   maint_msg_timeout = 60.0

   # Reliable message settings
   per_msg_timeout = 60.0
   tx_timeout = 600.0             # Long transaction timeout for retries

   # Streaming with larger windows
   streaming_read_timeout = 600
   ack_wait = 30


エッジ/階層型 FL
----------------------

エッジデバイス配備向けの設定:

.. code-block:: python

   # Edge device timeouts
   update_timeout = 30
   job_timeout = 300.0
   device_wait_timeout = 120.0

   # Hierarchical FL
   assess_interval = 1.0
   update_interval = 2.0


XGBoost のセキュア学習
----------------------------

ヒストグラムベースの XGBoost 向けの設定:

.. code-block:: python

   # Controller
   configure_task_timeout = 300
   start_task_timeout = 30
   progress_timeout = 7200

   # Reliable messaging for large histograms
   per_msg_timeout = 120.0
   tx_timeout = 600.0
   xgb_server_ready_timeout = 30


サイト横断のモデル評価
--------------------------

サイト間でモデルを評価する際の設定:

.. code-block:: python

   from nvflare.app_common.workflows.cross_site_model_eval import CrossSiteModelEval

   controller = CrossSiteModelEval(
       submit_model_timeout=900,        # 15 min for large model submission
       validation_timeout=7200,         # 2 hours for thorough validation
       wait_for_clients_timeout=600,    # 10 min for clients to connect
   )


連合統計
--------------

統計計算のための設定:

.. code-block:: python

   from nvflare.app_common.workflows.statistics_controller import StatisticsController

   controller = StatisticsController(
       result_wait_timeout=60,          # 1 min per statistic
       min_clients=2,
   )


Split Learning
-------------------

Split Neural Network の学習向けの設定:

.. code-block:: python

   from nvflare.app_common.workflows.splitnn_workflow import SplitNNController

   controller = SplitNNController(
       task_timeout=30,                 # 30 sec for task assignment
       num_rounds=10,
   )


Flower 統合
----------------

Flower フレームワーク連携のための設定:

.. code-block:: python

   from nvflare.app_opt.flower.flower_job import FlowerJob

   job = FlowerJob(
       superlink_ready_timeout=30.0,    # 30 sec for Flower server
       configure_task_timeout=60,
       start_task_timeout=30,
       progress_timeout=7200,           # 2 hours for training
       per_msg_timeout=30.0,
       tx_timeout=300.0,
       client_shutdown_timeout=10.0,
   )


設定ファイルの場所
========================

このセクションでは、タイムアウトの設定ファイルがどこに配置されているか、また各ファイルが
どのタイムアウトを制御するかを説明します。設定は **システムレベル** (スタートアップキット)と
**ジョブレベル** (アプリケーション)に分かれています。

システムレベルの設定(スタートアップキット)
------------------------------------------------

システムレベルのタイムアウトはスタートアップキットで設定され、すべてのジョブに適用されます。
これらのファイルは、各参加者の ``local/`` ディレクトリに配置されています。

**スタートアップキットの構成:**

.. code-block:: text

   startup_kit/
   ├── server/
   │   └── local/
   │       ├── fed_server.json          # Server heartbeat, admin timeouts
   │       ├── comm_config.json         # F3/CellNet communication layer
   │       └── resources.json           # Resource configuration
   │
   ├── site-1/ (client)
   │   └── local/
   │       ├── fed_client.json          # Client heartbeat, retry timeouts
   │       ├── comm_config.json         # F3/CellNet communication layer
   │       └── resources.json           # Resource configuration
   │
   └── admin/
       └── local/
           └── admin.json               # Admin session timeouts

**デプロイ後のシステムパス:**

デプロイ後、これらのファイルは以下の場所に配置されます。

.. list-table::
   :header-rows: 1
   :widths: 20 40 40

   * - コンポーネント
     - スタートアップキットのパス
     - デプロイ後のパス
   * - サーバー
     - ``startup_kit/server/local/``
     - ``/opt/nvflare/workspace/server/local/`` または ``~/nvflare/workspace/server/local/``
   * - クライアント(サイト)
     - ``startup_kit/site-\*/local/``
     - ``/opt/nvflare/workspace/site-\*/local/`` または ``~/nvflare/workspace/site-\*/local/``
   * - 管理(Admin)
     - ``startup_kit/admin/local/``
     - ``/opt/nvflare/workspace/admin/local/`` または ``~/nvflare/workspace/admin/local/``

**システムレベルの設定ファイル:**

.. list-table::
   :header-rows: 1
   :widths: 22 22 56

   * - ファイル
     - 場所
     - 制御するタイムアウト
   * - fed_server.json
     - server/local/
     - ``heart_beat_timeout``, ``admin_timeout``, ``task_request_interval``, ``heartbeat_timeout``
   * - fed_client.json
     - site-\*/local/
     - ``heart_beat_interval``, ``retry_timeout``, ``communication_timeout``
   * - comm_config.json
     - server/local/, site-\*/local/
     - ``heartbeat_interval``, ``subnet_heartbeat_interval``, ``streaming_read_timeout``, ``streaming_ack_interval``, ``max_message_size``
   * - resources.json
     - server/local/, site-\*/local/
     - リソースの割り当てと上限
   * - admin.json
     - admin/local/
     - ``idle_timeout``, ``login_timeout``, ``command_timeout``

**注意**: システムレベルのファイルを変更した場合、対象となる FLARE のコンポーネントを
再起動する必要があります。


ジョブレベルの設定
------------------------

ジョブレベルのタイムアウトはジョブごとに設定され、その特定のジョブについてデフォルト値を
上書きします。これらのファイルはジョブの ``app/config/`` ディレクトリに配置されています。

**ジョブの設定ファイル:**

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - ファイル
     - 場所
     - 制御するタイムアウト
   * - application.conf
     - app/config/
     - タスクのタイムアウト、ストリーミングのタイムアウト、ランナー同期のタイムアウト
   * - config_fed_client.json
     - app/config/
     - Executor のタイムアウト、Client API のタスク交換、Pipe Handler の設定
   * - config_fed_server.json
     - app/config/
     - Controller のタイムアウト、ワークフローコンポーネントの構成

**ジョブレベルのタイムアウトを設定する方法:**

1. **Recipe API** - ``recipe.add_client_config()`` を使用してクライアントパラメータを渡します。

   .. code-block:: python

      # Apply to all clients
      recipe.add_client_config({
          "get_task_timeout": 300,
          "submit_task_result_timeout": 300,
      })

      # Apply to specific clients
      recipe.add_client_config({
          "get_task_timeout": 600,
      }, clients=["site-1", "site-2"])

2. **ジョブ設定ファイル** - ``app/config/`` ディレクトリ内:

   - ``config_fed_client.json`` - クライアント側の Executor およびタスク交換の設定
   - ``config_fed_server.json`` - サーバー側の Controller およびワークフローの設定

設定例
============

fed_server.json (サーバー設定)
------------------------------------

.. code-block:: json

   {
     "heart_beat_timeout": 600,
     "admin_timeout": 10.0,
     "servers": [
       {
         "heart_beat_timeout": 600
       }
     ]
   }


comm_config.json (F3/CellNet レイヤー)
------------------------------------------

.. code-block:: json

   {
     "heartbeat_interval": 10,
     "subnet_heartbeat_interval": 5,
     "streaming_read_timeout": 300,
     "streaming_ack_interval": 4194304,
     "streaming_reliable": false,
     "streaming_retry_wait": 5.0,
     "streaming_retry_timeout": 60.0,
     "streaming_retry_max_pending_bytes": 33554432,
     "streaming_chunk_size": 1048576,
     "max_message_size": 1048576
   }


Client API の設定 (config_fed_client.json)
------------------------------------------------

.. code-block:: json

   {
     "TASK_EXCHANGE": {
       "heartbeat_timeout": 60.0,
       "heartbeat_interval": 5.0,
       "resend_interval": 2.0,
       "pipe": {
         "ARG": {
           "root_url": "tcp://localhost:8002"
         }
       }
     }
   }


application.conf の設定
----------------------------

.. code-block::

   # Task communication timeouts
   get_task_timeout = 60.0
   submit_task_result_timeout = 120.0
   task_check_timeout = 5.0

   # Cell/messaging timeouts
   cell_wait_timeout = 5.0

   # Streaming timeouts
   streaming_per_request_timeout = 600.0
   np_download_chunk_size = 4194304
   tensor_download_chunk_size = 4194304

   # Runner sync timeouts
   runner_sync_timeout = 10.0
   max_runner_sync_timeout = 60.0

   # Shutdown
   end_run_readiness_timeout = 10.0

   # Server startup/dead-job safety flags
   strict_start_job_reply_check = false
   sync_client_jobs_require_previous_report = true


.. _server_startup_dead_job_safety_flags:

サーバー起動とデッドジョブの安全フラグ
------------------------------------------

これらの ``application.conf`` のフラグは、ジョブ起動時およびクライアントのハートビート同期の
際に使用されるサーバー側の安全制御です。

.. list-table::
   :header-rows: 1
   :widths: 36 12 52

   * - パラメータ
     - デフォルト
     - 目的
   * - strict_start_job_reply_check
     - false
     - 厳密な START_JOB 応答検証を有効にします(応答の欠落/タイムアウトおよび OK 以外のリターンコードを検出します)。
   * - sync_client_jobs_require_previous_report
     - true
     - 「クライアント上にジョブが存在しない」ことをデッドジョブのシグナルとして扱う前に、事前に正常なハートビート報告があることを必須とします。

推奨される使い方:

- ``strict_start_job_reply_check`` は後方互換性のためデフォルトで ``false`` です。
  非厳密モードでは、タイムアウトしたクライアントはアクティブなセットから黙って除外され、
  ジョブは継続します。ただし、それらのタイムアウトについては ``min_sites`` /
  ``required_sites`` の制約が **適用されない** ため、起動時の問題が検出されないまま
  進行する可能性があります。
  厳密モードでは、タイムアウトが検出されて表面化します。``required_sites`` と ``min_sites``
  がチェックされ、制約が依然として満たされている場合にのみ(警告付きで)ジョブが継続します。
  タイムアウトを可視化し、起動時に制約を強制したい場合は厳密モードを有効にしてください。
- 起動時の競合状態や一時的なハートビートの遅延によって誤ったデッドジョブ報告が発生するのを
  防ぐため、``sync_client_jobs_require_previous_report=true`` (デフォルト)のままにして
  ください。
- ``sync_client_jobs_require_previous_report=false`` を設定するのは、最初にジョブが欠落した
  ハートビートで即座にデッドジョブ検出が発動する従来の動作に戻す場合のみにしてください。


管理クライアントセッション (Python API)
--------------------------------------------

.. code-block:: python

   from nvflare.fuel.flare_api.flare_api import new_secure_session

   # Create session with connection timeout
   sess = new_secure_session(
       username="admin@nvidia.com",
       startup_kit_location="/path/to/startup",
       timeout=30.0,
   )

   # Set session-specific command timeout
   sess.set_timeout(60.0)  # 60 seconds for commands

   # Monitor job with timeout
   rc = sess.monitor_job(
       job_id,
       timeout=3600,       # 1 hour max
       poll_interval=5.0,  # Check every 5 seconds
   )

   # Reset to server default
   sess.unset_timeout()


タイムアウトを延長した Recipe
----------------------------------

.. code-block:: python

   from nvflare.app_opt.pt.recipes import FedAvgRecipe

   recipe = FedAvgRecipe(
       name="large_model_training",
       model={"class_path": "model.LargeModel", "args": {}},
       min_clients=8,
       num_rounds=100,
       shutdown_timeout=120.0,
       train_script="client.py",
   )

   # Client timeout parameters
   recipe.add_client_config({
       "get_task_timeout": 300,
       "submit_task_result_timeout": 300,
   })


CCWF/Swarm Learning の設定
--------------------------------

.. code-block:: python

   from nvflare.app_opt.pt.recipes.swarm import SwarmLearningRecipe

   recipe = SwarmLearningRecipe(
       min_clients=3,
       num_rounds=10,
       model=model,
       train_script="train.py",
       cross_site_eval_timeout=600.0,
       round_timeout=3600,   # P2P model-transfer ACK budget; increase for large models
   )


Flower 統合
----------------

.. code-block:: python

   from nvflare.app_opt.flower.recipe import FlowerRecipe

   recipe = FlowerRecipe(
       server_app=ServerApp(...),
       client_app=ClientApp(...),
       superlink_ready_timeout=30.0,
       configure_task_timeout=300,
       start_task_timeout=30,
       progress_timeout=7200,
       per_msg_timeout=30.0,
       tx_timeout=300.0,
       client_shutdown_timeout=10.0,
   )


エッジデバイスの設定
------------------------

.. code-block:: python

   from nvflare.edge.tools.edge_fed_buff_recipe import EdgeFedBuffRecipe

   recipe = EdgeFedBuffRecipe(
       model=MyModel(),
       update_timeout=30,
       job_timeout=600.0,
       device_wait_timeout=120.0,
   )


TaskExchanger の設定
--------------------------

.. code-block:: python

   from nvflare.app_common.executors.task_exchanger import TaskExchanger

   executor = TaskExchanger(
       read_interval=0.5,
       heartbeat_interval=5.0,
       heartbeat_timeout=120.0,
       resend_interval=5.0,
       peer_read_timeout=120.0,
       result_poll_interval=1.0,
   )


LauncherExecutor の設定
------------------------------

.. code-block:: python

   from nvflare.app_common.executors.launcher_executor import LauncherExecutor

   executor = LauncherExecutor(
       launch_timeout=60.0,
       task_wait_timeout=3600.0,
       last_result_transfer_timeout=600.0,
       external_pre_init_timeout=300.0,
       peer_read_timeout=120.0,
       monitor_interval=0.5,
       read_interval=0.5,
       heartbeat_interval=10.0,
       heartbeat_timeout=120.0,
   )


ModelController ベースのワークフロー
----------------------------------------

.. code-block:: python

   from nvflare.app_common.workflows.fedavg import FedAvg

   controller = FedAvg(
       num_clients=8,
       num_rounds=100,
   )

   # Task with timeout
   controller.send_model_and_wait(
       targets=None,
       data=model,
       timeout=3600,  # 1 hour per round
   )


ScatterAndGather の設定
------------------------------

.. code-block:: python

   from nvflare.app_common.workflows.scatter_and_gather import ScatterAndGather

   controller = ScatterAndGather(
       min_clients=4,
       num_rounds=50,
       train_timeout=7200,              # 2 hours per round
       wait_time_after_min_received=30, # Wait 30s for stragglers
       task_check_interval=1.0,
   )


CyclicController の設定
------------------------------

.. code-block:: python

   from nvflare.app_common.workflows.cyclic_ctl import CyclicController

   controller = CyclicController(
       num_rounds=10,
       task_assignment_timeout=30,  # 30 sec to request task
   )


TIE Controller の設定
--------------------------

.. code-block:: python

   from nvflare.app_common.tie.controller import TieController

   controller = TieController(
       configure_task_timeout=60,
       start_task_timeout=30,
       job_status_check_interval=5.0,
       max_client_op_interval=120.0,
       progress_timeout=7200.0,
   )


注意事項とベストプラクティス
====================================

**一般的なルール:**

- タイムアウトの値は、特に指定がない限り **秒** 単位です
- ``None`` または ``0`` は、多くの場合タイムアウトの制限がない(無期限に待機する)ことを意味します
- チャンクサイズの値が ``0`` の場合、ストリーミングは無効化され、ネイティブのシリアライズが使用されます

**重要な制約:**

- ``heartbeat_interval`` は ``heartbeat_timeout`` より **小さく** する必要があります
- ``task_assignment_timeout`` は ``task.timeout`` **以下** である必要があります
- ``task_result_timeout`` は ``task.timeout`` **以下** である必要があります
- リトライを機能させるため、``per_msg_timeout`` は ``tx_timeout`` **以下** にするべきです
- ``agent_heartbeat_interval`` は ``agent_connection_timeout`` より **小さく** する必要があります
- **重要**: テンソルストリーミングを使用する場合、すべてのクライアントがテンソルを受信するのを
  待っている間にタスク取得のタイムアウトが発生するのを防ぐため、``get_task_timeout`` は
  ``wait_send_task_data_all_clients_timeout`` **以上** である必要があります

**テンソルストリーミングのタイムアウトに関する警告:**

テンソルストリーミングが有効な場合、``get_task_timeout`` が明示的に設定されていないと、
communicator のタイムアウトがデフォルトとして使用されます。ストリーミングのタイムアウト
(``wait_send_task_data_all_clients_timeout``)が communicator のタイムアウトを超えると、
他のクライアントが重みを受信するのを待っている間にクライアントがタイムアウトする可能性が
あります。これによりテンソルストリーミングの処理が再開され、クライアントが空のテンソルを
受け取り、ジョブが失敗する可能性があります。

**テンソルストリーミングにおける推奨される関係:**

.. code-block:: text

   get_task_timeout >= wait_send_task_data_all_clients_timeout >= tensor_send_timeout * num_clients

**優先順位:**

- セッション固有のタイムアウトは、サーバーのデフォルト値を上書きします
- クライアント設定の上書きは ``recipe.add_client_config()`` を通じて設定できます
- ``comm_config.json`` の設定は、すべての F3/CellNet 通信に適用されます

**コンポーネント別のベストプラクティス:**

*Controller:*

- 開発中は ``timeout=0`` (タイムアウトなし)から始めてください
- 想定されるラウンド時間に基づいて適切な ``train_timeout`` を設定してください
- サイト横断の評価では、``validation_timeout`` は最長の検証時間を超えるようにしてください
- 低速なクライアントを待つ時間を制限するには ``wait_for_clients_timeout`` を使用してください

*Executor:*

- ``external_pre_init_timeout`` は、モデルの読み込みとライブラリのインポートを賄えるようにしてください
- ``heartbeat_timeout`` は ``heartbeat_interval`` の 2〜3 倍にするべきです
- ``last_result_transfer_timeout`` は結果のサイズに基づいて設定してください
- IPC の場合: ``agent_connection_timeout`` > ``agent_heartbeat_interval`` * 3

*ワークフロー:*

- ``progress_timeout`` はハングしたジョブを検出します。想定されるラウンド時間の 2〜3 倍に設定してください
- ``job_status_check_interval`` は応答性とオーバーヘッドのトレードオフです
- 統計の場合: ``result_wait_timeout`` は合計ではなく統計量ごとの時間です

*ネットワーク/ストリーミング:*

- 高レイテンシのネットワークでは ``per_msg_timeout`` と ``tx_timeout`` を大きくしてください
- ``streaming_read_timeout`` は想定される最も遅い転送に対応できるようにしてください
- 信頼性の低い接続では ``ack_wait`` を長めにしてください

**デバッグのヒント:**

- タイムアウト関連のメッセージを確認するには、デバッグログを有効にしてください
- タイムアウトの統計については、CoreCell の ``num_timeout_reqs`` カウンターを確認してください
- 接続性の問題を早期に検出するため、ハートビートのステータスを監視してください
- どのタイムアウトが発動しているかを特定するため、ログ中の "timeout" を確認してください
- IPC の問題については、``agent_connection_timeout`` とエージェントのログを確認してください
- サードパーティ統合 (TIE) では、``max_client_op_interval`` の発動を監視してください

**よくあるタイムアウトのパターン:**

1. **階層化されたタイムアウト**: より上位のタイムアウトは、下位のタイムアウトを上回るべきです

   - ``progress_timeout`` > ``train_timeout`` > ``task_wait_timeout``
   - ``validation_timeout`` > バッチごとの検証時間 * バッチ数

2. **ハートビートの関係**: 常に適切な比率を保ってください

   - ``heartbeat_timeout`` = ``heartbeat_interval`` の 3〜6 倍
   - ``agent_heartbeat_timeout`` = ``agent_heartbeat_interval`` の 3〜6 倍

3. **リトライの余裕**: リトライのための余裕を残してください

   - ``tx_timeout`` > ``per_msg_timeout`` * 想定リトライ回数
   - ``task.timeout`` > ``task_assignment_timeout`` + 実際の作業時間
