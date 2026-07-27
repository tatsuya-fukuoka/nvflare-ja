.. _model_learner:

#############
Model Learner
#############

.. warning::

   Model Learner のパターンは非推奨です。後方互換性のために引き続き利用可能ですが、新規の
   プロジェクトでは :ref:`job_recipe` と :ref:`client_api` を使用してください。


はじめに
========

:github_nvflare_link:`ModelLearner <nvflare/app_common/abstract/model_learner.py>` の目的は、ユーザーに公開される FLARE 固有の概念を最小限に抑えることで、学習ロジックを書きやすくすることです。

ModelLearner の中心的な概念は :github_nvflare_link:`FLModel <nvflare/app_common/abstract/fl_model.py>` であり、これは馴染みのある学習用語でフェデレーテッドラーニングの機能をサポートする構造を定義します。
具体的な Model Learner を作成する際、研究者は FLModel オブジェクトのみを用いて学習および検証のメソッドを実装します。
研究者は Shareable や FLContext といった FLARE 固有の概念を扱う必要がなくなりますが、FLModel だけでは不十分な高度なケースのためにそれらも引き続き利用できます。

Model Learner の作成方法
========================

具体的な Model Learner を作成するには、ModelLearner クラスを継承します。以下は NPLearner の例です。

.. code-block:: python

   from nvflare.app_common.abstract.model_learner import ModelLearner
   from nvflare.app_common.abstract.fl_model import FLModel, ParamsType


   class NPLearner(ModelLearner):

以下のメソッドを実装する必要があります。

.. code-block:: python

      def initialize(self):
      def train(self, model: FLModel) -> Union[str, FLModel]:
      def get_model(self, model_name: str) -> Union[str, FLModel]:
      def validate(self, model: FLModel) -> Union[str, FLModel]:
      def configure(self, model: FLModel)
      def abort(self)
      def finalize(self)

これらのメソッドの説明については、:class:`ModelLearner<nvflare.app_common.abstract.model_learner.ModelLearner>` の docstring を参照してください。

初期化と終了処理
----------------

ModelLearner に初期化が必要な場合は、``initialize`` メソッドに初期化ロジックを記述してください。このメソッドは学習ジョブの開始前に一度だけ呼び出されます。
ModelLearner の基底クラスは、初期化ロジックで利用できる多くの便利なメソッドを提供しています。

同様に、ModelLearner を適切に終了させる必要がある場合もあります。
その場合は、そのロジックを ``finalize`` メソッドに記述してください。このメソッドは学習ジョブの終了時に一度だけ呼び出されます。

学習ロジック
------------

学習ロジックは ``train`` メソッドと ``validate`` メソッドに実装します。すべての学習情報は FLModel オブジェクトに含まれています。
同様に、学習メソッドの結果は、処理が成功した場合は FLModel オブジェクト、何らかの理由で処理が失敗した場合は ReturnCode を表す str のいずれかになります。

FLModel オブジェクトの params_type を確認し、期待するパラメータが含まれていることを確かめてください。

可能であれば、学習ロジックの中で、特に長時間実行されるステップの前後で、ModelLearner に中断が要求されていないかを定期的に確認すべきです。
これは ``self.is_aborted()`` メソッドを呼び出すことで確認できます。典型的な使用パターンは次のとおりです。

.. code-block:: python

	if self.is_aborted():
   		return ReturnCode.TASK_ABORTED


学習ロジックを継続できない状況に陥った場合は、学習メソッドから適切な ReturnCode を返すだけで構いません。

要求されたモデルの返却
----------------------

ModelLearner は、指定された種類のモデル (例: ベストモデル) を返すよう要求されることがあります。
例えば、学習が完了したとき、サーバーはローカルのベストモデルを返すよう要求し、それを他のサイトへ送って検証させることがあります。
これをサポートするには、``get_model`` メソッドを実装し、要求されたモデルを返す必要があります。

動的な設定
----------

ローカルに設定された情報に基づく静的な設定ではなく、サーバーから送られてくる情報に基づいて ModelLearner を動的に設定したい場合は、``configure`` メソッドを実装することで実現できます。
FLModel オブジェクトには、モデル学習機能のための設定パラメータを指定します。

グレースフルな中断
------------------

ModelLearner は、学習メソッドの実行中に中断を要求されることがあります (例: ユーザーが ``abort_job`` コマンドを発行する、サーバーの Controller がタスクの中断を決定するなど)。
学習メソッドが使用しているフレームワーク (MONAI、Ignite、TensorFlow など) によっては、学習フレームワークをグレースフルに中断させるために何らかの処理が必要になる場合があります。
その場合は、そのロジックを ``abort`` メソッドに記述します。

``abort`` メソッドは任意です。学習フレームワークが中断できない、または中断する必要がない場合は、このメソッドを実装する必要はありません。

ロギングメソッド
----------------

ModelLearner の基底クラスは、ロギングのための便利なメソッドを提供しています。

.. code-block:: python

   def debug(self, msg: str)
   def info(self, msg: str)
   def error(self, msg: str)
   def warning(self, msg: str)
   def exception(self, msg: str)
   def critical(self, msg: str)

これらのメソッドを使って、学習ロジックの中でさまざまなログレベルのログメッセージを作成できます。

追加コンポーネントの取得
------------------------

FLARE ランタイムは、Learner の実装で利用できる多くのサービスコンポーネント (統計ロギング、セキュリティ、設定サービスなど) を提供しています。
これらのオブジェクトは、ModelLearner クラスが提供する次のメソッドで取得できます。

.. code-block:: python

   def get_component(self, component_id: str) -> Any

通常、これは Learner の初期化時に呼び出すべきです。

以下は、CIFAR10ModelLearner で AnalyticsSender クライアントコンポーネントを利用する例です。

.. code-block:: python

   self.writer = self.get_component(
      self.analytic_sender_id
   )

コンテキスト情報の取得
----------------------

FLModel オブジェクトには学習タスクに関する重要な情報が含まれています。さらに、次のようなコンテキスト情報が必要になる場合があります。

- site_name: 学習サイトの名前
- engine: 追加の情報とサービスを提供する FLARE エンジン
- workspace: データの読み書きに使用できるワークスペース
- job_id: ジョブの ID
- app_root: ワークスペース内の現在のジョブのルートディレクトリ
- shareable: タスクに付随する Shareable オブジェクト
- fl_ctx: タスクに付随する FLContext オブジェクト

これらは Learner オブジェクト (self) から直接利用できます。

ModelLearner の基底クラスは、Shareable および FLContext オブジェクトのプロパティを取得するための便利なメソッドも提供しています。

.. code-block:: python

   def get_shareable_header(self, key: str, default=None)
   def get_context_prop(self, key: str, default=None)

Model Learner のインストール方法
================================

Model Learner を開発したら、それを学習クライアントにインストールする必要があります。
Model Learner は FLARE が提供する ModelLearnerExecutor と組み合わせて動作させる必要があります。
以下の例は、ジョブの ``config_fed_client.json`` で Model Learner をどのように設定するかを示しています。

.. code-block:: json

   {
      "format_version": 2,
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.executors.model_learner_executor.ModelLearnerExecutor",
               "args": {
                  "learner_id": "np_learner"
               }
            }
         }
      ],
      "task_result_filters": [
      ],
      "task_data_filters": [
      ],
      "components": [
         {
            "id": "np_learner",
            "path": "np_learner.NPLearner",
            "args": {
            }
         }
      ]
   }

次の点に注意してください。

- ``executor`` の ``path`` は ``nvflare.app_common.executors.model_learner_executor.ModelLearnerExecutor`` でなければなりません。
- ``executor`` の ``learner_id`` と ``components`` の ``id`` は一致していなければなりません (この例では ``np_learner``)。
- ``np_learner`` コンポーネントの path は、あなたの Model Learner の実装を指している必要があります。

その他のリソース
================

:github_nvflare_link:`ModelLearner <nvflare/app_common/abstract/model_learner.py>` と :github_nvflare_link:`FLModel <nvflare/app_common/abstract/fl_model.py>` の API に加えて、ModelLearner を使用した以下の例も参照してください。

- :github_nvflare_link:`CIFAR10 ModelLearner <examples/advanced/cifar10/pt/learners/cifar10_model_learner.py>`
