.. _client_controlled_workflows:

##################################################################
クライアント制御ワークフロー
##################################################################

サーバーベースの制御では、通常、FLクライアントが送信する結果に機密情報(例: トレーニング済みのモデル重み)が含まれる可能性があるため、
サーバーがすべてのクライアントから信頼されていることを前提としています。サーバーが常に信頼できるという前提は、成り立たない場合があります。
サーバーが信頼できない場合、サーバーは機密情報を含む通信に関与してはなりません。これを実現するために、NVFlareは
クライアント間のピアツーピア通信を可能にするクライアント制御ワークフロー\ (Client Controlled Workflows: CCWF)\ を導入しています。

連合学習のワークフローには、管理すべき2つの側面があります。全体的なジョブステータス管理(クライアントサイトの健全性)と、
トレーニングロジック管理(タスクをどのように、いつ割り当てるか)です。サーバー制御ワークフローでは、両方の側面をサーバーが管理します。

クライアント制御ワークフローでは、学習ロジックの管理はクライアント(ピア)によって行われます。FLクライアントは、FLサーバーを介さずに
他のクライアントと通信することで学習制御ロジックを実行します(ピアツーピア)。サーバーの仕事は、全体的なジョブステータスの監視のみとなります。
これは、異常な状態(例: クライアントのクラッシュやスタック)が発生した場合に、ジョブを永久に実行し続けるのではなく、
すみやかに中断できるようにするためです。

クライアント制御ワークフローは、以下の実装を提供します:

    - クライアント制御ワークフローを開発するための汎用フレームワーク
    - よく使用される3つのピアツーピアワークフロー:
        - サイクリック学習(Cyclic learning)
        - スウォーム学習(Swarm learning)
        - クロスサイトモデル評価

****************************************************************************************************
クライアント制御ワークフロー開発フレームワーク
****************************************************************************************************
NVFlareはマルチジョブシステムです。ジョブがシステムに送信されると、サーバーはジョブをスケジュールし、関連するすべてのサイト(サーバーと
クライアント)にデプロイします。このフレームワークは、すべてのクライアント制御ワークフローに共通するパターンを捉えています:

    - ワークフローの設定
    - ワークフロー開始前のクライアントの同期
    - 指定された開始ポイントからのワークフローの開始
    - 全体的なジョブ進捗の監視
    - ワークフローの適切な終了

このフレームワークは、:class:`nvflare.app_common.ccwf.server_ctl.ServerSideController` と
:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` という2つの基底クラスで実装されています。

サーバー側コントローラー
================================================

すべてのFLAREジョブには、サーバー側のコントローラーが必要です。クライアント制御ワークフローでは、:class:`nvflare.app_common.ccwf.server_ctl.ServerSideController` 基底クラスが、
機密性の高いトレーニング情報を一切扱わないジョブライフサイクル管理を実装しています。トレーニングの実行と
機密データ通信を制御するのは、:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController`\ (およびそのサブクラス)です。

すべてのクライアント制御ワークフローには、この基底クラスを拡張したサーバー側コントローラーが必要です。

.. code-block:: python

    class ServerSideController(Controller):
        def __init__(
            self,
            num_rounds: int,
            start_round: int = 0,
            task_name_prefix: str = "wf",
            configure_task_timeout=Constant.CONFIG_TASK_TIMEOUT,
            end_workflow_timeout=Constant.END_WORKFLOW_TIMEOUT,
            start_task_timeout=Constant.START_TASK_TIMEOUT,
            task_check_period: float = Constant.TASK_CHECK_INTERVAL,
            job_status_check_interval: float = Constant.JOB_STATUS_CHECK_INTERVAL,
            starting_client=None,
            starting_client_policy: str = DefaultValuePolicy.ANY,
            participating_clients=None,
            result_clients=None,
            result_clients_policy: str = DefaultValuePolicy.ALL,
            max_status_report_interval: float = Constant.PER_CLIENT_STATUS_REPORT_TIMEOUT,
            progress_timeout: float = Constant.WORKFLOW_PROGRESS_TIMEOUT,
            private_p2p: bool = True,
        ):

ServerSideControllerの初期化引数
------------------------------------------------------------

``num_rounds`` - 実行するラウンド数。これはワークフロー設定パラメータであり、すべてのクライアントに送信されます。

``start_round`` - 開始ラウンド番号。これはワークフロー設定パラメータであり、すべてのクライアントに送信されます。

``task_name_prefix`` - このワークフローのタスク名のプレフィックス。ワークフローでは、サーバーコントローラーとクライアントコントローラーの間で
複数のタスク(例: configとstart)が必要です。これらのタスクの完全な名前は<prefix>_configと<prefix>_startです。サブクラスは追加のタスクを送信する場合があります。
これらのタスクに共通のプレフィックスを付けることで、FLクライアントのタスクエグゼキューターの設定が容易になります。config_fed_client.jsonで
クライアント側エグゼキューターに各タスク名を明示的に指定する代わりに、そのエグゼキューターに<prefix>_*を指定するだけで済みます。これにより、<prefix>を持つ
すべてのタスクが指定されたエグゼキューターにルーティングされます。

``participating_clients`` - ジョブに参加するクライアントの名前。Noneの場合、すべてのクライアントが参加者になります。

``result_clients`` - 最終的な学習結果を受け取るクライアントの名前。最終結果がサーバーに送信されサーバーに保持されるサーバー制御ワークフローとは異なり、
クライアント制御ワークフローでは、結果はクライアントのみが保持します。

``result_clients_policy`` - result_clientsの名前が明示的に指定されていない場合に、それをどのように決定するか。指定可能な値は次のとおりです:
  - ``ALL`` - すべての参加クライアント
  - ``ANY`` - 参加クライアントのうち任意の1つ
  - ``EMPTY`` - result_clientsなし
  - ``DISALLOW`` - 暗黙の指定を許可しない - result_clientsを明示的に指定する必要があります

``configure_task_timeout`` - configタスクに対するクライアントの応答をタイムアウトまで待つ時間。

``starting_client`` - 開始クライアントの名前。すべての参加クライアントがconfigタスクを正常に完了した後、ServerSideControllerは
指定された開始クライアントにワークフローを開始するタスクを送信します。

``starting_client_policy`` - 名前が明示的に指定されていない場合に、開始クライアントをどのように決定するか。指定可能な値は次のとおりです:
  - ``ANY`` - 参加クライアントのうち任意の1つ(ランダムに選択)
  - ``EMPTY`` - 開始クライアントなし
  - ``DISALLOW`` - 暗黙の指定を許可しない - starting_clientを明示的に指定する必要があります

``start_task_timeout`` - 開始クライアントが"start"タスクを完了するまで待つ時間。タイムアウトした場合、ジョブは中断されます。
starting_clientが指定されていない場合、startタスクは送信されないことに注意してください。

``max_status_report_interval`` - クライアントがステータスレポートを欠かすことが許容される最大時間。言い換えると、クライアントがこの時間だけ
ステータスの報告に失敗した場合、そのクライアントは問題があるとみなされ、ジョブは中断されます。

``progress_timeout`` - ワークフローが進捗しないことが許容される最大時間。言い換えると、この時間内に少なくとも1つの参加クライアントが
進捗している必要があります。そうでない場合、ワークフローは問題があるとみなされ、ジョブは中断されます。

``end_workflow_timeout`` - ワークフロー終了メッセージのタイムアウト。

ServerSideControllerの処理ロジック
------------------------------------------------------------

ServerSideControllerの処理ロジックは次のとおりです:

    - ジョブの開始時に、サーバーはジョブのすべての参加クライアントに設定パラメータをブロードキャストします(<prefix>_configタスク)。これには別の目的もあります。すべてのクライアントがこのジョブを実行する準備ができていることを確認することです。いずれかのクライアントがタイムアウトまでに設定の取得や処理に失敗した場合、ジョブは中断されます。
    - starting_clientが指定されている場合、サーバーは開始クライアントに<prefix>_startタスクを送信します。開始クライアントがワークフローの開始に失敗した場合、ジョブは中断されます。
    - ワークフローの完了を待ちます。この間、各クライアントは定期的にステータス更新をサーバーに送信する必要があります。クライアントが指定された時間(max_status_report_interval)内に更新の送信に失敗した場合、ジョブは中断されます。設定された時間(progress_timeout)の間、どのクライアントからも全体的な進捗がない場合、ジョブは中断されます。クライアントがワークフローの完了を報告すると、ジョブは正常に終了します。
    - ジョブが終了したとき(中断または正常終了)、すべてのクライアントにワークフローを終了するメッセージを送信します。

クライアント側コントローラー
====================================================

:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` は、クライアント側における :class:`nvflare.app_common.ccwf.server_ctl.ServerSideController`
の対となる存在で、エグゼキューターとして実装されています。ServerSideControllerと連携して、ジョブライフサイクル管理機能
(ワークフローの設定と開始、ジョブステータス更新の報告など)を実装します。
さらに、具体的なワークフローを実装するサブクラスが必要とする共通機能(例: ステータス更新、結果受信クライアントへの最終結果のブロードキャスト)
のための便利なメソッドも提供します。

.. code-block:: python

    class ClientSideController(Executor, TaskController):
        def __init__(
            self,
            task_name_prefix: str,
            learn_task_name=AppConstants.TASK_TRAIN,
            persistor_id=AppConstants.DEFAULT_PERSISTOR_ID,
            shareable_generator_id=AppConstants.DEFAULT_SHAREABLE_GENERATOR_ID,
            learn_task_check_interval=Constant.LEARN_TASK_CHECK_INTERVAL,
            learn_task_ack_timeout=Constant.LEARN_TASK_SEND_TIMEOUT,
            learn_task_abort_timeout=Constant.LEARN_TASK_ABORT_TIMEOUT,
            final_result_ack_timeout=Constant.FINAL_RESULT_SEND_TIMEOUT,
            allow_busy_task: bool = False,
        ):

初期化引数:
--------------------

``task_name_prefix`` - このワークフローのタスク名のプレフィックス。サーバー制御ワークフローとは異なり、クライアント制御ワークフローではクライアントが互いにタスクを送信します。それらのタスクはすべてこのプレフィックスを付けて命名されます。

``learn_task_name`` - 通常、すでに実装済みの学習エグゼキューターが実行するタスクの名前です。既存の学習エグゼキューターを変更することなく、クライアント制御ワークフローで使用できます。学習タスクの名前をClientSideControllerに伝えるだけで済みます。

``persistor_id`` - persistorコンポーネントのID。persistorは、初期モデルのロードと、トレーニングプロセス中の結果(すなわち最良モデルおよび/または最終モデル)の保存に使用されます。

``shareable_generator_id`` - shareable generatorコンポーネントのID。shareable generatorは、learnableオブジェクト(例: 完全なモデル)とshareableオブジェクト(例: トレーニング対象の重みや、重み差分のような部分的なトレーニング結果)の間の変換を担当します。

``learn_task_check_interval`` - 実行すべき新しい学習タスクをチェックする間隔。学習タスクは専用スレッドで実行され(一度に1タスク)、そのスレッドが定期的に実行すべき学習タスクをチェックします。

``learn_task_ack_timeout`` - 学習タスクを割り当てられたクライアントからのackを受信するまでのタイムアウト。学習タスクはあるクライアントから別のクライアントに割り当てられます。学習タスクを受信すると、受信側クライアントはそれをタスク実行スレッドのキューに入れ、タスク送信元のクライアントにackを送信します。

``learn_task_abort_timeout`` - 学習タスクの中断を待つタイムアウト。特定の状況下では、現在実行中の学習タスクを中断する必要があります(例: ユーザーからabortコマンドを受け取った場合)。

``final_result_ack_timeout`` - 最終結果を送信した後、クライアントからの応答を受信するまでのタイムアウト。ワークフローの終了時に、最終結果を保持するクライアントは、設定されたすべての"result clients"に最終結果を配布します。この引数は、それらのクライアントが結果の受領を確認するまでどれだけ待つかを指定します。

``allow_busy_task`` - 現在の学習タスクの実行中に新しい学習タスクの受信を許可するかどうか。許可しない場合、クライアントはサーバーに致命的エラーを報告し、ジョブが中断されます。許可する場合、現在の学習タスクは中断され、新しく受信したタスクが実行されます。

ClientSideControllerの処理ロジック
------------------------------------------------------------

"config"タスクを受信すると、すべての設定パラメータが検証・処理されます。エラーが発生した場合、エラーコードがサーバーに返され、
ジョブが中断されます。

"start"タスクを受信すると、start_workflowメソッド(サブクラスが実装)が呼び出されます。エラーが発生した場合、エラーコードが
サーバーに返され、ジョブが中断されます。

サーバーからタスクを取得しようとするたびに、現在のジョブステータスレポートが ``GetTask`` リクエストに添付されます。

:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` 基底クラスは、サブクラスがジョブステータスを更新するためのメソッドを提供します。ただし、ジョブステータスの変更は
すぐにサーバーに送信されるわけではありません。ステータスの変更は、定期的に発生するGetTaskリクエストと共にのみ送信されます。そのため、サーバーに報告する前に
サブクラスがジョブステータスを複数回更新する可能性があります。最後のステータス変更のみがサーバーに報告されます。ステータス報告の目的は、
ジョブがまだ進行中であることをサーバーに知らせることなので、これで問題ありません。

サーバーからワークフロー終了メッセージを受信すると、実行中の学習タスクがあれば、その実行を停止します。

.. _ccwf_cyclic_learning:

******************************************
サイクリック学習
******************************************

サイクリック学習(Cyclic Learning)では、学習プロセスは複数のラウンドで行われます。各ラウンドでは、参加クライアントが、
あらかじめ決められた順序に従って順番にトレーニングを行います。各クライアントは、シーケンスの前のクライアントから受け取った結果を元にトレーニングします。

開始クライアントは初期モデルを担当し、初期モデルは設定されたpersistorによってロードされます。

前のクライアントからモデルを受け取ると、次のロジックが実行されます:

    - 設定されたshareable generatorを呼び出して、受け取ったモデル重みをLearnableオブジェクトに変換します。このLearnableが現在のグローバルモデルです。このステップは不要に思えるかもしれませんが、重要なステップです。特にモデルがPyTorchベースでない場合、Learnableオブジェクトは単純な重みの辞書ではない可能性があります。
    - 学習エグゼキューターを呼び出してトレーニングタスクを実行し、トレーニング結果を返します。
    - 設定されたshareable generatorを呼び出して、トレーニング結果をグローバルモデルのlearnableオブジェクトに適用します。これによりグローバルモデルが更新されます。このステップは、トレーニング結果が重み差分のみを含む場合に必要です。重み差分は、次のクライアントのトレーニングのために直接送信することはできません。
    - クライアントがこのラウンドのシーケンスの最後であり、かつこのラウンドが最終ラウンドの場合、トレーニングは完了です。グローバルモデルを設定されたすべての結果クライアント(result clients)にブロードキャストします。
    - クライアントがこのラウンドのシーケンスの最後であるが、このラウンドが最終ラウンドでない場合、設定された順序ポリシー(固定またはランダム)に基づいて、次のラウンドのクライアントシーケンスを再計算します。
    - shareable generatorを呼び出して、グローバルモデルをshareableなモデルパラメータに変換します。これにより、次のクライアントのトレーニングのために、Learnableオブジェクト(単純な重みの辞書である場合もそうでない場合もあります)からモデルパラメータが抽出されます。
    - シーケンスの次のクライアントにモデルパラメータを送信します。

サイクリック学習ワークフローは、:class:`nvflare.app_common.ccwf.cyclic_server_ctl.CyclicServerController`\ (:class:`nvflare.app_common.ccwf.server_ctl.ServerSideController` のサブクラス)と
:class:`nvflare.app_common.ccwf.cyclic_client_ctl.CyclicClientController`\ (:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` のサブクラス)で実装されています。

サイクリック学習: サーバー側コントローラー
==================================================================================

.. code-block:: python

    class CyclicServerController(ServerSideController):
        def __init__(
            self,
            num_rounds: int,
            task_name_prefix=Constant.TN_PREFIX_CYCLIC,
            start_task_timeout=Constant.START_TASK_TIMEOUT,
            configure_task_timeout=Constant.CONFIG_TASK_TIMEOUT,
            task_check_period: float = Constant.TASK_CHECK_INTERVAL,
            job_status_check_interval: float = Constant.JOB_STATUS_CHECK_INTERVAL,
            participating_clients=None,
            result_clients=None,
            starting_client: str = "",
            max_status_report_interval: float = Constant.PER_CLIENT_STATUS_REPORT_TIMEOUT,
            progress_timeout: float = Constant.WORKFLOW_PROGRESS_TIMEOUT,
            private_p2p: bool = True,
            cyclic_order: str = CyclicOrder.FIXED,
        ):

追加の初期化引数は ``cyclic_order`` のみで、各ラウンドのサイクリックシーケンスをどのように計算するか(固定順序またはランダム順序)を指定します。

すべての初期化引数のうち、明示的に指定する必要があるのは ``num_rounds`` のみです。その他はすべてデフォルト値を使用できます:

    - ジョブのすべてのクライアントが参加
    - 開始クライアントはランダムに選択
    - すべてのクライアントが結果クライアントでもある - すべてのクライアントが最終結果を受け取る
    - クライアントシーケンスはすべてのラウンドで固定

サイクリック学習: クライアント側コントローラー
======================================================================================

.. code-block:: python

    class CyclicClientController(ClientSideController):
        def __init__(
            self,
            task_name_prefix=Constant.TN_PREFIX_CYCLIC,
            learn_task_name=AppConstants.TASK_TRAIN,
            persistor_id=AppConstants.DEFAULT_PERSISTOR_ID,
            shareable_generator_id=AppConstants.DEFAULT_SHAREABLE_GENERATOR_ID,
            learn_task_check_interval=Constant.LEARN_TASK_CHECK_INTERVAL,
            learn_task_abort_timeout=Constant.LEARN_TASK_ABORT_TIMEOUT,
            learn_task_ack_timeout=Constant.LEARN_TASK_ACK_TIMEOUT,
            final_result_ack_timeout=Constant.FINAL_RESULT_ACK_TIMEOUT,
        ):

追加の初期化引数はありません。

クライアント側では、このワークフローには次の3つのコンポーネントが必要です:

    - 指定された ``learn_task_name`` に対応するエグゼキューターが必要です
    - 指定された ``persistor_id`` に対応するpersistorコンポーネントが必要です
    - 指定された ``shareable_generator_id`` に対応するshareable generatorコンポーネントが必要です

最終結果が大きすぎてデフォルトのタイムアウトに収まらない場合は、``final_result_ack_timeout`` を適切に調整する必要があるかもしれません。

サイクリック学習の設定例
================================================================

サイクリック学習: config_fed_server.json
------------------------------------------------------------------------------

.. code-block:: json

    {
      "format_version": 2,
      "task_data_filters": [],
      "task_result_filters": [],
      "components": [],
      "workflows": [
        {
          "id": "rr",
          "path": "nvflare.app_common.ccwf.CyclicServerController",
          "args": {
            "num_rounds": 10
          }
        }
      ]
    }

サイクリック学習: config_fed_client.json
------------------------------------------------------------------------------

.. code-block:: json

    {
      "format_version": 2,
      "executors": [
        {
          "tasks": [
            "train"
          ],
          "executor": {
            "path": "nvflare.app_common.ccwf.comps.np_trainer.NPTrainer",
            "args": {}
          }
        },
        {
          "tasks": ["cyclic_*"],
          "executor": {
            "path": "nvflare.app_common.ccwf.CyclicClientController",
            "args": {
              "learn_task_name": "train",
              "persistor_id": "persistor",
              "shareable_generator_id": "shareable_generator"
            }
          }
        }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
        {
          "id": "persistor",
          "path": "nvflare.app_common.np.np_model_persistor.NPModelPersistor",
          "args": {}
        },
        {
          "id": "shareable_generator",
          "path": "nvflare.app_common.ccwf.comps.simple_model_shareable_generator.SimpleModelShareableGenerator",
          "args": {}
        }
      ]
    }

.. note::

    - ``cyclic_`` プレフィックスの付いたすべてのタスクは、CyclicClientController(エグゼキューターです)にルーティングされます。
    - CyclicServerControllerによって割り当てられるタスクは2つあります:
        - ``cyclic_config``
        - ``cyclic_start``
    - トレーニングプロセス中にクライアントによって割り当てられるタスクは2つあります:
        - ``cyclic_learn``: クライアントにトレーニングの実行を依頼するタスクです。
        - ``cyclic_report_final_learn_result``: 最終結果を保持するクライアントから他のクライアントに最終結果を報告するために送信されます


.. note::

    注記: configタスクとstartタスクには、モデル関連のデータは含まれません。


.. note::

    注記: ``cyclic_learn`` と ``cyclic_rcv_final_learn_result`` にはモデルデータが含まれます。プライバシーが懸念される場合は、``task_data_filters`` を適用できます(送信側クライアントにはOUTフィルター、受信側クライアントにはINフィルター)。

.. _ccwf_swarm_learning:

****************************************
スウォーム学習
****************************************
スウォーム学習(Swarm learning)は、連合学習の分散化された形態であり、集約とモデルトレーニング制御の責務を、
中央サーバーに集約するのではなく、すべてのピアに分散します。

スウォーム学習では、トレーニングは複数のラウンドで行われます。各ラウンドでは、すべてのクライアントの中から集約クライアントが
ランダムに選ばれ、すべてのトレーニングクライアントが現在のグローバルモデルパラメータに対してトレーニングタスクを実行します。完了すると、すべてのクライアントは
トレーニング結果を、集約用に指定されたクライアントに送信します。集約された結果は現在のグローバルモデルに適用され、
それが次のラウンドのトレーニングのベースになります。このプロセスは、設定されたラウンド数が完了するまで繰り返されます。

開始クライアントは初期モデルを担当し、初期モデルは設定されたpersistorによってロードされます。

ワークフローの終了時に、最終的なトレーニング結果は、最終結果を受け取るように設定されたすべてのクライアント(``result_clients``)にブロードキャストされます。

以下は、SwarmClientControllerの詳細な処理ロジックです:

    - ワークフローはstarting_clientから開始されます。persistorを使用して初期モデルをロードし、shareable generator(learnable_to_shareable)を使用して初期トレーニングパラメータを準備します。
    - 設定された"aggr_clients"リストから、次のラウンドの集約クライアントとなるクライアントをランダムに選択します。
    - トレーニング用に設定されたすべてのクライアント(training_clients)と集約クライアントに、トレーニングパラメータを含む"learn"タスクをブロードキャストします。タスクヘッダーには、集約クライアント名や現在のラウンド番号などが含まれます。
    - すべてのトレーニングクライアントは、``train`` タスクに設定されたエグゼキューターを呼び出してトレーニングを行います。
    - 完了すると、すべてのトレーニングクライアントは結果を集約クライアントに送信します。
    - "learn"タスクを受信すると、集約クライアントは次を行います:
        - shareable generatorを呼び出して、現在のグローバルモデルを計算します(``shareable_to_learnable``)。
        - トレーニングクライアントからの結果を待つためのGathererオブジェクトをセットアップします。集約クライアントがトレーニングクライアントでもある場合があることに注意してください。
    - 別のクライアントからトレーニング結果を受信すると、集約クライアントのGathererオブジェクトは、設定されたアグリゲーターを呼び出して結果を受け入れます。アグリゲーターの ``accept`` メソッド呼び出しの前(``AppEventType.BEFORE_CONTRIBUTION_ACCEPT``)と後(``AppEventType.AFTER_CONTRIBUTION_ACCEPT``)にイベントが発火されます。これらのイベントは、最良モデル選択の実装に非常に役立ちます。
    - すべての結果を受信した後(またはタイムアウトなどの他の終了条件が発生した後)、集約クライアントは次を行います:
        - アグリゲーターの ``aggregate`` メソッドを呼び出して集約結果を取得します。呼び出しの前(``AppEventType.BEFORE_AGGREGATION``)と後(``AppEventType.AFTER_AGGREGATION``)にイベントが発火されます。
        - shareable generatorを呼び出して、集約結果を現在のグローバルモデルに適用します(``shareable_to_learnable``)
        - すべてのラウンドが完了していない場合、次のラウンドの準備をします:
            - 次のラウンドの集約クライアントをランダムに選択します
            - shareable generatorを呼び出してトレーニングパラメータを準備します(learnable_to_shareable)。
            - 新しいラウンドのために他のクライアントに"learn"タスクをブロードキャストします
        - すべてのラウンドが完了した場合:
            - 最終結果をすべてのresult_clientsにブロードキャストします
            - どのクライアントが最良の結果を持っているかを確認し、そのクライアントに最良モデルをすべてのresult_clientsに配布するよう依頼します。

スウォーム学習ワークフローは、:class:`nvflare.app_common.ccwf.swarm_server_ctl.SwarmServerController`\ (:class:`nvflare.app_common.ccwf.server_ctl.ServerSideController` のサブクラス)と
:class:`nvflare.app_common.ccwf.swarm_client_ctl.SwarmClientController`\ (:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` のサブクラス)で実装されています。

最良モデルの選択
================================
オプションとして、サーバー制御の連合平均ワークフロー(SAG)と同様に、モデル選択ウィジェットを使用して最良のグローバルモデルを
決定できます。このウィジェットは、アグリゲーターの ``accept`` および ``aggregate`` 呼び出しのBEFOREおよびAFTERイベントをリッスンし、
トレーニングクライアントから報告された検証メトリクスの集約値を動的に計算します。より良いメトリクスが達成されると、
最良のメトリクス値と共に ``AppEventType.GLOBAL_BEST_MODEL_AVAILABLE`` イベントを発火します。persistorが
このイベントをリッスンしていれば、現在のグローバルモデル(現時点の最良)を永続化できます。

しかし、集約が常にサーバー上で行われ、常に単一のグローバルモデルしか存在しないサーバー制御のSAGとは異なり、
スウォーム学習の過程では多くのクライアントが集約を行う可能性があります。各集約クライアントは、
それぞれのモデルセレクターによって計算された、いわば自分の最良グローバルモデルを持つことができます。これらの最良グローバルモデルの中から
最良のものを見つける必要があります。これは次のように実現されます:

    - ``learn`` タスクのヘッダーを使用して、現在のグローバル最良(メトリクス値と、そのモデルを保持するクライアントの名前)を記憶します。初期状態ではどちらもNoneです。
    - SwarmClientControllerは ``AppEventType.GLOBAL_BEST_MODEL_AVAILABLE`` イベントをリッスンします。このイベントが発火されると、タスクヘッダー内の現在の最良値(存在する場合)とメトリクス値を比較します。新しい値の方が良ければタスクヘッダーを更新します。このヘッダー情報は次の ``learn`` タスクに引き継がれます。
    - 最終的に、グローバル最良(利用可能な場合)のみが結果クライアントに配布されます。

スウォーム学習: サーバー側コントローラー
================================================================================

.. code-block:: python

    class SwarmServerController(ServerSideController):
        def __init__(
            self,
            num_rounds: int,
            start_round: int = 0,
            task_name_prefix=Constant.TN_PREFIX_SWARM,
            start_task_timeout=Constant.START_TASK_TIMEOUT,
            configure_task_timeout=Constant.CONFIG_TASK_TIMEOUT,
            task_check_period: float = Constant.TASK_CHECK_INTERVAL,
            job_status_check_interval: float = Constant.JOB_STATUS_CHECK_INTERVAL,
            participating_clients=None,
            result_clients=None,
            starting_client: str = "",
            max_status_report_interval: float = Constant.PER_CLIENT_STATUS_REPORT_TIMEOUT,
            progress_timeout: float = Constant.WORKFLOW_PROGRESS_TIMEOUT,
            aggr_clients=None,
            train_clients=None,
        ):

タスク名プレフィックスのデフォルト値は"swarm"です。

追加の初期化引数は次のとおりです:

    - ``aggr_clients``: 集約を行うクライアント。指定しない場合、すべての参加クライアントが集約クライアントになります。
    - ``train_clients``: トレーニングを行うクライアント。指定しない場合、すべての参加クライアントがトレーニングクライアントになります。

スウォーム学習: クライアント側コントローラー
====================================================================================

.. code-block:: python

    class SwarmClientController(ClientSideController):
        def __init__(
            self,
            task_name_prefix=Constant.TN_PREFIX_SWARM,
            learn_task_name=AppConstants.TASK_TRAIN,
            persistor_id=AppConstants.DEFAULT_PERSISTOR_ID,
            shareable_generator_id=AppConstants.DEFAULT_SHAREABLE_GENERATOR_ID,
            aggregator_id=AppConstants.DEFAULT_AGGREGATOR_ID,
            metric_comparator_id=None,
            learn_task_check_interval=Constant.LEARN_TASK_CHECK_INTERVAL,
            learn_task_abort_timeout=Constant.LEARN_TASK_ABORT_TIMEOUT,
            learn_task_ack_timeout=Constant.LEARN_TASK_ACK_TIMEOUT,
            learn_task_timeout=None,
            final_result_ack_timeout=Constant.FINAL_RESULT_ACK_TIMEOUT,
            min_responses_required: int = 1,
            wait_time_after_min_resps_received: float = 10.0,
        ):

クライアント側では、このワークフローには次のコンポーネントが必要です:

    - 指定された ``learn_task_name`` に対応するエグゼキューターが必要です
    - 指定された ``persistor_id`` に対応するpersistorコンポーネントが必要です
    - 指定された ``shareable_generator_id`` に対応するshareable generatorコンポーネントが必要です
    - 指定された ``aggregator_id`` に対応するアグリゲーターコンポーネントが必要です。
    - ``metric_comparator_id`` が指定されている場合は、オプションのMetric Comparatorが必要です。メトリクス値は任意の型を取り得るため、スウォーム学習ワークフローでは現在の最良メトリクスと計算されたメトリクス値を比較できる必要があり、Metric Comparatorがこの比較操作を助けます。この引数が設定されていない場合は、メトリクス値が単純な数値であることを前提とする ``NumberMetricComparator`` が使用されます。

集約の動作は、次の引数で設定されます:

    - ``min_responses_required`` - 収集(gathering)を終了する前に必要な最小応答数
    - ``wait_time_after_min_resps_received`` - 最小応答数を受信した後、さらなる応答の可能性を待つ秒数
    - ``learn_task_timeout`` - 収集をタイムアウトさせるまで、現在のlearnタスクをどれだけ待つか

スウォーム学習の例
====================================

このセクションでは、レシピを使用する方法(推奨)と従来のJSON設定を使用する方法で、スウォーム学習をセットアップする方法を示します。

レシピの使用(推奨)
------------------------------------------------------

効率的なスウォーム学習のセットアップには ``SwarmLearningRecipe`` を使用します:

.. code-block:: python

    from nvflare.app_opt.pt.recipes.swarm import SwarmLearningRecipe
    from nvflare.recipe.sim_env import SimEnv

    # Create swarm learning recipe
    # Model can be class instance or dict config
    # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
    recipe = SwarmLearningRecipe(
        name="swarm_learning",
        model=MyModel(),
        min_clients=3,
        num_rounds=10,
        train_script="train.py",
        train_args={"batch_size": 32, "epochs": 5},
        progress_timeout=7200,
        learn_task_timeout=None,       # No per-task time limit
        learn_task_ack_timeout=3600,   # P2P task-transfer ACK budget
        final_result_ack_timeout=3600, # P2P final-result ACK budget
        max_concurrent_submissions=1,
    )

    # Configure large model parameters if needed (server-side only)
    recipe.add_server_config({
        "np_download_chunk_size": 2097152,
        "tensor_download_chunk_size": 2097152,
        "streaming_per_request_timeout": 600
    })

    # Run in simulation
    env = SimEnv(num_clients=3)
    recipe.execute(env)

名前付きパラメータが推奨APIです。あまり一般的でない ``SwarmServerConfig`` や ``SwarmClientConfig`` のフィールドについては、
``server_config_overrides`` または ``client_config_overrides`` を渡してください。これらの辞書は最後に
浅くマージ(shallow-merge)されるため、重複する辞書の値は意図的に名前付きパラメータより優先されます。
``round_timeout`` は、明示的なパラメータが省略された場合に両方の確認応答(ACK)タイムアウトを設定する互換ショートカットとして
引き続き利用できます。``client_config_overrides`` は、レシピが管理するエグゼキューター、アグリゲーター、persistor、
shareable generator、``min_responses_required`` を置き換えることはできません。カスタムコンポーネントやクォーラム設定には
``BaseSwarmLearningRecipe`` を使用してください。スケジューラー、サーバーコントローラー、クライアント集約のクォーラムの
整合性を保つため、``min_clients`` は名前付きパラメータでのみ設定してください。

高度なカスタマイズには、サーバーとクライアントの設定を明示的に指定して ``BaseSwarmLearningRecipe`` を使用します:

.. code-block:: python

    from nvflare.app_common.ccwf.recipes.swarm import BaseSwarmLearningRecipe
    from nvflare.app_common.ccwf.ccwf_job import SwarmServerConfig, SwarmClientConfig

    server_config = SwarmServerConfig(
        num_rounds=10,
        start_task_timeout=300,
        progress_timeout=7200,
    )

    client_config = SwarmClientConfig(
        executor=my_executor,
        aggregator=my_aggregator,
        persistor=my_persistor,
        shareable_generator=my_generator,
    )

    recipe = BaseSwarmLearningRecipe(
        name="custom_swarm",
        server_config=server_config,
        client_config=client_config,
    )

.. note::
   注記: ``BaseSwarmLearningRecipe`` で明示的な ``SwarmClientConfig`` を使用する場合は、大規模モデルに対して
   ``learn_task_ack_timeout`` と ``final_result_ack_timeout`` を手動で設定してください。``SwarmLearningRecipe`` では、
   対応する名前付きパラメータを優先してください。``round_timeout`` は互換ショートカットとして引き続き両方の値を設定できます。

クライアント脱落の許容(min_clients)
----------------------------------------------------------------------------

``min_clients`` を設定すると、少なくともその数のクライアントが設定に成功すればワークフローを進行させることができます。
不足している参加者はジョブの中断を引き起こすのではなく、警告としてログに記録されます。

.. code-block:: python

    recipe = SwarmLearningRecipe(
        name="swarm",
        model=MyModel(),
        min_clients=3,    # Workflow proceeds if >= 3 of the configured clients are ready;
        num_rounds=10,    # remaining clients are logged as warnings
        train_script="train.py",
    )

``min_clients=0`` を設定すると、設定されたすべてのクライアントが必須になります(後方互換の動作)。
これは、デプロイメントフェーズを制御するジョブスケジューラーの ``min_clients`` パラメータとは別物です。

JSON設定の使用(上級者向け)
----------------------------------------------------------------------

きめ細かな制御が必要なユーザー向けに、同等のJSON設定を示します。

**config_fed_server.json:**

.. code-block:: json

    {
      "format_version": 2,
      "task_data_filters": [],
      "task_result_filters": [],
      "components": [],
      "workflows": [
        {
          "id": "swarm_controller",
          "path": "nvflare.app_common.ccwf.SwarmServerController",
          "args": {
            "num_rounds": 10
          }
        }
      ]
    }

.. note::

    注記: 必須の引数は ``num_rounds`` のみです。

**config_fed_client.json:**

.. code-block:: json

    {
      "format_version": 2,
      "executors": [
        {
          "tasks": [
            "train"
          ],
          "executor": {
            "path": "nvflare.app_common.ccwf.comps.np_trainer.NPTrainer",
            "args": {}
          }
        },
        {
          "tasks": ["swarm_*"],
          "executor": {
            "path": "nvflare.app_common.ccwf.SwarmClientController",
            "args": {
              "learn_task_name": "train",
              "learn_task_timeout": 5.0,
              "persistor_id": "persistor",
              "aggregator_id": "aggregator",
              "shareable_generator_id": "shareable_generator",
              "min_responses_required": 2,
              "wait_time_after_min_resps_received": 1
            }
          }
        }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
        {
          "id": "persistor",
          "path": "nvflare.app_common.ccwf.comps.np_file_model_persistor.NPFileModelPersistor",
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
            "expected_data_kind": "WEIGHT_DIFF"
          }
        },
        {
          "id": "model_selector",
          "path": "nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector",
          "args": {}
        }
      ]
    }

.. note::

    - ``swarm_`` プレフィックスの付いたすべてのタスクは、:class:`nvflare.app_common.ccwf.swarm_client_ctl.SwarmClientController`\ (エグゼキューターです)にルーティングされます。

.. note::

    - :class:`nvflare.app_common.ccwf.swarm_server_ctl.SwarmServerController` によって割り当てられるタスクは2つあります:
        - swarm_config
        - swarm_start

.. note::

    - トレーニングプロセス中にクライアントによって割り当てられるタスクがいくつかあります:
        - swarm_learn: クライアントにトレーニングの実行を依頼するタスクです。
        - swarm_report_learn_result: トレーニングクライアントから集約クライアントに、トレーニング結果を報告するために送信されます。
        - swarm_report_final_learn_result: 最終結果(最終および/または最良のグローバルモデル)を保持するクライアントから他のクライアントに、最終結果を報告するために送信されます


.. note::

    注記: swarm_configタスクとswarm_startタスクには、モデル関連のデータは含まれません。


.. note::

    注記: クライアントが割り当てるタスクにはモデルデータが含まれます。プライバシーが懸念される場合は、task_data_filtersを適用できます(送信側クライアントにはOUTフィルター、受信側クライアントにはINフィルター)。

.. _swarm_learning_large_models:

大規模モデル向けのスウォーム学習パラメータ
====================================================================================

大規模モデル(例: LLM)でスウォーム学習を実行する場合、より大きなペイロードとより長い処理時間に対応するために、
さまざまなタイムアウトおよびチャンク化パラメータの調整が必要になることがあります。

デフォルトのタイムアウト値
----------------------------------------------------

次の表は、スウォーム学習で使用されるすべてのデフォルトのタイムアウト値を示しています。これらのデフォルト値を理解することで、
大規模モデルのワークロードに対してどのパラメータを調整すべきかを判断しやすくなります。

**これらの値を上書きする方法:**

これらのタイムアウト値は、ジョブ設定ファイルで上書きできます:

- **クライアント側パラメータ**: ``config_fed_client.conf``\ (または ``.json``)の ``SwarmClientController`` エグゼキューターのargsに設定します。
- **サーバー側パラメータ**: ``config_fed_server.conf``\ (または ``.json``)の ``SwarmServerController`` ワークフローのargsに設定します。
- **グローバルストリーミングパラメータ**: 設定ファイルのトップレベルに設定します(例: ``np_download_chunk_size``\ 、``tensor_download_chunk_size``)。

**Recipe APIユーザー**\ (推奨)は、``add_server_config()`` メソッドを使用します:

.. code-block:: python

    # Add streaming parameters to server config (server-side only)
    recipe.add_server_config({
        "np_download_chunk_size": 2097152,
        "tensor_download_chunk_size": 2097152,
        "streaming_per_request_timeout": 600
    })

**Job APIユーザー**\ は、辞書を渡して ``job.to_server()`` を使用します:

.. code-block:: python

    job.to_server({"np_download_chunk_size": 2097152, "streaming_per_request_timeout": 600})

.. list-table:: スウォーム学習のデフォルトタイムアウト
   :header-rows: 1
   :widths: 25 10 25 40

   * - 定数名
     - デフォルト
     - 設定パラメータ
     - 説明
   * - ``CONFIG_TASK_TIMEOUT``
     - 300
     - ``config_task_timeout``\ (サーバー)
     - ジョブ開始時に、クライアントが設定タスクに応答するために許容される時間。
   * - ``START_TASK_TIMEOUT``
     - 10
     - ``start_task_timeout``\ (サーバー)
     - 開始クライアントがワークフローを開始するために許容される時間。
   * - ``END_WORKFLOW_TIMEOUT``
     - 2.0
     - ``end_workflow_timeout``\ (サーバー)
     - ワークフロー終了メッセージの確認応答に許容される時間。
   * - ``TASK_CHECK_INTERVAL``
     - 0.5
     - ``task_check_interval``\ (クライアント)
     - タスクステータスをチェックする間隔。
   * - ``JOB_STATUS_CHECK_INTERVAL``
     - 2.0
     - ``job_status_check_interval``\ (サーバー)
     - サーバーがジョブステータスをチェックする間隔。
   * - ``PER_CLIENT_STATUS_REPORT_TIMEOUT``
     - 90.0
     - (内部)
     - クライアントがステータスを報告しないまま経過できる最大時間。
   * - ``WORKFLOW_PROGRESS_TIMEOUT``
     - 3600.0
     - ``progress_timeout``\ (サーバー)
     - ワークフローの進捗が一切ない状態で許容される最大時間。
   * - ``LEARN_TASK_CHECK_INTERVAL``
     - 1.0
     - ``learn_task_check_interval``\ (クライアント)
     - 新しい学習タスクをチェックする間隔。
   * - ``LEARN_TASK_ACK_TIMEOUT``
     - 10
     - ``learn_task_ack_timeout``\ (クライアント)
     - クライアントがlearnタスクの受領を確認応答するために許容される時間。
   * - ``LEARN_TASK_ABORT_TIMEOUT``
     - 5.0
     - ``learn_task_abort_timeout``\ (クライアント)
     - 要求された際に学習タスクが中断するために許容される時間。
   * - ``FINAL_RESULT_ACK_TIMEOUT``
     - 10
     - ``final_result_ack_timeout``\ (クライアント)
     - クライアントが最終結果の受領を確認応答するために許容される時間。
   * - ``GET_MODEL_TIMEOUT``
     - 10
     - ``get_model_timeout``\ (クライアント)
     - 別のクライアントからモデルを取得するために許容される時間。
   * - ``MAX_TASK_TIMEOUT``
     - 3600
     - ``learn_task_timeout``\ (クライアント)
     - 単一のタスクが完了するために許容される最大時間。

クライアント側パラメータ
------------------------------------------------

大規模モデルでは、以下のSwarmClientControllerパラメータが特に重要です:

**タイムアウトとフロー制御:**

- ``learn_task_timeout``: 集約クライアントがラウンドの完了を待つ時間の上限。\ **デフォルト: None**\ 。大規模モデルでは\ **推奨: 3600〜7200**\ 。
- ``learn_task_ack_timeout``: learnタスク送信の確認応答のタイムアウト。\ **デフォルト: 10**\ 。大規模モデルの初期化は遅くなる可能性があるため、\ **推奨: 300以上**\ 。
- ``final_result_ack_timeout``: 最終結果のブロードキャスト後のACKのタイムアウト。\ **デフォルト: 10**\ 。最終結果の配布は多くの場合最大のペイロードになるため、\ **推奨: 300〜600**\ 。
- ``request_to_submit_result_msg_timeout``: 結果送信要求メッセージのタイムアウト。\ **デフォルト: 5.0**\ 。\ **推奨: 10〜30**\ 。
- ``request_to_submit_result_interval``: 送信許可が得られなかった場合のリトライ間隔。\ **デフォルト: 1.0**\ 。\ **推奨: 2〜5**\ 。
- ``request_to_submit_result_max_wait``: 送信許可を待つ合計時間の上限。\ **デフォルト: None**\ 。大規模モデルでは\ **推奨: 600〜1200**\ 。
- ``max_concurrent_submissions``: 同時送信の最大数。\ **デフォルト: 1**\ 。メモリ圧迫を軽減するため\ **推奨: 1**\ 。
- ``min_responses_required``: 集約を開始するために必要な最小クライアント結果数。\ **デフォルト: 1**\ 。3クライアント実行では\ **推奨: 2**\ 。
- ``wait_time_after_min_resps_received``: 最小応答数受信後の追加待機時間。\ **デフォルト: 10.0**\ 。\ **推奨: 120〜300**\ 。

**大規模モデル向けのクライアント設定例:**

.. code-block::

    executors = [
      {
        tasks = ["swarm_*"]
        executor {
          path = "nvflare.app_common.ccwf.SwarmClientController"
          args {
            learn_task_timeout = 3600
            learn_task_ack_timeout = 300
            final_result_ack_timeout = 300
            request_to_submit_result_msg_timeout = 10.0
            request_to_submit_result_interval = 2.0
            request_to_submit_result_max_wait = 600.0
            max_concurrent_submissions = 1
            min_responses_required = 2
            wait_time_after_min_resps_received = 120
          }
        }
      }
    ]

**ダウンロードとチャンク化の動作:**

- ``np_download_chunk_size``: numpy配列のダウンロードのチャンクサイズ。\ **デフォルト: 2097152 (2MB)**\ 。値0はストリーミングを無効化し、ネイティブなシリアライズを使用するためメモリが急増する可能性があります。
- ``tensor_download_chunk_size``: PyTorchテンソルのダウンロードのチャンクサイズ。\ **デフォルト: 2097152 (2MB)**\ 。値0はストリーミングを無効化します。

.. code-block::

    np_download_chunk_size = 2097152
    tensor_download_chunk_size = 2097152

サーバー側パラメータ
--------------------------------------------

**SwarmServerController:**

- ``num_rounds``: トレーニングラウンドの総数。
- ``start_task_timeout``: ワークフロー開始のタイムアウト。\ **デフォルト: 10 (START_TASK_TIMEOUT)**\ 。大規模モデルの初期化には\ **推奨: 300**\ 。
- ``progress_timeout``: ワークフロー全体の進捗タイムアウト。\ **デフォルト: 3600.0 (WORKFLOW_PROGRESS_TIMEOUT)**\ 。大規模モデルでは\ **推奨: 7200以上**\ 。

**大規模モデル向けのサーバー設定例:**

.. code-block::

    workflows = [
      {
        id = "swarm_controller"
        path = "nvflare.app_common.ccwf.SwarmServerController"
        args {
          num_rounds = 25
          start_task_timeout = 300
          progress_timeout = 7200
        }
      }
    ]

**CrossSiteEvalServerController(有効化する場合):**

- ``eval_task_timeout``: 評価タスクのタイムアウト。\ **デフォルト: 300 (CONFIG_TASK_TIMEOUT)**\ 。大規模モデルでは\ **推奨: 1200**\ 。

オプションのNVFlareグローバル設定
------------------------------------------------------------------

これらのフレームワークレベルの設定は、大きなペイロードの転送に影響します:

- ``streaming_per_request_timeout``: ストリーミングダウンロードのリクエストごとのタイムアウト。\ **デフォルト: 600**\ 。大規模モデルでは\ **推奨: 600以上**\ 。

.. code-block::

    streaming_per_request_timeout: 600

推奨される最小限のパラメータセット
------------------------------------------------------------------

大規模モデル向けに少数のパラメータのみ調整する場合は、以下から始めてください:

1. ``learn_task_timeout`` - ラウンドの完了に十分な時間を確保します
2. ``final_result_ack_timeout`` - 大きな結果の配布に十分な時間を確保します
3. ``request_to_submit_result_max_wait`` - 適切な集約ウィンドウを確保します
4. ``progress_timeout`` - ワークフローの早すぎる終了を防ぎます
5. ``np_download_chunk_size`` と ``tensor_download_chunk_size`` - メモリ効率の良いストリーミングを有効にします

.. _ccwf_cross_site_evaluation:

**********************************************
クロスサイト評価
**********************************************

クロスサイト評価(Cross Site Evaluation: CSE)ワークフローの目的は、クライアントサイトが互いのモデルを評価できるようにすることです。オプションで、追加のグローバルモデルをクライアントが評価することもできます。

サーバー制御のCSEでは、各サイトは最初に自分のモデルをサーバーに送信し、サーバーがそのモデルを他のサイトにブロードキャストして評価させます。サーバーは、サーバーが所有する追加のモデルを他のサイトに送信して評価させることもできます。すべてのモデル評価結果はサーバーに送り返されるため、ユーザーは結果に容易にアクセスできます。

クライアント制御のCSEでは、クライアントのモデルは配布のためにサーバーに送られません。代わりに、クライアント同士が直接通信して、検証のためにモデルを共有します。モデル評価結果は引き続きサーバーに送信されるため、ユーザーは結果に容易にアクセスできます。

クライアント制御のCSEには、いくつかの概念があります:

  - 評価者(Evaluators) - モデルを評価し、評価メトリクスを生成するクライアント。
  - 被評価者(Evaluatees) - 評価対象のローカルモデルを持つクライアント
  - グローバルモデルクライアント - 評価対象のグローバルモデルを持つクライアント

CSEクライアント制御ワークフローは、ローカルモデルとグローバルモデルのいずれか、または両方の評価に使用できます。

以下は詳細な制御ロジックです:

  - サーバーはすべてのクライアントに"config"タスクをブロードキャストします。configには、誰が評価者・被評価者であるか、どのクライアントがグローバルモデルクライアントであるかの情報が含まれます。
  - 各クライアントはconfig情報を処理します。グローバルモデルクライアントとして設定されている場合、グローバルモデル名をサーバーに送信します。評価者として設定されている場合、評価機能を持っているかどうかを確認します。持っていない場合、サーバーにエラーを報告します。被評価者として設定されている場合、ローカルモデルを持っているかどうかを確認します。持っていない場合、サーバーにエラーを報告します。
  - サーバーはすべてのクライアントからの設定応答を処理します。エラーが報告された場合、ジョブは中断されます。
  - サーバーはまず、グローバルモデルクライアントがモデル名を報告していれば、グローバルモデルの評価を試みます。各グローバルモデル名について、サーバーはすべての評価者に"eval"リクエストをブロードキャストしてモデルを評価させます。このリクエストには、モデルの名前と、モデルを持つクライアントの名前のみが含まれます。
  - 次にサーバーは、クライアントのローカルモデルの評価を試みます。被評価者として設定された各クライアントについて、サーバーはすべての評価者に"eval"リクエストをブロードキャストします。このリクエストには被評価者の名前が含まれます。
  - クライアント側では、"eval"リクエストを受信すると、次を行います:
  - モデルを持つクライアントに"get_model"タスクを送信します。
  - 受信したモデルに対して"validate"メソッドを実行します。
  - 結果をサーバーに送り返します
  - クライアント側では、"get_model"タスクを受信すると、モデルのタイプに応じてモデルを特定します:
  - グローバルモデルの場合、persistorオブジェクトを呼び出してモデルを特定します
  - ローカルモデルの場合、"submit_model"タスクに設定されたエグゼキューターを呼び出します。
  - サーバー側では、評価結果を受信すると、次を行います:
  - AppEventType.VALIDATION_RESULT_RECEIVEDイベントタイプを発火し、他のウィジェットが結果を処理できるようにします
  - サーバー制御のCSEと同じフォルダー構造を使用して、ジョブのワークスペースに保存します。

CSEワークフローは、:class:`nvflare.app_common.ccwf.cse_server_ctl.CrossSiteEvalServerController`\ (:class:`nvflare.app_common.ccwf.server_ctl.ServerSideController` のサブクラス)と
:class:`nvflare.app_common.ccwf.cse_client_ctl.CrossSiteEvalClientController`\ (:class:`nvflare.app_common.ccwf.client_ctl.ClientSideController` のサブクラス)で実装されています。

クロスサイト評価: サーバー側コントローラー
==========================================================================================

.. code-block:: python

    class CrossSiteEvalServerController(ServerSideController):
        def __init__(
            self,
            task_name_prefix=Constant.TN_PREFIX_CROSS_SITE_EVAL,
            start_task_timeout=Constant.START_TASK_TIMEOUT,
            configure_task_timeout=Constant.CONFIG_TASK_TIMEOUT,
            eval_task_timeout=30,
            task_check_period: float = Constant.TASK_CHECK_INTERVAL,
            job_status_check_interval: float = Constant.JOB_STATUS_CHECK_INTERVAL,
            progress_timeout: float = Constant.WORKFLOW_PROGRESS_TIMEOUT,
            participating_clients=None,
            evaluators=None,
            evaluatees=None,
            global_model_client=None,
            max_status_report_interval: float = Constant.PER_CLIENT_STATUS_REPORT_TIMEOUT,
            eval_result_dir=AppConstants.CROSS_VAL_DIR,
        ):

タスク名プレフィックスのデフォルト値は"cse"です。

追加の初期化引数は次のとおりです:

``eval_task_timeout`` - クライアントによるモデル評価に許容される最大時間。

``evaluators`` - モデルを評価するクライアント。デフォルトでは、すべてのクライアントが評価者です。

``evaluatees`` - モデルが評価されるクライアント。デフォルトでは、すべてのクライアントが被評価者です。ローカルモデルを評価しない場合は、この引数に特別な値"@none"を設定できます。

``global_model_client`` - 評価対象のグローバルモデルを持つクライアント。デフォルトでは、クライアントのリストからランダムに1つのクライアントが選択されます。グローバルモデルを評価したくない場合は、この引数に特別な値"@none"を設定できます。

``evaluatees`` と ``global_model_client`` の両方を"@none"に設定することはできません。


クロスサイト評価: クライアント側コントローラー
==============================================================================================


.. code-block:: python

    class CrossSiteEvalClientController(ClientSideController):
        def __init__(
            self,
            task_name_prefix=Constant.TN_PREFIX_CROSS_SITE_EVAL,
            submit_model_task_name=AppConstants.TASK_SUBMIT_MODEL,
            validation_task_name=AppConstants.TASK_VALIDATION,
            persistor_id=AppConstants.DEFAULT_PERSISTOR_ID,
            get_model_timeout=Constant.GET_MODEL_TIMEOUT,
        ):

タスク名プレフィックスのデフォルト値は"cse"です。

追加の初期化引数は次のとおりです:

``submit_model_task_name`` - モデルを提出するタスクの名前。これは、ローカル最良モデルの提出をすでにサポートしているトレーナーエグゼキューターにマッピングされる必要があります。

``validation_task_name`` - モデルを検証するタスクの名前。これは、モデル検証をすでにサポートしているトレーナーエグゼキューターにマッピングされる必要があります。

``get_model_timeout`` - クライアントXがクライアントYのモデルを評価しようとする場合、クライアントXはまずYにモデルを要求するリクエストを送信します。この引数は、このリクエストのタイムアウトを設定します。

Model Persistor
---------------
CSEワークフローでは、グローバルモデルクライアントは ``get_model_inventory`` メソッドを実装したModel Persistorを持っている必要があります。
このメソッドは、利用可能なグローバルモデルの名前を返すために呼び出されます。persistorは ``get_model`` メソッドも実装する必要があります。
このメソッドは、他のクライアントが評価するために、persistorからモデルを取得する際に呼び出されます。

クロスサイト評価の例
==========================================

このセクションでは、レシピを使用する方法(推奨)と従来のJSON設定を使用する方法で、クロスサイト評価をセットアップする方法を示します。

レシピの使用(推奨)
------------------------------------------------------

**クロスサイト評価付きスウォーム学習:**

クロスサイト評価をオプションで有効にしたスウォーム学習には、``SwarmLearningRecipe`` を使用します:

.. code-block:: python

    from nvflare.app_opt.pt.recipes.swarm import SwarmLearningRecipe
    from nvflare.recipe.sim_env import SimEnv

    # Create swarm learning recipe with cross-site evaluation enabled
    # Model can be class instance or dict config
    # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
    recipe = SwarmLearningRecipe(
        name="swarm_with_cse",
        model=MyModel(),
        min_clients=3,
        num_rounds=3,
        train_script="train.py",
        do_cross_site_eval=True,
        cross_site_eval_timeout=300,
        round_timeout=3600,   # P2P model-transfer ACK budget; increase for large models (7B+)
    )

    # Configure large model parameters if needed (server-side only)
    recipe.add_server_config({
        "np_download_chunk_size": 2097152,
        "tensor_download_chunk_size": 2097152,
        "streaming_per_request_timeout": 600
    })

    # Run in simulation
    env = SimEnv(num_clients=3)
    recipe.execute(env)

.. note::

    注記: このレシピは、クライアントが互いのモデルを直接評価するCCWFのピアツーピアなクロスサイト評価を使用します。
    従来のサーバー制御によるクロスサイト評価については、:ref:`cross_site_model_evaluation` を参照してください。

JSON設定の使用(上級者向け)
----------------------------------------------------------------------

きめ細かな制御が必要なユーザー向けに、同等のJSON設定を示します。

クロスサイト評価: config_fed_server.json
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: json

    {
      "format_version": 2,
      "task_data_filters": [],
      "task_result_filters": [],
      "components": [
        {
          "id": "json_generator",
          "path": "nvflare.app_common.widgets.validation_json_generator.ValidationJsonGenerator",
          "args": {}
        }
      ],
      "workflows": [
        {
          "id": "swarm_controller",
          "path": "nvflare.app_common.ccwf.SwarmServerController",
          "args": {
            "num_rounds": 3
          }
        },
        {
          "id": "cross_site_eval",
          "path": "nvflare.app_common.ccwf.CrossSiteEvalServerController",
          "args": {
          }
        }
      ]
    }


.. note::

    注記: json_generatorコンポーネントは、ジョブの終了時に、サイト横断検証の結果を人間が読める形式で
    示すJSONファイルを作成するためにも使用されます。

クロスサイト評価: config_fed_client.json
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: json

    {
      "format_version": 2,
      "executors": [
        {
          "tasks": [
            "train", "submit_model", "validate"
          ],
          "executor": {
            "path": "nvflare.app_common.ccwf.comps.np_trainer.NPTrainer",
            "args": {}
          }
        },
        {
          "tasks": ["swarm_*"],
          "executor": {
            "path": "nvflare.app_common.ccwf.SwarmClientController",
            "args": {
              "learn_task_name": "train",
              "learn_task_timeout": 5.0,
              "persistor_id": "persistor",
              "aggregator_id": "aggregator",
              "shareable_generator_id": "shareable_generator",
              "min_responses_required": 2,
              "wait_time_after_min_resps_received": 1
            }
          }
        },
        {
          "tasks": ["cse_*"],
          "executor": {
            "path": "nvflare.app_common.ccwf.CrossSiteEvalClientController",
            "args": {
              "submit_model_task_name": "submit_model",
              "validation_task_name": "validate",
              "persistor_id": "persistor"
            }
          }
        }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
        {
          "id": "persistor",
          "path": "nvflare.app_common.ccwf.comps.np_file_model_persistor.NPFileModelPersistor",
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
            "expected_data_kind": "WEIGHT_DIFF"
          }
        },
        {
          "id": "model_selector",
          "path": "nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector",
          "args": {}
        }
      ]
    }

.. note::

      - ``cse_`` プレフィックスの付いたすべてのタスクは、:class:`nvflare.app_common.ccwf.cse_client_ctl.CrossSiteEvalClientController`\ (エグゼキューターです)にルーティングされます。
      - 以下のタスクは :class:`nvflare.app_common.ccwf.cse_server_ctl.CrossSiteEvalServerController` によって割り当てられます:
        - ``cse_config``
        - ``cse_eval``
      - 以下のタスクは、トレーニングプロセス中にクライアントによって割り当てられます:
        - ``cse_ask_for_model``: あるクライアントから別のクライアントに、評価のためにモデルを要求するために送信されます。

.. note::

    注記: このワークフローには"start"タスクはありません。

.. note::

    注記: ``cse_config`` タスクと ``cse_eval`` タスクには機密性の高いモデルデータは含まれません。

.. note::

    注記: ``ask_for_model`` タスクへの応答にはモデルデータが含まれます。プライバシーが懸念される場合は、``task_result_filters`` を適用できます(応答側クライアントにはOUTフィルター、要求側クライアントにはINフィルター)。

大規模モデル向けのクロスサイト評価パラメータ
======================================================================================

大規模モデルでクロスサイト評価を実行する場合、より大きなモデル転送とより長い評価時間に対応するために、タイムアウトパラメータの調整が必要になることがあります。

サーバー側パラメータ
--------------------------------------------

**CrossSiteEvalServerController:**

- ``eval_task_timeout``: クライアントによるモデル評価に許容される最大時間。評価は高コストになり得るため、大規模モデルでは\ **推奨: 1200以上**\ 。
- ``configure_task_timeout``: 設定タスクのタイムアウト。大規模モデルの初期化には\ **推奨: 300**\ 。
- ``progress_timeout``: ワークフロー全体の進捗タイムアウト。大規模モデルでは\ **推奨: 7200以上**\ 。

**大規模モデル向けのサーバー設定例:**

.. code-block:: json

    {
      "id": "cross_site_eval",
      "path": "nvflare.app_common.ccwf.CrossSiteEvalServerController",
      "args": {
        "eval_task_timeout": 1200,
        "configure_task_timeout": 300,
        "progress_timeout": 7200
      }
    }

クライアント側パラメータ
------------------------------------------------

**CrossSiteEvalClientController:**

- ``get_model_timeout``: 別のクライアントにモデルを要求する際のタイムアウト。大規模モデルでは\ **推奨: 600以上**\ 。

**大規模モデル向けのクライアント設定例:**

.. code-block:: json

    {
      "tasks": ["cse_*"],
      "executor": {
        "path": "nvflare.app_common.ccwf.CrossSiteEvalClientController",
        "args": {
          "submit_model_task_name": "submit_model",
          "validation_task_name": "validate",
          "persistor_id": "persistor",
          "get_model_timeout": 600
        }
      }
    }

ダウンロードとチャンク化の動作
------------------------------------------------------------

クロスサイト評価中の大規模モデル転送では、チャンク化が設定されていることを確認してください:

- ``np_download_chunk_size``: NumPy配列のダウンロードのチャンクサイズ。\ **推奨: 2097152 (2MB)**
- ``tensor_download_chunk_size``: PyTorchテンソルのダウンロードのチャンクサイズ。\ **推奨: 2097152 (2MB)**

チャンク化設定の詳細については、:ref:`swarm_learning_large_models` を参照してください。
