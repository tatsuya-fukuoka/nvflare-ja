##########################################
ワークフローと Controller
##########################################

ワークフローは 1 つ以上の Controller を持ち、それぞれが特定の協調戦略を実装します。例えば、ScatterAndGather
(SAG) Controller は、fed-average 型のフェデレーテッド学習で一般的に使用される代表的な戦略を実装しています。
CrossSiteValidation Controller は、すべてのクライアントサイトが他のすべてのサイトのモデルを評価できるようにする
戦略を実装しています。任意の数の Controller を組み合わせてワークフローを構成できます。

FLModel ベースの :ref:`model_controller` を提供しており、これはユーザーが Controller を書くための分かりやすい方法を提供します。
また、より FLARE 固有の機能を備えた従来の :ref:`Controller API <controllers>` もあり、既存のワークフローの多くはこれを基盤としています。

サーバー側の Controller を用いて、サーバー制御によるフェデレーテッドラーニングのワークフロー (fed-average、cyclic controller、cross-site evaluation) をいくつか実装しています。
これらのワークフローでは、FL クライアントは Controller からタスクを割り当てられ、そのタスクを実行し、結果をサーバーに提出します。

場合によっては、サーバーが信頼できないときには、機微な情報を伴う通信にサーバーを関与させるべきではありません。
この懸念に対処するため、NVFlare はクライアント間のピアツーピア通信を実現する Client Controlled Workflows (CCWF) を導入しています。


Controller は ``config_fed_server.json`` の workflows セクションで設定できます。

.. code-block::

  workflows = [
      {
          id = "fedavg_ctl",
          path = "nvflare.app_common.workflows.fedavg.FedAvg",
          args {
              min_clients = 2,
              num_rounds = 3,
              persistor_id = "persistor"
          }
      }
  ]

JobAPI を使って Controller を設定するには、Controller を定義してサーバーに送信します。
このコードにより、Controller 用のサーバー設定が自動的に生成されます。

.. code-block:: python

  controller = FedAvg(
      num_clients=2,
      num_rounds=3,
      persistor_id = "persistor"
  )
  job.to(controller, "server")

さまざまな種類の Controller に関する詳細は、以下のセクションを参照してください。

.. toctree::
   :maxdepth: 3

   controllers/model_controller
   controllers/controllers
   controllers/client_controlled_workflows
