.. _component_configuration:

**************************************************************************
NVFLAREのコンポーネント設定とイベント処理
**************************************************************************

NVFLAREには、設定に定義された任意のコンポーネントを動的に構築できる強力な設定メカニズムがあります。
このセクションでは、NVFLAREの設定がどのように機能するか、新しいコンポーネントをどのように開発し、
そのコンポーネントをNVFLAREに認識させ、NVFLAREのイベントに自動登録させるかについて説明します。

背景
==========
ジョブが送信されると、システムはdeploy-mapの設定に基づいてジョブをFLサーバーとFLクライアントにデプロイします。
FLサーバーとクライアントは、それぞれジョブ設定(fed_server_config.jsonとfed_client_config.json)を解析します。
設定の解析中に、システムは設定に基づいてPythonオブジェクトも動的に構築します。インスタンス化された各FLComponentは、
FLAREのイベントループにも登録されます。

このメカニズムは非常に強力です。FLComponentを拡張したカスタムクラスを定義し、そのFLComponentを
設定ファイル(fed_server_config.jsonまたはfed_client_config.json)に登録すれば、そのコンポーネントがFLAREシステムにロードされることが期待できます。

コンポーネントがロードされると、設定ファイルで指定した ``component_id`` によってコンポーネントを見つけることができます。

コンポーネントの設定と検索
------------------------------------------------------------
コンポーネント設定を理解するために、ジョブ設定を見て、コンポーネントがどのように定義され使用されるかを
確認しましょう。以下は :ref:`hello_pt_job_api` のサーバー側設定です。

.. code-block:: json

    {
        //<some lines skipped>
        "components": [
            {
                "id": "persistor",
                "path": "nvflare.app_opt.pt.file_model_persistor.PTFileModelPersistor",
                "args": {}
            },
            {
                "id": "shareable_generator",
                "path": "nvflare.app_common.shareablegenerators.full_model_shareable_generator.FullModelShareableGenerator",
                "args": {}
            },
            {
                "id": "aggregator",
                "path": "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator",
                "args": {
                    "expected_data_kind": "WEIGHTS"
                }
            },
            {
                "id": "model_locator",
                "path": "pt_model_locator.PTModelLocator",
                "args": {}
            },
            //<some lines skipped>
        ],
        "workflows": [
            {
                "id": "pre_train",
                "path": "nvflare.app_common.workflows.initialize_global_weights.InitializeGlobalWeights",
                "args": {
                    "task_name": "get_weights"
                }
            },
            {
                "id": "scatter_and_gather",
                "path": "nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather",
                "args": {
                    "min_clients": 2,
                    "num_rounds": 2,
                    "start_round": 0,
                    "wait_time_after_min_received": 10,
                    "aggregator_id": "aggregator",
                    "persistor_id": "persistor",
                    "shareable_generator_id": "shareable_generator",
                    "train_task_name": "train",
                    "train_timeout": 0
                }
            },
            {
                "id": "cross_site_validate",
                "path": "nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval",
                "args": {
                    "model_locator_id": "model_locator"
                }
            }
        ]
    }

componentsとworkflowsの2つのセクションに注目してください。

コンポーネント設定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
FLAREのジョブ設定はコンポーネントのリストを定義します。ここでは、1つのコンポーネントに焦点を当てるため、他の多くのコンポーネントを省略しています:

.. code-block:: json

    {
        "id": "aggregator",
        "path": "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator",
        "args": {
            "expected_data_kind": "WEIGHTS"
        }
    },

コンポーネント設定は3つの部分で構成されます:
    - コンポーネントid: 例えば ``"id": "aggregator"``
    - コンポーネントパス: 完全修飾クラスパスで、``"path"`` として指定します。例: ``"path": "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator"``
    - コンポーネント引数。例: ``"args": {"expected_data_kind": "WEIGHTS"}``

このクラス定義を見ると、この設定が実際にはクラスのコンストラクタにマッピングされていることがわかります:

.. code-block:: python

    class InTimeAccumulateWeightedAggregator(Aggregator):

        def __init__(
            self,
            exclude_vars: Union[str, Dict[str, str], None] = None,
            aggregation_weights: Union[Dict[str, Any], Dict[str, Dict[str, Any]], None] = None,
            expected_data_kind: Union[DataKind, Dict[str, DataKind]] = DataKind.WEIGHT_DIFF,
        ):

このクラスはexclude_vars、aggregation_weights、expected_data_kindという3つの引数を取ることに注意してください。いずれもデフォルト値を持っています。

上記の設定は、要するに1つの引数を使ってクラスをインスタンス化するようシステムに求めるもので、残りの2つの引数はデフォルト値が使われます。

.. code-block:: python

    a = InTimeAccumulateWeightedAggregator(expected_data_kind = "WEIGHTS")

コンポーネントの ``config_type``
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
場合によっては、コンストラクタの引数としてではなく、辞書としてコンポーネントに引数を渡す必要があります。config_typeはその型を指定するのに役立ちます。

例:

.. code-block:: json

    {
        "id": "shareable_generator",
        "path": "nvflare.app_opt.pt.fedopt.PTFedOptModelShareableGenerator",
        "args": {
            "device": "cpu",
            "source_model": "model",
            "optimizer_args": {
                "path": "torch.optim.SGD",
                "args": {
                    "lr": 1.0,
                    "momentum": 0.6
                },
                "config_type": "dict"
            },
            "lr_scheduler_args": {
                "path": "torch.optim.lr_scheduler.CosineAnnealingLR",
                "args": {
                    "T_max": "{num_rounds}",
                    "eta_min": 0.9
                },
                "config_type": "dict"
            }
        }
    },

次の設定に注目してください:

.. code-block:: json

    "optimizer_args": {
        "path": "torch.optim.SGD",
        "args": {
            "lr": 1.0,
            "momentum": 0.6
        },
        "config_type": "dict"
    },

実行時引数を辞書として"torch.optim.SGD"に渡す必要があります。ここではコンストラクタへの2つの引数としてではなく、
1つの辞書引数として渡すことを意図していると設定パーサーに知らせるために、次のように指定します:

.. code-block:: json

    "config_type": "dict"

``config_type`` を指定しない場合、デフォルトは"Component"です。

Name、Path、class_path
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
標準のジョブ設定パーサーは、組み込みのコンポーネントパス認可を実行します。保護されたジョブ設定では、
コンポーネントを ``"path"`` または ``"class_path"`` で指定します。``"class_path"`` は ``"path"`` のエイリアスです。
両方が存在する場合は ``"path"`` が優先され、記述されたとおりに検証されます。``"name"`` を使用するコンポーネント設定は、
このポリシーにより拒否されます。下位レベルのコンポーネントビルダーは、保護されたジョブ設定フロー以外のコンテキストでは
引き続き ``"name"`` を解決できますが、ジョブ設定の例では ``"path"`` または ``"class_path"`` を使用してください。

設定は次のとおりです::

    "path": "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator"

.. note::

    注記: Recipe APIは引き続き ``class_path`` を受け付け、ジョブ設定のエクスポート時に正規化する場合があります。
    実行時のジョブ設定では、コンポーネント設定に ``path`` またはそのエイリアスである ``class_path`` を使用できます。

コンポーネントの検索
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
コンポーネントが登録されると、component_idを通じてアクセスできます。上記の例では"id": "aggregator"です。

コンポーネントを見つけるには、ランタイムエンジンを使用できます。fl_ctxがFL_Contextオブジェクトであるとすると、次のようにしてコンポーネントを取得できます:

.. code-block:: python

    engine = fl_ctx.get_engine()
    component = engine.get_component(component_id)

失敗のシナリオ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
システムは設定に基づいてクラスを動的にインスタンス化するため、クラスのインスタンス化が失敗する場合があります。
例えば、必須のargsが提供されていない場合や、コンストラクタが例外をスローする場合です。

このような場合、失敗の原因はクラスのインスタンス化ですが、クラスのインスタンス化の失敗が設定の解析に起因しているため、
FLAREはこのエラーを設定エラーとして報告することがあります。トレースバックを確認して、失敗の根本原因を見つける必要があります。

ワークフロー設定
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ジョブ設定の2番目の部分は、``workflows`` キーによるワークフロー設定です。

workflowsはワークフローのリストを定義します。上記の例では、3つのワークフローが定義されています:

    - pre_trainのためのInitializeGlobalWeights
    - scatter_and_gatterによるトレーニングのためのScatterAndGather
    - cross_site_validateによる検証のためのCrossSiteModelEval

各ワークフローは、特別なタイプのFLComponent(:ref:`Controller <controllers>` と呼ばれます)に対応しており、
``id``\ 、``path``\ 、およびクラス定義に一致する引数という同じコンポーネント構造を持ちます。

コントローラーの引数には、プリミティブ型(int、strなど)、または別のコンポーネントのidを指定できます。

検証ワークフローを見ると、CrossSiteModelEvalには"model_locator_id"が必要です。"model_locator_id"の値は"model_locator"で、
これは設定に定義されているコンポーネントの1つのidとして指定されています。

フィルター設定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``task_data_filters`` や ``task_result_filters`` などの追加のオプションフィルターがあります。これらは :ref:`filters` メカニズムに対応しています。

コンポーネントイベント
--------------------------------------------
コンポーネントがコンポーネント設定に基づいて動的にインスタンス化されることを理解した上で、コンポーネントのもう1つの重要な側面が
イベント処理です。

NVIDIA FLAREには強力なイベントメカニズムが備わっており、:ref:`fl_component` のサブクラスであるすべてのオブジェクトに
動的な通知を送信できます。NVFLAREのイベントシステムをより深く理解するには、:ref:`event_system` を参照してください。

システムイベントの例は次のとおりです::

    SYSTEM_START,
    SYSTEM_END,
    ABOUT_TO_START_RUN,
    START_RUN,
    ABOUT_TO_END_RUN
    END_RUN
    START_WORKFLOW
    END_WORKFLOW
    ABORT_TASK
    JOB_DEPLOYED
    JOB_STARTED
    JOB_COMPLETED
    JOB_ABORTED
    JOB_CANCELLED

.. note::

    注記: これはすべてのイベントを網羅したリストではありません。

連合学習(Federated Learning)アプリケーションでは、多くのアプリケーションレベルのイベントが定義され発火されます。以下はその例です::

    BEFORE_AGGREGATION
    END_AGGREGATION

    BEFORE_INITIALIZE
    AFTER_INITIALIZE
    BEFORE_TRAIN
    BEFORE_TRAIN_TASK
    AFTER_TRAIN
    TRAINING_STARTED
    TRAINING_FINISHED
    TRAIN_DONE

    LOCAL_BEST_MODEL_AVAILABLE
    GLOBAL_BEST_MODEL_AVAILABLE

    BEFORE_VALIDATE_MODEL
    AFTER_VALIDATE_MODEL

    ROUND_STARTED
    ROUND_DONE

    INITIAL_MODEL_LOADED

    AFTER_AGGREGATION
    GLOBAL_WEIGHTS_UPDATED

    CROSS_VAL_INIT
    RECEIVE_BEST_MODEL


各FLComponentは、そのコンポーネントがサーバーコンポーネントかクライアントコンポーネントかに応じて、特定のシステムイベントと
アプリケーションイベントを受け取ります。FLComponentクラスは、イベントを処理するか無視するかを決定できます。

コンポーネント設定とイベント処理
------------------------------------------------------------------
コンポーネント設定の2番目のアプローチは、イベントを処理するためにコンポーネントを登録することです。

これまでのコンポーネント設定のアプローチでは、ジョブ設定にコンポーネントを定義し、エンジンを使用してcomponent_idで
コンポーネントを検索していました。この新しいアプローチでは、コンポーネントのidは実際には重要ではなく、
おそらく使用されません。

必要なのは、指定されたイベントを処理するFLComponentを定義することだけです。コンポーネントの直接検索はありません。
コンポーネントがシステムにロードされてさえいれば、FLComponentはイベントハンドラ内でその役割を果たします。

前のセクションで学んだように、コンポーネントをシステムにロードするには、ジョブ設定ファイルにコンポーネントの設定を
追加するだけで済みます。

決めなければならないのは、コンポーネントをどこに配置するかだけです。サーバー側(fed_server_config.json)か
クライアント側(fed_client_config.json)かです。

このメカニズムの具体的な例を1つ示します。NVFLAREの多くの例では、ジョブのコンポーネントに次の記述があることに気づいたかもしれません::

    {
        "id": "model_selector",
        "path": "nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector",
        "args": {}
    }

:class:`nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector` は、保存すべき最良のグローバルモデルを
選択するために設計されたFLComponentで、通常は"validate"タスクに関連付けられています。IntimeModelSelectorは
アプリケーションイベントを処理し、クライアントから送り返された検証スコアに基づいて最良のモデルを選択します。
このモデル選択メカニズムを活用したい場合は、このコンポーネントをサーバーのジョブコンポーネント設定に追加するだけで済みます
(上記に示した設定コード)。

.. code-block:: python

    class IntimeModelSelector(Widget):

        ...

        def handle_event(self, event_type: str, fl_ctx: FLContext):
            if event_type == EventType.START_RUN:
                self._startup()
            elif event_type == AppEventType.ROUND_STARTED:
                self._reset_stats()
            elif event_type == AppEventType.BEFORE_CONTRIBUTION_ACCEPT:
                self._before_accept(fl_ctx)
            elif event_type == AppEventType.BEFORE_AGGREGATION:
                self._before_aggregate(fl_ctx)

        ...

        def _before_aggregate(self, fl_ctx):

        ...

            if self.val_metric > self.best_val_metric:
                self.best_val_metric = self.val_metric

            ...

                # Fire event to notify that the current global model is a new best
                self.fire_event(AppEventType.GLOBAL_BEST_MODEL_AVAILABLE, fl_ctx)

    ...

IntimeModelSelectorが ``BEFORE_AGGREGATION`` イベントを処理する際、最良のモデルを見つけると、単に別のアプリケーションイベント
``AppEventType.GLOBAL_BEST_MODEL_AVAILABLE`` を発火することに注目してください。

永続化を担当する別のFLComponent(persistor)は ``GLOBAL_BEST_MODEL_AVAILABLE`` イベントをリッスンし、
モデルを取得してストレージの場所に保存できます。

異なる基準や異なるイベントに基づく別のモデルセレクターを作成することにした場合は、新しいFLComponentを作成し
(IntimeModelSelectorをサブクラス化するか、単純にゼロから作成)、そのコンポーネントをジョブ設定に追加するだけで済みます。
