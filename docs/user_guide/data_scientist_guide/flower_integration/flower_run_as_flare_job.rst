********************************************************
Flower アプリケーションを FLARE ジョブとして実行する
********************************************************

FLARE で Flower アプリケーションを実行する前に、FLARE と Flower の両フレームワークが Python 環境に
インストールされている必要があります。現在の NVFlare ``main`` および NVFlare 2.8 のリリース候補系列は、
Flower の SuperLink 設定フローを使用しており、 ``flwr>=1.26`` を必要とします。

.. code-block:: shell

    pip install 'flwr>=1.26'

リリース済みの NVFlare 2.7.x を使用している場合は、 ``flwr>=1.16,<1.26`` と Flower サンプルの 2.7 ブランチ
またはタグを使用してください。NVFlare 2.7.x は依然として Flower のレガシーな ``--federation-config`` CLI
オプションを使用しますが、Flower 1.26 以降ではこのオプションは無視されます。

Flower 1.26 以降では、NVFlare ジョブの SuperLink 接続は ``pyproject.toml`` では設定しません。代わりに、
NVFlare がジョブスコープの Flower 設定ファイルを ``$FLWR_HOME/config.toml`` に書き出し、それを使って
``flwr run`` 、 ``flwr list`` 、 ``flwr stop`` をジョブに動的に割り当てられた SuperLink コントロール API の
アドレスに接続します。NVFlare 2.7.x との互換性のための回避策としてグローバルな ``~/.flwr/config.toml`` を
作成しないでください。動的なポートは、Flower 1.26 以降をサポートする NVFlare のバージョンによって設定される
必要があります。

Flower アプリケーションを FLARE のジョブとして実行するには、次の手順に従います。

    - Flower アプリケーションのコード (Python コード) をすべてジョブの "custom" フォルダにコピーします。学習関数はすべて FLARE ではなく Flower 側で実装される点に注意してください。
    - ``config_fed_server.json`` と ``config_fed_client.json`` を作成します
    - 作成したジョブを FLARE システムに送信して実行します

完全な例については、次を参照してください:
:github_nvflare_link:`Hello Flower <examples/hello-world/hello-flower>`

サーバー設定: config_fed_server.json
========================================
一般的なサーバー設定は次のようになります。

.. code-block:: json

    {
        "format_version": 2,
        "task_data_filters": [],
        "task_result_filters": [],
        "components": [
        ],
        "workflows": [
            {
                "id": "ctl",
                "path": "nvflare.app_opt.flower.controller.FlowerController",
                "args": {}
            }
        ]
    }

:class:`FlowerController<nvflare.app_opt.flower.controller.FlowerController>` には、以下に示すように、
その挙動を細かく調整するために設定できる追加の引数があります。

.. code-block:: python

    class FlowerController(TieController):
        def __init__(
            self,
            num_rounds=1,
            database: str = "",
            superlink_ready_timeout: float = 10.0,
            superlink_grace_period: float = 2.0,
            superlink_min_query_interval=10.0,
            monitor_interval: float = 0.5,
            configure_task_name=TieConstant.CONFIG_TASK_NAME,
            configure_task_timeout=TieConstant.CONFIG_TASK_TIMEOUT,
            start_task_name=TieConstant.START_TASK_NAME,
            start_task_timeout=TieConstant.START_TASK_TIMEOUT,
            job_status_check_interval: float = TieConstant.JOB_STATUS_CHECK_INTERVAL,
            max_client_op_interval: float = TieConstant.MAX_CLIENT_OP_INTERVAL,
            progress_timeout: float = TieConstant.WORKFLOW_PROGRESS_TIMEOUT,
            int_client_grpc_options=None,
            run_config: Optional[dict] = None,
        ):
            """Constructor of FlowerController

            Args:
                num_rounds: number of rounds. Not used in this version.
                database: database name
                superlink_ready_timeout: how long to wait for the superlink to become ready before starting server app
                superlink_grace_period: how long to wait for superlink to gracefully shutdown
                superlink_min_query_interval: minimal interval for querying superlink for status
                monitor_interval: how often to check flower run status
                configure_task_name: name of the config task
                configure_task_timeout: max time allowed for config task to complete
                start_task_name: name of the start task
                start_task_timeout: max time allowed for start task to complete
                job_status_check_interval: how often to check job status
                max_client_op_interval: max time allowed for missing client requests
                progress_timeout: max time allowed for missing overall progress
                int_client_grpc_options: internal grpc client options
                run_config: optional dict for flwr run --run-config arguments
            """

``num_rounds`` と ``database`` の引数は現在使用されていません。

ほとんどの引数は既定値のままで十分です。特殊なケースでは、以下の引数の調整が必要になる場合があります。

``Superlink_ready_timeout`` - superlink プロセスが最初に起動され、server-app プロセスを起動する前に
準備完了状態になっている必要があります。superlink が準備完了になる (ポートが開かれ、server-app を受け入れられる状態になる)
までには時間がかかる場合があります。既定値は 10 秒であり、ほとんどのケースではこれで十分です。
不足する場合は、値を増やす必要があるかもしれません。


残りの引数はジョブのライフサイクル管理のためのものです。その意味は
:ref:`XGBoost コントローラ <secure_xgboost_controller>` で使用されているものと同じです。


クライアント設定: config_fed_client.json
--------------------------------------------
一般的なクライアント設定は次のようになります。

.. code-block:: json

    {
        "format_version": 2,
        "executors": [
            {
                "tasks": ["*"],
                "executor": {
                    "path": "nvflare.app_opt.flower.executor.FlowerExecutor",
                    "args": {}
                }
            }
        ],
        "task_result_filters": [],
        "task_data_filters": [],
        "components": []
    }

FlowerExecutor には、以下に示すように、その挙動を細かく調整するために設定できる追加の引数があります。

.. code-block:: python

    class FlowerExecutor(TieExecutor):
        def __init__(
            self,
            start_task_name=Constant.START_TASK_NAME,
            configure_task_name=Constant.CONFIG_TASK_NAME,
            per_msg_timeout=10.0,
            tx_timeout=100.0,
            client_shutdown_timeout=5.0,
        ):

``per_msg_timeout`` と ``tx_timeout`` は、サーバーへのリクエスト送信に使用される
:class:`ReliableMessage<nvflare.apis.utils.reliable_message.ReliableMessage>` を設定します。

``client_shutdown_timeout`` は、FL クライアントを停止する際に Flower の client-app プロセスが
グレースフルにシャットダウンするのを何秒間待つかを指定します。client-app プロセスがこの時間内に
シャットダウンしない場合、Flare によって強制終了されます。
