.. _unsafe_component_detection:

********************************
安全でないコンポーネントの検出
********************************
NVFLARE はコンポーネント化されたアーキテクチャに基づいており、FL ジョブは設定ファイルで構成された
コンポーネントによって実行されます。これらのコンポーネントは、ジョブ実行の開始時に作成されます。コンポーネントが
安全でなく、機微な情報を漏洩する可能性があるという問題に対処するため、NVFLARE はイベントベースの解決策を採用しています。

NVFLARE は非常に強力で柔軟なイベントの仕組みを備えており、システムワークフローの定められた時点 (例: ジョブの開始 / 終了、
タスク実行の前 / 後など) にカスタムコードを差し込むことができます。そうした時点で、NVFLARE はイベントを発火し、
それらのイベントを処理する :ref:`fl_component` オブジェクトを呼び出します。

``BEFORE_BUILD_COMPONENT`` イベント型を使うと、カスタムの FLComponent が、設定処理の時点で安全でないジョブ
コンポーネントを検出できるようになります。このイベント型は、設定プロセッサーがジョブコンポーネント
(エグゼキューター、フィルターなど) の構築を開始する前に発火されます。また、別のコンポーネントの ``args`` の中に
再帰的にネストされたコンポーネント設定に対しても発火されます。

安全でないジョブコンポーネントの検出
====================================
安全でないジョブコンポーネントを検出するには、次の ComponentChecker の例のように、このイベントを処理する
カスタムの FLComponent オブジェクトを作成するだけです。この例は意図的に最小限のものです。イベント処理の
パターンと、問題が見つかったときに ``UnsafeComponentError`` を送出する方法を示しています。コンポーネントの
安全性ポリシーの、本番向けの完全な実装ではありません。

.. code-block:: python

    from nvflare.apis.event_type import EventType
    from nvflare.apis.fl_component import FLComponent
    from nvflare.apis.fl_constant import FLContextKey
    from nvflare.apis.fl_context import FLContext
    from nvflare.apis.fl_exception import UnsafeComponentError

    class ComponentChecker(FLComponent):
        def handle_event(self, event_type: str, fl_ctx: FLContext):
            if event_type == EventType.BEFORE_BUILD_COMPONENT:
                comp_config = fl_ctx.get_prop(FLContextKey.COMPONENT_CONFIG)
                if "name" in comp_config:
                    raise UnsafeComponentError("component config must use path or class_path")
                elif "path" in comp_config:
                    component_path = comp_config["path"]
                elif "class_path" in comp_config:
                    component_path = comp_config["class_path"]
                else:
                    return
                if component_path == "bad_package.BadComponent":
                    raise UnsafeComponentError(f"component is not allowed: {component_path}")


重要な点は次のとおりです。

    - クラスは FLComponent を継承しなければなりません
    - まったく同じシグネチャに従って handle_event メソッドを定義します
    - event_type が ``EventType.BEFORE_BUILD_COMPONENT`` かどうかを確認します
    - fl_ctx で提供される情報に基づいて、構築されようとしているコンポーネントを確認します。fl_ctx には多くのプロパティがあります。最も重要なのは、コンポーネントの設定データの dict である ``COMPONENT_CONFIG`` です。fl_ctx には ``WORKSPACE_OBJECT`` もあり、これによりジョブのワークスペース内の任意のファイルにアクセスできます。
    - 構築されようとしているコンポーネントに何らかの問題が検出された場合は、意味のあるテキストを添えて ``UnsafeComponentError`` 例外を送出します。

fl_ctx の以下のプロパティも役立つ可能性があります。

``FLContextKey.COMPONENT_NODE`` - 設定構造 (ツリーと見なすことができます) の中でのコンポーネントの位置に関する情報を
提供します。 ``args`` の中にネストされたコンポーネント設定の場合、このパスには各ネストレベルが含まれます。
例えば ``component.args.child.args.worker`` のようになります。

``FLContextKey.CONFIG_CTX`` - 設定構造全体に関する情報を提供します。

``FLContextKey.CURRENT_JOB_ID`` - 現在のジョブの ID です。

``FLContextKey.JOB_META`` - 現在のジョブに関するメタ情報 (例: ジョブ投入者の名前、組織、ロール) を含む dict です。

``FLContextKey.WORKSPACE_OBJECT`` - このオブジェクトは、ワークスペース内のファイルのパスを判定するための便利なメソッドを多数提供します

組み込みのコンポーネントパス認可機能を使用する
------------------------------------------------
BYOC が無効の場合、NVFLARE はジョブ設定の解析中に、組み込みのコンポーネントパス認可チェックを実行します。サイトは
``resources.json`` に認可コンポーネントをインストールしなくても、この保護を受けられます。このポリシーは、
サイトのトップレベルの ``resources.json`` または ``resources.json.default`` にある ``class_allow_list`` に
一致するクラスパスのみを許可します。標準のプロビジョニングでは、精選された組み込みコンポーネントのリストがインストールされます。
``class_allow_list`` が設定されていない場合、NVFLARE は以下に示す精選された組み込みのデフォルトを使用し、その暗黙の
ポリシー判断について監査イベントを記録します。明示的に設定されたリストは、そのデフォルトを置き換えます。

``SimEnv`` も、POC や本番の認可を変更することなく、新しいシミュレーションワークスペースにこの精選されたリストをインストールします。

アップグレード時の移行に関する注意: このポリシー導入以前に作成されたスタートアップキットには ``class_allow_list`` が
含まれていない場合があります。そのようなサイトは、自動的に組み込みのデフォルトを使用します。サイトがデフォルトを置き換える
必要がある場合 (例えば、BYOC を使わないジョブのためにレビュー済みのサイトローカルなクラスを認可する場合) にのみ、
各サイトの ``resources.json`` または ``resources.json.default`` にトップレベルの ``class_allow_list`` を追加してください。

このチェックは、NVFLARE の JSON 設定フローを通じて構築されるすべてのコンポーネント設定に適用されます。これには、別の
コンポーネントの ``args`` の中に任意の深さでネストされたコンポーネント設定も含まれます。また、マルチプロセス
エグゼキューターや ``engine.build_component()`` のようなランタイムのビルダーによって後から構築される可能性のある、
辞書やリストの中のコンポーネント設定も、構築前にチェックします。マルチプロセスエグゼキューターの ``components``
エントリは、エントリに ``"config_type": "dict"`` が設定されていてもチェックされます。それらのエントリも後で
コンポーネントとして構築されるためです。イベントを発火せずにコンポーネント設定を検証したいコードからは、
``authorize_component_config(...)`` を直接呼び出すこともできます。

ジョブで BYOC が有効になっている場合、この組み込みのクラス許可リストのチェックはスキップされます。BYOC の認可が、
ジョブが提供するカスタムコードの読み込みをすでに許可しているためです。

このポリシーのもとでは、コンポーネント設定は完全修飾クラスパスのキーとして ``path`` または ``class_path`` の
いずれかを使用しなければなりません。両方が存在する場合は ``path`` が優先されます。判定にはキーの存在が使われ、
値の真偽値は使われません。 ``path`` が存在していても空または不正な場合は、 ``class_path`` にフォールバックせずに
拒否されます。 ``name`` を含むコンポーネント設定は、組み込みのパス認可機能によって拒否されます。BYOC を使わない
ジョブでは、完全修飾クラスパスを ``class_allow_list`` と照合できるように ``path`` または ``class_path`` を
使用してください。

``class_allow_list`` は、許可されたコンポーネントパスのプレフィックスのリストです。パッケージのプレフィックスは、
Python のパッケージ境界で一致させるために ``.`` で終わるようにしてください。例えば ``"nvflare."`` のようにします。
末尾に ``.`` がないエントリは完全修飾のドット区切りパスでなければならず、完全一致または ``.`` の境界で照合されます。
例えば ``"nvflare"`` はあいまいであるとして拒否され、 ``"nvflare."`` は ``"nvflareevil.module.Component"`` に
一致しません。

隣接する ``class_list_enforcement_mode`` 設定は、 ``"enforce"`` (デフォルト) または ``"warn"`` を受け付けます。
``"warn"`` モードでは、 ``class_allow_list`` の範囲外のコンポーネントも読み込みが許可され、文脈を含む警告が
ログに記録され、ジョブおよび一致しなかったクラスパスごとに 1 件の監査イベントが記録されます。
``class_allow_list`` のどこかに ``"*"`` が現れる場合、すべてのコンポーネントクラスが許可され、残りのエントリは
無視され、監査イベントにはポリシーの出所と、許可リストのチェックが迂回されたことが記録されます。この場合、
強制モードは効果を持ちません。監査書き込みに失敗した場合は、ワイルドカードの警告を繰り返すことなく、後続の該当する
コンポーネントチェック時に再試行されます。シミュレーターの実行では、監査機構が何もしないため警告ログが使用されます。
``"warn"`` と ``"*"`` は、信頼できる環境における一時的な移行手段としてのみ使用してください。

プロビジョニングされた ``resources.json.default`` の結果
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
プロビジョニングがスタートアップキットを生成する際、サーバーおよびクライアントの ``resources.json.default``
ファイルには、以下のトップレベルの ``class_allow_list`` が含まれます。これは、この設定が省略された場合の組み込みの
フォールバックでもあります。運用者は、BYOC を使わないジョブが読み込みを許可されるクラスに合わせて、
``resources.json`` または ``resources.json.default`` でこれを置き換えることができます。

``DEFAULT_CLASS_ALLOW_LIST`` に含まれるすべてのクラスは、信頼できないジョブ制御の引数を与えられても、安全に
インポートおよび構築できるものでなければなりません。デフォルトリストへの今後の追加は、このセキュリティ基準のもとでの
レビューが必要であり、コンストラクタの副作用や、ファイル、プロセス、ネットワーク、デシリアライゼーション、その他の
特権的な操作を引き起こす可能性のある引数値も対象となります。

.. code-block:: json

    {
        "format_version": 2,
        "class_list_enforcement_mode": "enforce",
        "class_allow_list": [
            "nvflare.app_common.aggregators.collect_and_assemble_model_aggregator.CollectAndAssembleModelAggregator",
            "nvflare.app_common.aggregators.intime_accumulate_model_aggregator.InTimeAccumulateWeightedAggregator",
            "nvflare.app_common.ccwf.comps.simple_model_shareable_generator.SimpleModelShareableGenerator",
            "nvflare.app_common.ccwf.cse_client_ctl.CrossSiteEvalClientController",
            "nvflare.app_common.ccwf.cse_server_ctl.CrossSiteEvalServerController",
            "nvflare.app_common.ccwf.cyclic_client_ctl.CyclicClientController",
            "nvflare.app_common.ccwf.cyclic_server_ctl.CyclicServerController",
            "nvflare.app_common.ccwf.swarm_client_ctl.SwarmClientController",
            "nvflare.app_common.ccwf.swarm_server_ctl.SwarmServerController",
            "nvflare.app_common.executors.statistics.statistics_executor.StatisticsExecutor",
            "nvflare.app_common.filters.statistics_privacy_filter.StatisticsPrivacyFilter",
            "nvflare.app_common.logging.job_log_receiver.JobLogReceiver",
            "nvflare.app_common.logging.job_log_streamer.JobLogStreamer",
            "nvflare.app_common.np.np_model_locator.NPModelLocator",
            "nvflare.app_common.np.np_model_persistor.NPModelPersistor",
            "nvflare.app_common.np.np_validator.NPValidator",
            "nvflare.app_common.psi.dh_psi.dh_psi_controller.DhPSIController",
            "nvflare.app_common.psi.file_psi_writer.FilePSIWriter",
            "nvflare.app_common.psi.psi_executor.PSIExecutor",
            "nvflare.app_common.shareablegenerators.full_model_shareable_generator.FullModelShareableGenerator",
            "nvflare.app_common.statistics.histogram_bins_cleanser.HistogramBinsCleanser",
            "nvflare.app_common.statistics.json_stats_file_persistor.JsonStatsFileWriter",
            "nvflare.app_common.statistics.min_count_cleanser.MinCountCleanser",
            "nvflare.app_common.statistics.min_max_cleanser.AddNoiseToMinMax",
            "nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent",
            "nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector",
            "nvflare.app_common.widgets.validation_json_generator.ValidationJsonGenerator",
            "nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval",
            "nvflare.app_common.workflows.cyclic_ctl.CyclicController",
            "nvflare.app_common.workflows.fedavg.FedAvg",
            "nvflare.app_common.workflows.lr.fedavg.FedAvgLR",
            "nvflare.app_common.workflows.lr.np_persistor.LRModelPersistor",
            "nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather",
            "nvflare.app_common.workflows.scaffold.Scaffold",
            "nvflare.app_common.workflows.statistics_controller.StatisticsController",
            "nvflare.app_opt.he.intime_accumulate_model_aggregator.HEInTimeAccumulateWeightedAggregator",
            "nvflare.app_opt.he.model_decryptor.HEModelDecryptor",
            "nvflare.app_opt.he.model_encryptor.HEModelEncryptor",
            "nvflare.app_opt.he.model_serialize_filter.HEModelSerializeFilter",
            "nvflare.app_opt.he.model_shareable_generator.HEModelShareableGenerator",
            "nvflare.app_opt.psi.dh_psi.dh_psi_task_handler.DhPSITaskHandler",
            "nvflare.app_opt.pt.fedopt.PTFedOptModelShareableGenerator",
            "nvflare.app_opt.pt.file_model_locator.PTFileModelLocator",
            "nvflare.app_opt.pt.recipes.fedeval.EvalController",
            "nvflare.app_opt.sklearn.kmeans_assembler.KMeansAssembler",
            "nvflare.app_opt.sklearn.svm_assembler.SVMAssembler",
            "nvflare.app_opt.tf.fedopt_ctl.FedOpt",
            "nvflare.app_opt.tf.file_model_locator.TFFileModelLocator",
            "nvflare.app_opt.tracking.mlflow.mlflow_receiver.MLflowReceiver",
            "nvflare.app_opt.tracking.mlflow.mlflow_writer.MLflowWriter",
            "nvflare.app_opt.tracking.tb.tb_receiver.TBAnalyticsReceiver",
            "nvflare.app_opt.tracking.tb.tb_writer.TBWriter",
            "nvflare.app_opt.tracking.wandb.wandb_receiver.WandBReceiver",
            "nvflare.app_opt.xgboost.histogram_based_v2.csv_data_loader.CSVDataLoader",
            "nvflare.app_opt.xgboost.histogram_based_v2.fed_controller.XGBFedController",
            "nvflare.app_opt.xgboost.histogram_based_v2.fed_executor.FedXGBHistogramExecutor",
            "nvflare.app_opt.xgboost.tree_based.bagging_aggregator.XGBBaggingAggregator",
            "nvflare.app_opt.xgboost.tree_based.executor.FedXGBTreeExecutor",
            "nvflare.app_opt.xgboost.tree_based.model_persistor.XGBModelPersistor",
            "nvflare.app_opt.xgboost.tree_based.shareable_generator.XGBModelShareableGenerator"
        ],
        "components": [
        ]
    }

上記のポリシーのもとでは、 ``"path": "subprocess.Popen"`` と設定された非 BYOC のジョブコンポーネントは、
``class_allow_list`` のどのエントリにも一致しないため拒否されます。 ``"class_path": "subprocess.Popen"`` にも
同じルールが適用されます。
プロビジョニングされるリストは、フレームワークのオプティマイザー、スケジューラー、モデルのクラスを意図的に除外して
います。ジョブがそれらのクラスを設定する場合、各サイトは BYOC を無効にしてジョブを実行する前に、レビュー済みの
クラスパスまたはパッケージのプレフィックスを ``class_allow_list`` に追加しなければなりません。

これは許可リストによるベースラインです。安全なジョブレビュー、最小権限のランタイム環境、コンテナまたはプロセスの
サンドボックス化、その他デプロイ環境に適した各種の制御を置き換えるものではありません。

コンポーネントチェッカーのインストール
----------------------------------------
コンポーネントチェッカーを定義したら (クラス名は自由に付けられます。ComponentChecker である必要はありません)、
それを FL サイトにインストールする必要があります。

まず、docker の管理方法によっては、カスタムコードを FL docker の一部として含めることができます。それが不可能な
場合は、FL サイトの ``<workspace_root>/local/custom`` フォルダに含めることができます。

次に、以下のように、このカスタムコンポーネントをサイトの ``resources.json`` に記載します。

.. code-block:: json

    {
        "format_version": 2,
        "components": [
            {
                "id": "comp_checker",
                "path": "comp_auth.ComponentChecker"
            }
        ]
    }

サイトのワークスペースは次のようになります。

.. code-block::

    workspace_root
        local
            resources.json
            ...
            custom
                comp_auth.py
        startup
        ...
