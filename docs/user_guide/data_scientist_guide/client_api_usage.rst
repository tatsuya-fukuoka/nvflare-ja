.. _client_api_usage:

########################
Client API の使い方
########################

FLARE Client API は、集中型のローカルトレーニングコードを連合学習(Federated Learning)コードへ
簡単に変換する手段を提供し、以下の利点があります:

* 数行のコード変更のみで済み、コードの再構成や新しいクラスの実装が不要
* ユーザーに公開される FLARE 固有の新しい概念の数を削減
* 異なるフレームワーク(PyTorch、PyTorch Lightning、HuggingFace)を使用した
  既存のローカルトレーニングコードから容易に適応可能

コアコンセプト
========================

代表的な連合学習(FL)ワークフローである「FedAvg」の一般的な構造は次のとおりです:

#. FL サーバーが初期モデルを初期化する
#. 各ラウンド(グローバルイテレーション)で:

   #. FL サーバーがグローバルモデルをクライアントに送信する
   #. 各 FL クライアントはこのグローバルモデルを起点として自身のデータでトレーニングする
   #. 各 FL クライアントはトレーニング済みモデルを送り返す
   #. FL サーバーがすべてのモデルを集約し、新しいグローバルモデルを生成する

クライアント側では、トレーニングワークフローは次のようになります:

#. FL サーバーからモデルを受信する
#. 受信したグローバルモデルに対してローカルトレーニングを実行する、および/または
   モデル選択のために受信したグローバルモデルを評価する
#. 新しいモデルを FL サーバーに送り返す

集中型トレーニングコードを連合学習に変換するには、コードを以下のステップを
実行するように適応させる必要があります:

#. 受信した :ref:`fl_model` から必要な情報を取得する
#. ローカルトレーニングを実行する
#. 結果を新しい :ref:`fl_model` に格納して送り返す

一般的なユースケースでは、Client API には 3 つの基本メソッドがあります:

* ``init()``: NVFlare Client API 環境を初期化します。
* ``receive()``: NVFlare 側からモデルを受信します。
* ``send()``: NVFlare 側にモデルを送信します。

ユーザーは Client API を使って、集中型トレーニングコードを連合学習に
変更できます。例:

.. code-block:: python

    import nvflare.client as flare

    flare.init() # 1. Initializes NVFlare Client API environment.
    input_model = flare.receive() # 2. Receives model from NVFlare side.
    params = input_model.params # 3. Obtain the required information from received FLModel

    # original local training code begins
    new_params = local_train(params)
    # original local training code ends

    output_model = flare.FLModel(params=new_params) # 4. Put the results in a new FLModel
    flare.send(output_model) # 5. Sends the model to NVFlare side.

わずか 5 行のコード変更で、集中型トレーニングコードを連合学習の設定に
変換できます。

FL ジョブの定義と実行
==========================================

Client API でトレーニングスクリプトを修正した後、ジョブの実行方法を定義する ``job.py`` ファイルを作成する必要があります:

.. code-block:: python

    # job.py - Define and run the FL job
    from nvflare.app_common.np.recipes.fedavg import NumpyFedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = NumpyFedAvgRecipe(
        name="my-fl-job",
        min_clients=2,
        num_rounds=3,
        # Model can be class instance, array, or dict config
        # For pre-trained weights: initial_ckpt="/server/path/to/model.npy"
        model=[[1, 2, 3], [4, 5, 6]],
        train_script="client.py",  # Points to your Client API script
    )

    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

その後、連合学習ジョブを実行します:

.. code-block:: bash

    python job.py

これだけです! トレーニングスクリプト(Client API 使用)とジョブ定義が連携して連合学習を実行します。

詳細なガイドとリソースについては、末尾の **さらに学ぶ** セクションを参照してください。

Client API リファレンス
==============================

以下は主要な Client API の概要一覧です。

.. list-table:: Client API
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントリンク
   * - init
     - NVFlare Client API 環境を初期化します。
     - :func:`init<nvflare.client.api.init>`
   * - receive
     - NVFlare 側からモデルを受信します。
     - :func:`receive<nvflare.client.api.receive>`
   * - send
     - NVFlare 側にモデルを送信します。
     - :func:`send<nvflare.client.api.send>`
   * - system_info
     - NVFlare のシステム情報を取得します。
     - :func:`system_info<nvflare.client.api.system_info>`
   * - get_job_id
     - ジョブ ID を取得します。
     - :func:`get_job_id<nvflare.client.api.get_job_id>`
   * - get_site_name
     - サイト名を取得します。
     - :func:`get_site_name<nvflare.client.api.get_site_name>`
   * - is_running
     - NVFlare システムが稼働中かどうかを返します。
     - :func:`is_running<nvflare.client.api.is_running>`
   * - is_train
     - 現在のタスクがトレーニングタスクかどうかを返します。
     - :func:`is_train<nvflare.client.api.is_train>`
   * - is_evaluate
     - 現在のタスクが評価タスクかどうかを返します。
     - :func:`is_evaluate<nvflare.client.api.is_evaluate>`
   * - is_submit_model
     - 現在のタスクが submit_model タスクかどうかを返します。
     - :func:`is_submit_model<nvflare.client.api.is_submit_model>`

.. list-table:: Lightning API
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントリンク
   * - patch
     - PyTorch Lightning の Trainer を FLARE で使用できるようにパッチします。
     - :func:`patch<nvflare.app_opt.lightning.api.patch>`

.. list-table:: HuggingFace API
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントリンク
   * - patch
     - HuggingFace の ``Trainer`` または TRL の ``SFTTrainer`` を FLARE で使用できるようにパッチします。
     - :func:`patch<nvflare.app_opt.hf.api.patch>`
   * - is_running
     - パッチされた HuggingFace トレーナーの FL ループを調整します。
     - :func:`is_running<nvflare.client.hf.is_running>`

.. list-table:: メトリクスロガー
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントリンク
   * - SummaryWriter
     - SummaryWriter は Tensorboard の SummaryWriter の使い方を模倣します。
     - :class:`SummaryWriter<nvflare.client.tracking.SummaryWriter>`
   * - WandBWriter
     - WandBWriter は Weights & Biases の使い方を模倣します。
     - :class:`WandBWriter<nvflare.client.tracking.WandBWriter>`
   * - MLflowWriter
     - MLflowWriter は MLflow の使い方を模倣します。
     - :class:`MLflowWriter<nvflare.client.tracking.MLflowWriter>`


動作するサンプル
==========================

さまざまなフレームワークでの完全な動作サンプル:

* PyTorch: :github_nvflare_link:`hello-pt <examples/hello-world/hello-pt>`
* NumPy: :github_nvflare_link:`hello-numpy <examples/hello-world/hello-numpy>`
* PyTorch Lightning: :github_nvflare_link:`hello-lightning <examples/hello-world/hello-lightning>`
* TensorFlow: :github_nvflare_link:`hello-tf <examples/hello-world/hello-tf>`
* HuggingFace Trainer: :github_nvflare_link:`hello-huggingface <examples/hello-world/hello-huggingface>`

各サンプルでは、Client API トレーニングスクリプト(``client.py``)とジョブレシピ定義(``job.py``)の両方を示しています。

さらに学ぶ
====================

* :ref:`job_recipe` - トレーニングスクリプトで FL ジョブを定義・実行する方法
* :ref:`client_api` - 詳細な例と技術的詳細を含むプログラミングガイド
* :ref:`hf_client_api` - HuggingFace Trainer 統合ガイド
* :mod:`nvflare.client.api` - 完全な API リファレンスドキュメント
* :mod:`nvflare.app_opt.lightning.api` - PyTorch Lightning 統合
