.. _model_controller:

###################
ModelController API
###################

FLARE の :mod:`ModelController<nvflare.app_common.workflows.model_controller>` API は、FLModel ベースのコントローラーワークフローを簡単に作成・カスタマイズできる手段を提供します。

* シンプルな API(run ルーチンと基本的な通信・ユーティリティ関数)による高い柔軟性
* 通信データ構造には :ref:`fl_model` を使用し、それ以外はすべて純粋な Python
* 既存コンポーネントや FLARE 固有の機能をサポートするオプション

.. note::

    注記: ModelController API は、ワークフローの記述を簡単にすることを目的とした高レベル API です。
    FLARE のすべての機能を備えた Controller の完全な柔軟性を好む、または必要とする場合は、:ref:`controllers` を参照してください。


コアコンセプト
==============

例として、よく知られた連合学習ワークフローである "FedAvg" を見てみましょう。これは次の手順で構成されます:

#. FL サーバーが初期モデルを初期化する
#. 各ラウンド(グローバルイテレーション)ごとに:

   #. FL サーバーがグローバルモデルをクライアントに送信する
   #. 各 FL クライアントはこのグローバルモデルから開始し、自身のデータでトレーニングする
   #. 各 FL クライアントはトレーニング済みモデルを送り返す
   #. FL サーバーがすべてのモデルを集約し、新しいグローバルモデルを生成する


ModelController を使ってこのワークフローを実装するには、いくつかの重要な要素があります:

* :class:`nvflare.app_common.workflows.model_controller.ModelController` をインポートしてサブクラス化する。
* ワークフローのロジックとして ``run()`` ルーチンを実装する。
* 通信には ``send_model()`` / ``send_model_and_wait()`` を利用し、FLModel を伴うタスクを対象クライアントに送信し、FLModel の結果を受信する。
* 事前定義されたユーティリティ関数やコンポーネントを使ってワークフローをカスタマイズするか、独自のロジックを実装する。


以下は、:class:`BaseFedAvg<nvflare.app_common.workflows.base_fedavg.BaseFedAvg>` 基底クラスを使った FedAvg ワークフローの例です:

.. code-block:: python

    # BaseFedAvg subclasses ModelController and defines common functions and variables such as aggregate(), update_model(), self.start_round, self.num_rounds
    class FedAvg(BaseFedAvg):

      # run routine that user must implement
      def run(self) -> None:
          self.info("Start FedAvg.")

          # load model (by default uses persistor, can provide custom method)
          model = self.load_model()
          model.start_round = self.start_round
          model.total_rounds = self.num_rounds

          # for each round (global iteration)
          for self.current_round in range(self.start_round, self.start_round + self.num_rounds):
              self.info(f"Round {self.current_round} started.")
              model.current_round = self.current_round

              # obtain self.num_clients clients
              clients = self.sample_clients(self.num_clients)

              # send model to target clients with default train task, wait to receive results
              results = self.send_model_and_wait(targets=clients, data=model)

              # use BaseFedAvg aggregate function
              aggregate_results = self.aggregate(
                  results, aggregate_fn=self.aggregate_fn
              )  # using default aggregate_fn with `WeightedAggregationHelper`. Can overwrite self.aggregate_fn with signature Callable[List[FLModel], FLModel]

              # update global model with aggregation results
              model = self.update_model(model, aggregate_results)

              # save model (by default uses persistor, can provide custom method)
              self.save_model(model)

          self.info("Finished FedAvg.")


以下は、:class:`ModelController<nvflare.app_common.workflows.model_controller.ModelController>` API の包括的な概要テーブルです:


.. list-table:: ModelController API
   :widths: 25 35 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントへのリンク
   * - run
     - ワークフローの run ルーチン。
     - :func:`run<nvflare.app_common.workflows.model_controller.ModelController.run>`
   * - send_model_and_wait
     - データを伴うタスクをターゲットに送信し(ブロッキング)、結果を待ちます。
     - :func:`send_model_and_wait<nvflare.app_common.workflows.model_controller.ModelController.send_model_and_wait>`
   * - send_model
     - データを伴うタスクをターゲットに送信し(ノンブロッキング)、コールバックを使用します。
     - :func:`send_model<nvflare.app_common.workflows.model_controller.ModelController.send_model>`
   * - sample_clients
     - num_clients 個のクライアントのリストを返します。
     - :func:`sample_clients<nvflare.app_common.workflows.model_controller.ModelController.sample_clients>`
   * - save_model
     - persistor を使ってモデルを保存します。
     - :func:`save_model<nvflare.app_common.workflows.model_controller.ModelController.save_model>`
   * - load_model
     - persistor からモデルを読み込みます。
     - :func:`load_model<nvflare.app_common.workflows.model_controller.ModelController.load_model>`


通信
====

ModelController はタスクベースの通信を用います。タスクがターゲットに送信され、ターゲットがタスクを実行して結果を返します。
:ref:`fl_model` は各タスクとともに送信される標準化されたデータ構造オブジェクトであり、結果として :ref:`fl_model` の応答が受信されます。

.. note::

    注記: :ref:`fl_model` オブジェクトは、具体的なタスクに応じて任意の種類のデータになり得ます。
    たとえば "train" や "validate" タスクでは、対象クライアントがモデルのトレーニングや検証を行えるように、タスクとともにモデルパラメーターを送信します。
    一方、モデルの送信を伴わない他の多くのタスク(例: "submit_model")では、:ref:`fl_model` は任意の種類のデータ(例: メタデータ、メトリクスなど)を含むことができ、まったく不要な場合もあります。


send_model_and_wait
-------------------
:func:`send_model_and_wait<nvflare.app_common.workflows.model_controller.ModelController.send_model_and_wait>` は、ターゲットにタスクを送信し、応答を待つことを可能にする中核的な通信関数です。

``data`` は :ref:`fl_model` オブジェクトであり、``task_name`` は対象のエグゼキューターが実行するタスクです(Client API のエグゼキューターはデフォルトで "train"、"validate"、"submit_model" をサポートしますが、エグゼキューターは任意のタスク名に対して作成できます)。

``targets`` は、``sample_clients()`` で取得したクライアント名から選択できます。

タスクが完了する(``min_responses`` 件の応答を受信するか、``timeout`` の時間が経過する)と、対象クライアントからの :ref:`fl_model` 応答を返します。

send_model
----------
:func:`send_model<nvflare.app_common.workflows.model_controller.ModelController.send_model>` は、
:func:`send_model_and_wait<nvflare.app_common.workflows.model_controller.ModelController.send_model_and_wait>` のノンブロッキング版であり、応答受信時にユーザー定義のコールバックが呼ばれます。

シグネチャ ``Callable[[FLModel], None]`` のコールバックを渡すことができ、各ターゲットから応答を受信したときに呼び出されます。

タスクは、``min_responses`` 件の応答を受信するか、``timeout`` の時間が経過するまで存続(standing)します。
この呼び出しは非同期であるため、同期の目的で Controller の :func:`get_num_standing_tasks<nvflare.apis.impl.controller.Controller.get_num_standing_tasks>` メソッドを使って存続中のタスク数を取得できます。

たとえば :github_nvflare_link:`CrossSiteEval <nvflare/app_common/workflows/cross_site_eval.py>` ワークフローでは、各クライアントのモデルを取得するために :func:`send_model<nvflare.app_common.workflows.model_controller.ModelController.send_model>` でタスクが非同期に送信されます。
その後、コールバックを通じて、クライアントのモデルが検証のために他のクライアントへ送信されます。
最後に、ワークフローは :func:`get_num_standing_tasks<nvflare.apis.impl.controller.Controller.get_num_standing_tasks>` を使って、存続中のすべてのタスクの完了を待ちます。
以下はこれらの関数の使用例です。詳細は :github_nvflare_link:`CrossSiteEval <nvflare/app_common/workflows/cross_site_eval.py>` の実装を参照してください。


.. code-block:: python

    class CrossSiteEval(ModelController):
        ...
        def run(self) -> None:
            ...
            # Create submit_model task and broadcast to all participating clients
            self.send_model(
                task_name=AppConstants.TASK_SUBMIT_MODEL,
                data=data,
                targets=self._participating_clients,
                timeout=self._submit_model_timeout,
                callback=self._receive_local_model_cb,
            )
            ...
            # Wait for all standing tasks to complete, since we used non-blocking `send_model()`
            while self.get_num_standing_tasks():
                if self.abort_signal.triggered:
                    self.info("Abort signal triggered. Finishing cross site validation.")
                    return
                self.debug("Checking standing tasks to see if cross site validation finished.")
                time.sleep(self._task_check_period)

            self.save_results()
            self.info("Stop Cross-Site Evaluation.")

        def _receive_local_model_cb(self, model: FLModel):
            # Send this model to all clients to validate
            model.meta[AppConstants.MODEL_OWNER] = model_name
            self.send_model(
                task_name=AppConstants.TASK_VALIDATION,
                data=model,
                targets=self._participating_clients,
                timeout=self._validation_timeout,
                callback=self._receive_val_result_cb,
            )
        ...


保存と読み込み
==============

persistor
---------
:func:`save_model<nvflare.app_common.workflows.model_controller.ModelController.save_model>` と :func:`load_model<nvflare.app_common.workflows.model_controller.ModelController.load_model>`
の各関数は、ModelController の初期化引数 ``persistor_id: str = "persistor"`` で設定された :class:`ModelPersistor<nvflare.app_common.abstract.model_persistor.ModelPersistor>` を利用します。

カスタムの保存・読み込み
------------------------
persistor を使う代わりに、独自のカスタム保存・読み込み関数を作成することもできます。

たとえば、モデルパラメーターには PyTorch の保存・読み込み関数を使い、FLModel のメタデータは :mod:`FOBS<nvflare.fuel.utils.fobs>` を使って別のファイルパスに個別に保存できます。

.. code-block:: python

    import torch
    from nvflare.fuel.utils import fobs

    class MyController(ModelController):
        ...
        def save_model(self, model, filepath=""):
            params = model.params
            # PyTorch save
            torch.save(params, filepath)

            # save FLModel metadata
            model.params = {}
            fobs.dumpf(model, filepath + ".metadata")
            model.params = params

        def load_model(self, filepath=""):
            # PyTorch load
            params = torch.load(filepath)

            # load FLModel metadata
            model = fobs.loadf(filepath + ".metadata")
            model.params = params
            return model


注記: ``torch.nn.Module``\ (初期 PyTorch モデルに使用)のような非プリミティブなデータ型については、
シリアライズおよびデシリアライズのために対応する FOBS デコンポーザーを設定する必要があります。
詳細は :ref:`serialization` を参照してください。

.. code-block:: python

  from nvflare.app_opt.pt.decomposers import TensorDecomposer

  fobs.register(TensorDecomposer)


追加機能
========

場合によっては、より高度な FLARE 固有の機能が役立つことがあります。

:mod:`BaseModelController<nvflare.app_common.workflows.base_model_controller>` クラスは、必要に応じてエンジン ``self.engine`` と FLContext ``self.fl_ctx`` へのアクセスを提供します。
``get_component()`` や ``build_component()`` などの関数を使って、コンポーネントを読み込んだり動的に構築したりできます。

さらに、基盤となる :mod:`Controller<nvflare.apis.impl.controller>` クラスは、追加の通信関数やタスク関連のユーティリティを提供します。
既存のワークフローの多くは、この低レベルの Controller API に基づいています。
詳細は :ref:`controllers` セクションを参照してください。

例
==

ModelController API を使った基本的なワークフローの例:

* :github_nvflare_link:`Cyclic <nvflare/app_common/workflows/cyclic.py>`
* :github_nvflare_link:`BaseFedAvg <nvflare/app_common/workflows/base_fedavg.py>`
* :github_nvflare_link:`FedAvg <nvflare/app_common/workflows/fedavg.py>`

高度な例:

* :github_nvflare_link:`Scaffold <nvflare/app_common/workflows/scaffold.py>`
* :github_nvflare_link:`FedOpt <nvflare/app_opt/pt/fedopt_ctl.py>`
* :github_nvflare_link:`Kaplan-Meier <examples/advanced/kaplan-meier-he/server_he.py>`
* :github_nvflare_link:`FedBPT <research/fed-bpt/src/global_es.py>`
