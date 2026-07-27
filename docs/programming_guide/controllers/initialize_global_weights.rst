.. _initialize_global_weights_workflow:

クライアント側でのグローバルモデル初期化のための Initialize Global Weights ワークフロー
--------------------------------------------------------------------------------------------
SAG コントローラーは、トレーニング開始前にグローバルモデル重みが初期化されていることを必要とします。現在、初期重みを提供するのは
Persistor コンポーネントの役割であり、事前に定義されたモデルファイルから読み込むか、カスタム Python コードを実行して動的に生成します。
1つ目のアプローチにはモデルファイルを定義する手間がかかり、2つ目のアプローチにはカスタム Python コードが必要で、
これはセキュリティリスクになり得ます。

そこで、FL クライアントの初期重みに基づいて初期モデル重みを生成する3つ目のアプローチを導入します。すなわち、クライアントから
重みを収集するコントローラーを作成し、このコントローラーを SAG の前に配置するのです。このコントローラーが
:class:`nvflare.app_common.workflows.initialize_global_weights.InitializeGlobalWeights` です。

InitializeGlobalWeights の使い方
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

以下の手順は、NVFLARE リポジトリの ``hello-pt`` サンプルに基づいています。

ステップ 1: config_fed_server.json を修正する
""""""""""""""""""""""""""""""""""""""""""""""""""

2つの変更が必要です:

    - PTFileModelPersistor コンポーネント設定から ``model`` 引数を削除する。
    - ワークフローの最初のコントローラーとして ``InitializeGlobalWeights`` コントローラーを追加する。

更新後のファイルは次のようになります:

.. literalinclude:: ../../resources/init_weights_1_config_fed_server.json
   :language: json


``PTFileModelPersistor`` は、モデルオブジェクトとしてカスタムの ``SimpleNetwork`` を必要としなくなった点に注意してください。

InitializeGlobalWeights の設定における ``task_name`` の値 "get_weights" に注意してください。

ステップ 2: config_fed_client.json を修正する
""""""""""""""""""""""""""""""""""""""""""""""""""

以下でハイライトされているように、トレーナーにタスク "get_weights" を追加します。このタスク名は、
config_fed_server.json 内の InitializeGlobalWeights 設定の task_name と一致していなければならない点に注意してください。

.. code-block:: json
    :emphasize-lines: 8

    {
      "format_version": 2,
      "executors": [
          {
              "tasks": [
                  "train",
                  "submit_model",
                  "get_weights"
              ],
              "executor": {
                  "path": "cifar10trainer.Cifar10Trainer",
                  "args": {
                      "lr": 0.01,
                      "epochs": 1
                  }
              }
          },
          {
              "tasks": [
                  "validate"
              ],
              "executor": {
                  "path": "cifar10validator.Cifar10Validator",
                  "args": {}
              }
          }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": []
    }


ステップ 3: Trainer コードを更新する
""""""""""""""""""""""""""""""""""""""""
Executor である Cifar10Trainer に、新しい ``pre_train_task_name``\ (デフォルトは "get_weights")が追加されます。

Trainer は、このタスクを処理して現在のモデル重み(ランダムに初期化されているはずのもの)を返すように拡張されます。

関連するコードスニペットは以下のとおりです:


.. code-block:: python

    class Cifar10Trainer(Executor):

        def __init__(self, lr=0.01, epochs=5,
                train_task_name=AppConstants.TASK_TRAIN,
                submit_model_task_name=AppConstants.TASK_SUBMIT_MODEL,
                exclude_vars=None,
                pre_train_task_name=AppConstants.TASK_GET_WEIGHTS):

.. code-block:: python

    def _get_model_weights(self) -> Shareable:
        # Get state dict and send as weights
        new_weights = self.model.state_dict()
        new_weights = {k: v.cpu().numpy() for k, v in new_weights.items()}

        outgoing_dxo = DXO(
            data_kind=DataKind.WEIGHTS, data=new_weights, meta={MetaKey.NUM_STEPS_CURRENT_ROUND: self._n_iterations}
        )
        return outgoing_dxo.to_shareable()


.. code-block:: python

    def execute(self, task_name: str, shareable: Shareable, fl_ctx: FLContext, abort_signal: Signal) -> Shareable:
        try:
            if task_name == self._pre_train_task_name:
                # return model weights
                return self._get_model_weights()
            elif task_name == self._train_task_name:
                ...

完全な実装は、``hello-pt`` の custom フォルダーにある ``cifar10trainer.py`` にあります。

InitializeGlobalWeights の詳細
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
クライアントの応答(各クライアントのモデル重み)を処理する際、``GlobalWeightsInitializer`` はそのうちの1つをグローバル重みとして選択します。
2つの重み選択方法をサポートしています("weight_method" 引数で指定します):

  - **first**\ : 最初に応答したクライアントが報告した重みを使用します。これがデフォルトの方法です。
  - **client**\ : 指定したクライアントが報告した重みを使用します。決定論的なトレーニングに役立つ場合があります。

厳密には、すべてのクライアントから報告された重みを検証・比較し、有効かつ互換性があることを確認すべきです。現時点では、
その正確な方法が明確でないため、``GlobalWeightsInitializer`` はこの処理を行っていません。

InitializeGlobalWeights の実装に関する注記
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``InitializeGlobalWeights`` コントローラーは、汎用の ``BroadcastAndProcess`` コントローラーを拡張して実装されています。

``BroadcastAndProcess`` コントローラーは、クライアントの応答を処理する ``ResponseProcessor`` コンポーネントを必要とします。``BroadcastAndProcess`` コントローラーは次のように動作します:

  - 設定された名前のタスクを、すべてのクライアントまたは設定されたクライアントのリストにブロードキャストし、データを要求します。
  - クライアントから応答を受信するたびに、``BroadcastAndProcess`` は ``ResponseProcessor`` コンポーネントを呼び出して応答を処理します。
  - タスクが完了する(すべてのクライアントから応答を受信するか、タイムアウトする)と、``BroadcastAndProcess`` は ``ResponseProcessor`` コンポーネントを呼び出して
    最終チェックを行います。

``InitializeGlobalWeights`` コントローラーは、``GlobalWeightsInitializer`` を ``ResponseProcessor`` コンポーネントとして
``BroadcastAndProcess`` を単純に拡張したものです。
