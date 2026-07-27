.. _available_recipes:

########################
利用可能なレシピ
########################

NVFlareは、一般的な連合学習(Federated Learning)アルゴリズムとワークフローのための、さまざまな事前構築済みレシピを提供しています。
レシピは、ジョブの設定と実行を簡素化する高レベルの宣言的APIです。

.. contents:: 目次
   :local:
   :depth: 2

はじめる前に
========================

このページは、利用可能なレシピクラスのカタログと、始めるための短いスニペット集です。
モデルの入力形式、チェックポイントの挙動、実行環境については
:ref:`job_recipe` を参照してください。共通のRecipeメソッド、ヘルパー、安定API仕様については
:ref:`recipe_api` を参照してください。

.. important::

   レシピの引数とヘルパーの設定は、生成されるジョブ定義の一部になります。
   実際のパスワード、トークン、APIキー、秘密鍵、その他の認証情報を、
   ネストされた辞書を含むいかなるレシピパラメータにも決して入れないでください。
   シークレットはサイトの環境変数またはマウントされたシークレットファイルに保持してください。
   ``secret_ref`` や ``secret_file_ref`` はサポートされているランタイム境界でのみ使用し、
   それ以外の場合はトレーニングコード内でシークレットを直接読み込んでください。
   サポートされている場所については :ref:`recipe_secrets` を参照してください。

Fed Task
==============

グローバルモデルのライフサイクルを持たない1ラウンドのワークフローには、``FedTaskRecipe`` を使用します。
これは、クライアント側での埋め込み抽出、前処理、特徴量生成、ローカル評価など、
サーバーがクライアント間で1回のスクリプト実行を調整するだけでよい連合タスクに便利です。

.. code-block:: python

    from nvflare.recipe import FedTaskRecipe, SimEnv

    recipe = FedTaskRecipe(
        name="extract-embeddings",
        task_name="embed",
        min_clients=2,
        task_script="client.py",
        task_args="--data-root /data --output-root /tmp/embeddings",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

``FedTaskRecipe`` は、選択されたクライアントに1つの ``FLModel`` タスクを送信し、それらの結果を待機します。
``model``、``initial_ckpt``、``model_persistor`` は必要ありません。

連合平均(FedAvg)
============================

複数のクライアントからのモデル更新を重み付き平均によって集約する、最も基本的な連合学習アルゴリズムです。

PyTorch FedAvg
--------------

.. code-block:: python

    from nvflare.app_opt.pt.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = FedAvgRecipe(
        name="fedavg-pt",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

大規模なPyTorchモデル更新の場合、``FedAvgRecipe`` は ``enable_tensor_disk_offload=True`` もサポートしており、
受信ストリーミングテンソルを一時ファイルに実体化することでサーバーのメモリ使用量を削減します。
サーバーの一時ディレクトリの設定に関するデプロイメント上の注意点については、
:ref:`連合学習サーバーの起動 <starting_fl_servers>` を参照してください。

**例:**

- `examples/hello-world/hello-pt <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-pt>`_
- `examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedavg <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedavg>`_

TensorFlow FedAvg
-----------------

.. code-block:: python

    from nvflare.app_opt.tf.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = FedAvgRecipe(
        name="fedavg-tf",
        min_clients=2,
        num_rounds=5,
        model=MyTFModel(),
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-tf <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-tf>`_
- `examples/advanced/cifar10/tf/cifar10_fedavg <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/tf/cifar10_fedavg>`_

NumPy FedAvg
------------

フレームワーク非依存またはNumPyベースのモデル向けです。

.. code-block:: python

    from nvflare.app_common.np.recipes import NumpyFedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = NumpyFedAvgRecipe(
        name="fedavg-numpy",
        min_clients=2,
        num_rounds=5,
        model=[0.0, 0.0, 0.0],
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-numpy <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-numpy>`_

Sklearn FedAvg
--------------

scikit-learnベースのモデル向けです。

.. code-block:: python

    from nvflare.app_opt.sklearn.recipes import SklearnFedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = SklearnFedAvgRecipe(
        name="fedavg-sklearn",
        min_clients=2,
        num_rounds=5,
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/sklearn-linear <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/sklearn-linear>`_

準同型暗号を用いたFedAvg
------------------------------------------------

準同型暗号によるセキュアな集約を用いたFedAvgです。

.. code-block:: python

    from nvflare.app_opt.pt.recipes import FedAvgRecipeWithHE
    from nvflare.recipe import ProdEnv

    recipe = FedAvgRecipeWithHE(
        name="fedavg-he",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
    )
    env = ProdEnv(
        startup_kit_location="/path/to/startup_kit/admin@nvidia.com",
        username="admin@nvidia.com",
    )
    run = recipe.execute(env)

.. note::
   ``FedAvgRecipeWithHE`` には、準同型暗号コンテキストファイルを含むプロビジョニング済みスタートアップキットが必要です。
   HEプロビジョニングを行った ``ProdEnv`` または ``PocEnv`` を使用してください。``SimEnv`` はサポートされていません。

**例:**

- `examples/advanced/cifar10/pt/cifar10-real-world#secure-aggregation-using-homomorphic-encryption <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-real-world#42-secure-aggregation-using-homomorphic-encryption>`_

WEIGHT_DIFF の互換性
------------------------------------------

``DataKind.WEIGHT_DIFF`` は、クライアントのエグゼキューターがパラメータ差分を送信し、
かつサーバーの集約パスが差分を受け付ける場合にのみサポートされます。クライアントの結果を正式に記述するのは、
そのクライアントの ``FLModel.params_type`` です。レシピの構築時には任意のトレーニングスクリプトの中身を
検査できないため、サポートされるデータ種別や、カスタムアグリゲーターが宣言する ``expected_data_kind`` といった、
レシピが管理するサーバー側設定のみが検証されます。

.. list-table:: レシピとアグリゲーターのサポート状況
   :header-rows: 1
   :widths: 28 30 12 30

   * - レシピ
     - サーバー集約パス
     - サポート
     - 必要な設定
   * - 統合版、PyTorch、TensorFlow、NumPyの各 ``FedAvgRecipe``、PyTorch FedProx
     - 組み込みの ``FedAvg`` ストリーミング集約
     - 対応
     - ``aggregator_data_kind=DataKind.WEIGHT_DIFF`` を設定し、クライアントから
       ``FLModel(params_type=ParamsType.DIFF)`` を返します。
   * - カスタム ``ModelAggregator`` を用いた ``FedAvgRecipe``
     - ユーザー提供のアグリゲーター
     - 条件付き
     - クライアントは ``ParamsType.DIFF`` を返します。カスタムアグリゲーターは差分モデルを
       受け付け、その集約結果において ``ParamsType.DIFF`` を維持する必要があります。
       ``expected_data_kind`` を宣言する場合は、``DataKind.WEIGHT_DIFF`` を宣言する必要があります。
   * - ``FedAvgRecipeWithHE``
     - ``HEInTimeAccumulateWeightedAggregator``
     - 対応
     - ``aggregator_data_kind=DataKind.WEIGHT_DIFF`` を設定し、クライアントから
       ``FLModel(params_type=ParamsType.DIFF)`` を返します。
   * - PyTorch ``FedOptRecipe``
     - ``InTimeAccumulateWeightedAggregator``
     - 対応
     - このレシピは集約パスを ``WEIGHT_DIFF`` に固定します。カスタムアグリゲーターが
       ``expected_data_kind`` を宣言する場合は、``DataKind.WEIGHT_DIFF`` を宣言する必要があります。
   * - TensorFlow ``FedOptRecipe``
     - 組み込みの ``FedAvg`` ストリーミング集約とFedOptモデル更新
     - 対応
     - クライアントから ``FLModel(params_type=ParamsType.DIFF)`` を返します。独立した
       ``aggregator_data_kind`` パラメータはありません。
   * - ``SwarmLearningRecipe``
     - ``InTimeAccumulateWeightedAggregator``
     - 対応
     - ``expected_data_kind=DataKind.WEIGHT_DIFF`` を設定し、クライアントから
       ``FLModel(params_type=ParamsType.DIFF)`` を返します。
   * - ``SklearnFedAvgRecipe``
     - 組み込みの ``FedAvg`` ストリーミング集約
     - 条件付き
     - クライアントスクリプトが差分を計算し、明示的に
       ``FLModel(params_type=ParamsType.DIFF)`` を返す必要があります。

標準の ``InTimeAccumulateWeightedAggregator`` と
``HEInTimeAccumulateWeightedAggregator`` は、``expected_data_kind`` が適切に設定されていれば
``WEIGHTS`` と ``WEIGHT_DIFF`` の両方を受け付けます。``expected_data_kind`` を宣言する
アグリゲーターは、構築時にレシピの設定と照合されます。

たとえば、PyTorch FedAvgを差分を使うように設定するには、次のようにします。

.. code-block:: python

    from nvflare.apis.dxo import DataKind
    from nvflare.app_opt.pt.recipes import FedAvgRecipe

    recipe = FedAvgRecipe(
        name="fedavg-diff",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
        aggregator_data_kind=DataKind.WEIGHT_DIFF,
    )

クライアントスクリプトは、ローカルパラメータからグローバルパラメータを引いた差分を計算し、明示的に返す必要があります。

.. code-block:: python

    import nvflare.client as flare
    from nvflare.app_common.abstract.fl_model import ParamsType

    flare.send(flare.FLModel(params=model_diff, params_type=ParamsType.DIFF))

カスタムアグリゲーターが互換性のない ``expected_data_kind`` を宣言している場合、レシピの構築時に、
設定された種別と宣言された種別の両方、およびそれらを揃える方法を示すエラーが発生します。


FedProx
=======

FedProxは、データの不均一性(heterogeneity)に対処するために、クライアントの損失関数に近接項(proximal term)を追加したFedAvgです。
標準のFedAvgRecipeを使用し、クライアント側でFedProx損失ヘルパーを利用します。
PyTorch FedProxは ``FedAvgRecipe`` を使用するため、ストリーミングされるPyTorchテンソル更新に対する
``enable_tensor_disk_offload=True`` も、PyTorch FedAvgと同じ挙動・制約でサポートします。

PyTorch FedProx
---------------

.. code-block:: python

    from nvflare.app_opt.pt.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    # FedProx uses FedAvgRecipe with FedProxLoss in the client training script
    recipe = FedAvgRecipe(
        name="fedprox-pt",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
        train_args="--fedproxloss_mu 0.01",  # Pass mu parameter to client
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

クライアントのトレーニングスクリプトでは、FedProxLossヘルパーを使用します。

.. code-block:: python

    from nvflare.app_opt.pt import PTFedProxLoss

    # In training loop:
    fedprox_loss = PTFedProxLoss(mu=fedproxloss_mu)
    for data, target in train_loader:
        optimizer.zero_grad()
        output = model(data)
        ce_loss = criterion(output, target)
        # Add FedProx regularization term
        prox_loss = fedprox_loss(model)
        loss = ce_loss + prox_loss
        loss.backward()
        optimizer.step()

**例:**

- `examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedprox <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedprox>`_

TensorFlow FedProx
------------------

.. code-block:: python

    from nvflare.app_opt.tf.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = FedAvgRecipe(
        name="fedprox-tf",
        min_clients=2,
        num_rounds=5,
        model=MyTFModel(),
        train_script="client.py",
        train_args="--fedproxloss_mu 0.01",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

クライアントのトレーニングスクリプトでは、TensorFlow版のFedProxLossを使用します。

.. code-block:: python

    from nvflare.app_opt.tf.fedprox_loss import TFFedProxLoss

    fedprox_loss = TFFedProxLoss(mu=fedproxloss_mu)
    # Use in training loop

**例:**

- `examples/advanced/cifar10/tf/cifar10_fedprox <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/tf/cifar10_fedprox>`_


FedOpt(連合最適化)
===============================

サーバー側オプティマイザー(SGD、Adamなど)を用いた連合最適化です。

PyTorch FedOpt
--------------

.. code-block:: python

    from nvflare.app_opt.pt.recipes import FedOptRecipe
    from nvflare.recipe import SimEnv

    recipe = FedOptRecipe(
        name="fedopt-pt",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
        optimizer_args={"path": "torch.optim.SGD", "args": {"lr": 1.0, "momentum": 0.6}},
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

.. note::
   PyTorch FedOptは、ストリーミングされるPyTorchテンソル更新に対する ``enable_tensor_disk_offload=True`` をサポートします。
   ``nvflare.client.config`` から ``ExchangeFormat`` をインポートし、
   ``server_expected_format=ExchangeFormat.PYTORCH`` を設定することで、サーバー側パスが集約前に
   更新をNumPyに変換せず、テンソルのまま保持するようにしてください。

**例:**

- `examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedopt <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedopt>`_

TensorFlow FedOpt
-----------------

.. code-block:: python

    from nvflare.app_opt.tf.recipes import FedOptRecipe
    from nvflare.recipe import SimEnv

    recipe = FedOptRecipe(
        name="fedopt-tf",
        min_clients=2,
        num_rounds=5,
        model=MyTFModel(),
        train_script="client.py",
        optimizer_args={"path": "tensorflow.keras.optimizers.SGD", "args": {"learning_rate": 1.0}},
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/cifar10/tf/cifar10_fedopt <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/tf/cifar10_fedopt>`_


SCAFFOLD
========

制御変量(control variates)を用いてデータの不均一性に対処するSCAFFOLDアルゴリズムです。

PyTorch SCAFFOLD
----------------

.. code-block:: python

    from nvflare.app_opt.pt.recipes import ScaffoldRecipe
    from nvflare.recipe import SimEnv

    recipe = ScaffoldRecipe(
        name="scaffold-pt",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

PyTorch Lightningクライアントは、FedAvgとSCAFFOLDで同じパッチ適用済みトレーニングスクリプトを使用できます。

.. code-block:: python

    import nvflare.client.lightning as flare
    from pytorch_lightning import Trainer

    trainer = Trainer(max_epochs=1)
    flare.patch(trainer)

    while flare.is_running():
        flare.receive()
        trainer.fit(model, datamodule=data_module)

``flare.patch`` はSCAFFOLDのグローバル制御変数を検出し、各オプティマイザーステップの後に ``PTScaffoldHelper`` を適用し、
返される ``FLModel`` に必要な制御差分を追加します。この自動パスは、Lightningの自動最適化(automatic optimization)で
オプティマイザーが1つであり、そのパラメータグループが各ステップで同一の有限かつ非負の学習率を使用し、
1ラウンドあたりの学習率の合計が正である場合をサポートします。サポートされる精度モードは
``32-true`` と ``bf16-mixed`` です。手動最適化(manual optimization)の場合は、``flare.patch`` を使わずに
明示的なreceive/train/sendループを用い、``PTScaffoldHelper`` を直接組み込む必要があります。

.. note::

   NVFlare 2.9.0以降、PyTorch SCAFFOLDの制御差分にはトレーニング可能なパラメータのみが含まれます。
   BatchNormのランニング統計量などのバッファは通常のモデル状態のままであるため、カスタムSCAFFOLDアグリゲーターは
   疎な制御辞書を受け付ける必要があります。トレーニング可能性はラウンド間で変化する可能性があり、
   新たにトレーニング可能になったローカル制御変数はゼロにリセットされます。トレーニングラウンドの途中で
   ``requires_grad`` を変更することはサポートされていません。

**例:**

- `examples/advanced/cifar10/pt/cifar10-sim/cifar10_scaffold <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_scaffold>`_
- `examples/hello-world/hello-lightning <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-lightning>`_

TensorFlow SCAFFOLD
-------------------

.. code-block:: python

    from nvflare.app_opt.tf.recipes import ScaffoldRecipe
    from nvflare.recipe import SimEnv

    recipe = ScaffoldRecipe(
        name="scaffold-tf",
        min_clients=2,
        num_rounds=5,
        model=MyTFModel(),
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/cifar10/tf/cifar10_scaffold <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/tf/cifar10_scaffold>`_


巡回学習(Cyclic Learning)
====================================================

クライアント間を巡回する順序で逐次的にトレーニングを行います。

PyTorch Cyclic
--------------

.. code-block:: python

    from nvflare.app_opt.pt.recipes import CyclicRecipe
    from nvflare.recipe import SimEnv

    recipe = CyclicRecipe(
        name="cyclic-pt",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="client.py",
        task_assignment_timeout=30,
        shutdown_timeout=120.0,  # External client process only
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-cyclic <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-cyclic>`_

``task_assignment_timeout`` はサーバーの ``CyclicController`` を設定し、
``shutdown_timeout`` はクライアントの ``ScriptRunner`` を設定します。名前付きレシピパラメータが
用意されていないコントローラーまたはランナーのオプションについては、
``server_config_overrides`` または ``client_config_overrides`` を使用してください。
これらの辞書は名前付きパラメータの後にシャローマージされるため、重複する辞書値は
オーバーライド側が優先されます。``task_check_period`` をオーバーライドする場合は正の値でなければなりません。

.. code-block:: python

    recipe = CyclicRecipe(
        name="cyclic-advanced",
        min_clients=2,
        model=MyModel(),
        train_script="client.py",
        task_assignment_timeout=30,
        server_config_overrides={"task_check_period": 1.0},
        client_config_overrides={"launch_once": False},
    )

TensorFlow Cyclic
-----------------

.. code-block:: python

    from nvflare.app_opt.tf.recipes import CyclicRecipe
    from nvflare.recipe import SimEnv

    recipe = CyclicRecipe(
        name="cyclic-tf",
        min_clients=2,
        num_rounds=5,
        model=MyTFModel(),
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)


XGBoostレシピ
========================

ツリーベースモデル向けの連合XGBoostです。

XGBoost水平連合(ヒストグラムベース)
------------------------------------------------------------------------

水平データ分割向けのヒストグラムベース連合XGBoostです。

.. code-block:: python

    from nvflare.app_opt.xgboost.recipes import XGBHorizontalRecipe
    from nvflare.recipe import SimEnv

    recipe = XGBHorizontalRecipe(
        name="xgb-horizontal",
        min_clients=2,
        num_rounds=10,
        xgb_params={"max_depth": 6, "eta": 0.1, "objective": "binary:logistic"},
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/xgboost/fedxgb <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb>`_

XGBoostバギング(ツリーベース)
------------------------------------------------------------

バギングを用いたツリーベース連合XGBoostです。

.. code-block:: python

    from nvflare.app_opt.xgboost.recipes import XGBBaggingRecipe
    from nvflare.recipe import SimEnv

    recipe = XGBBaggingRecipe(
        name="xgb-bagging",
        min_clients=2,
        training_mode="bagging",
        num_rounds=10,
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/xgboost/fedxgb (job_tree.py) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb>`_

XGBoost垂直連合
----------------------------

垂直データ分割向けの連合XGBoostです。

.. code-block:: python

    from nvflare.app_opt.xgboost.recipes import XGBVerticalRecipe
    from nvflare.recipe import SimEnv

    recipe = XGBVerticalRecipe(
        name="xgb-vertical",
        min_clients=2,
        num_rounds=10,
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/xgboost/fedxgb (job_vertical.py) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb>`_
- `examples/advanced/xgboost/fedxgb_secure (job_vertical.py) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb_secure>`_


Sklearn特化レシピ
================================

K-Means FedAvg
--------------

連合K-Meansクラスタリングです。

.. code-block:: python

    from nvflare.app_opt.sklearn.recipes import KMeansFedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = KMeansFedAvgRecipe(
        name="kmeans",
        min_clients=2,
        num_rounds=5,
        n_clusters=3,
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/sklearn-kmeans <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/sklearn-kmeans>`_

SVM FedAvg
----------

連合サポートベクターマシン(SVM)です。

.. code-block:: python

    from nvflare.app_opt.sklearn.recipes import SVMFedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = SVMFedAvgRecipe(
        name="svm",
        min_clients=2,
        num_rounds=5,
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/sklearn-svm <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/sklearn-svm>`_

ロジスティック回帰FedAvg
------------------------------------------------

連合ロジスティック回帰です。

.. code-block:: python

    from nvflare.app_common.np.recipes.lr.fedavg import FedAvgLrRecipe
    from nvflare.recipe import SimEnv

    recipe = FedAvgLrRecipe(
        name="lr",
        min_clients=2,
        num_rounds=5,
        train_script="client.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-lr <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-lr>`_


連合統計
====================

分散したデータに対して連合統計量を計算します。

.. code-block:: python

    from nvflare.recipe import SimEnv
    from nvflare.recipe.fedstats import FedStatsRecipe

    recipe = FedStatsRecipe(
        name="stats",
        stats_output_path="./output",
        sites=["site-1", "site-2"],
        statistic_configs={"count": {}, "mean": {}, "stddev": {}},
        stats_generator=my_stats_generator,
    )
    env = SimEnv(clients=["site-1", "site-2"])
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-tabular-stats <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-tabular-stats>`_
- `examples/advanced/federated-statistics/df_stats <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/federated-statistics/df_stats>`_
- `examples/advanced/federated-statistics/image_stats <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/federated-statistics/image_stats>`_

Kaplan-Meier生存時間解析
------------------------------------------------

ビン化されたイベントヒストグラムに対するオプションの準同型暗号を用いた、連合Kaplan-Meier生存時間解析です。
``KMRecipe`` は、パッケージレベルのレシピとしてエクスポートされているのではなく、
Kaplan-Meierの例の ``job.py`` 内で定義されています。

``from job import KMRecipe`` が正しく解決されるよう、Kaplan-Meierの例のディレクトリからスニペットを実行してください。

.. code-block:: bash

    cd examples/advanced/kaplan-meier-he

.. code-block:: python

    from job import KMRecipe
    from nvflare.recipe import SimEnv

    # KMRecipe is defined in examples/advanced/kaplan-meier-he/job.py
    recipe = KMRecipe(
        num_clients=5,
        encryption=True,
        data_root="/tmp/nvflare/dataset/km_data",
        he_context_path_client="/tmp/nvflare/he_context/he_context_client.txt",
        he_context_path_server="/tmp/nvflare/he_context/he_context_server.txt",
    )
    env = SimEnv(num_clients=5)
    run = recipe.execute(env)

**例:**

- `examples/advanced/kaplan-meier-he <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/kaplan-meier-he>`_


連合評価
====================

事前学習済みモデルを複数のサイトにわたって評価します。

PyTorch FedEval
---------------

事前学習済みPyTorchモデルをすべてのクライアントに送信し、各クライアントのローカルデータで評価します。

.. code-block:: python

    from nvflare.app_opt.pt.recipes.fedeval import FedEvalRecipe
    from nvflare.recipe import SimEnv

    recipe = FedEvalRecipe(
        name="eval_job",
        model=MyModel(),
        eval_ckpt="/path/to/pretrained_model.pt",
        min_clients=2,
        eval_script="client.py",
        eval_args="--batch_size 32",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

.. note::
   ``eval_ckpt`` は **必須** です。次のいずれかを指定できます。

   * 事前学習済みチェックポイント(.pt、.pth)へのサーバー上の絶対パス、または
   * ジョブに同梱されるローカルチェックポイントファイルへの相対パスまたは絶対パス
     (たとえば ``prepare_initial_ckpt`` などのユーティリティを利用)。

   サーバー側の絶対パスを指定する場合、ジョブの構築時にそのチェックポイントファイルが
   ローカルに存在しなくてもかまいません。

**例:**

- `examples/hello-world/hello-lightning-eval <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-lightning-eval>`_


クロスサイト評価
================================

すべてのクライアントサイトにわたってモデルを評価します(各クライアントのモデルをすべてのデータセットと突き合わせて比較します)。

.. code-block:: python

    from nvflare.app_common.np.recipes import NumpyCrossSiteEvalRecipe
    from nvflare.recipe import SimEnv

    recipe = NumpyCrossSiteEvalRecipe(
        name="cross-eval",
        min_clients=2,
        eval_script="evaluate.py",
        eval_args="--data_root /path/to/data",
        initial_ckpt="/path/to/pretrained_model.npy",  # Optional: evaluate specific model
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

.. note::
   - カスタムの評価ロジックを指定するには ``eval_script`` を使用します。指定しない場合は、
     組み込みのダミーバリデーター(テスト専用)が使用されます。
   - 特定の事前学習済みモデルを評価するには ``initial_ckpt`` を使用します。指定しない場合、
     このレシピはトレーニング実行ディレクトリのモデルを評価します。

**例:**

- `examples/hello-world/hello-numpy-cross-val <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-numpy-cross-val>`_


Private Set Intersection(PSI)
==============================================================

クライアント間でプライベートな集合の共通部分(積集合)を計算します。

.. code-block:: python

    from nvflare.app_common.psi.recipes import DhPSIRecipe
    from nvflare.recipe import SimEnv

    recipe = DhPSIRecipe(
        name="psi",
        min_clients=2,
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/advanced/psi/user_email_match <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/psi/user_email_match>`_


Flower連携
====================

Flowerベースの連合学習ジョブを実行します。

.. code-block:: python

    from nvflare.app_opt.flower.recipe import FlowerRecipe
    from nvflare.recipe import SimEnv

    recipe = FlowerRecipe(
        name="flower-job",
        min_clients=2,
        flower_content="path/to/flower/app",
        run_config={"num-server-rounds": 5}, # Optional: used to override default values in pyproject.toml
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

**例:**

- `examples/hello-world/hello-flower <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-flower>`_

サーバー事前デプロイ型Flowerアプリモード
--------------------------------------------------------------------------------

BYOC(Bring Your Own Code)が制限されている本番環境デプロイメントでは、サーバーに事前デプロイされた
Flowerアプリを使用できます。このモードでは、Flowerアプリケーションコードは **管理者の管理下にあり、
既知のパスでサーバーに事前インストール** されており、NVFlareがFlowerのFAB
(Flower Application Bundle)メカニズムを介してクライアントに配布します。これによりBYOC認可が不要になります。

**重要**: ``flower_app_path`` は、ユーザーが指定するパスではなく、**サーバー管理者によって事前承認・管理
されている** アプリを参照する必要があります。パスはワークスペースの ``local/custom/`` ディレクトリ内に
なければなりません。これにより、アプリがジョブ送信時にユーザーが任意に選んだものではなく、
サーバー管理者によって事前デプロイされ管理されていることが保証されます。

.. code-block:: python

    from nvflare.app_opt.flower.recipe import FlowerRecipe
    from nvflare.recipe import ProdEnv

    recipe = FlowerRecipe(
        name="flower-job",
        min_clients=2,
        flower_app_path="local/custom/preapproved_apps/my_app",  # Admin-predeployed on server
        run_config={"num-server-rounds": 5},
    )
    env = ProdEnv(startup_kit_location="/path/to/startup_kit")
    run = recipe.execute(env)

**主な違い:**

- ``flower_content``: FlowerアプリをジョブZIPにパッケージします(BYOC認可が必要)
- ``flower_app_path``: サーバー上の **管理者事前デプロイ済み** アプリを参照します(BYOC不要)

**セキュリティモデル**: ``BYOC無効`` かつ ``flower_predeployed=true`` の状態で
``flower_app_path`` を使用する場合:

- **ユーザー提供のNVFlareカスタムコード** はジョブを通じてデプロイされません
- Flowerアプリコードは **サーバー管理者が管理する/事前承認されたアプリの場所** からのみ配布されます
- 管理者による管理を強制するため、``flower_app_path`` は ``local/custom/`` で始まる必要があります
- ユーザーが任意に選んだパスは **許可されません**。指定されたディレクトリ内の事前承認済みアプリのみが許可されます

**認可要件:**

``flower_app_path`` を使用するサイトでは、``authorization.json`` で
``server-predeployed-flwr`` 権限が付与されている必要があります。デフォルトでは、この権限は
すべてのロールに対して ``"none"`` (拒否)に設定されています。有効化するには次のようにします。

.. code-block:: json

    {
      "format_version": "1.0",
      "permissions": {
        "lead": {
          "server-predeployed-flwr": "any"
        }
      }
    }

サイトの認可ポリシーの詳細については :ref:`site_policy_management` を参照してください。


Swarm Learning
==============

中央サーバーを必要としない分散型の連合学習です。

.. code-block:: python

    from nvflare.app_opt.pt.recipes.swarm import SwarmLearningRecipe
    from nvflare.recipe import SimEnv

    recipe = SwarmLearningRecipe(
        name="swarm",
        model=MyModel(),
        min_clients=3,
        num_rounds=5,
        train_script="client.py",
        initial_ckpt="/path/to/pretrained.pt",  # Optional: pre-trained weights
        progress_timeout=7200,
        learn_task_timeout=None,  # No training-task time limit
        learn_task_ack_timeout=3600,
        final_result_ack_timeout=3600,
        max_concurrent_submissions=1,
    )
    env = SimEnv(num_clients=3)
    run = recipe.execute(env)

.. note::
   大規模モデル(2 GB超)の場合は、以下のパラメータを調整してください。

   - ``learn_task_timeout`` (デフォルト ``None``): トレーニングタスクの最大実行時間。
   - ``learn_task_ack_timeout`` と ``final_result_ack_timeout``: ピアツーピア(P2P)モデル転送の
     確認応答の許容時間。互換ショートカット ``round_timeout`` は、それぞれの明示的なパラメータが
     省略された場合に両方を設定します。
   - ``progress_timeout`` (デフォルト 3600秒): ワークフローが進行しない状態の最大許容時間。
   - ``max_concurrent_submissions`` (デフォルト 1、最小 1): 同時に行える集約送信の数。
   - ``pipe_type`` (デフォルト ``"cell_pipe"``): セルネットワーキングが利用できない場合や
     サードパーティのサブプロセス連携には ``"file_pipe"`` を設定します。
   - ``submit_result_timeout``、``download_complete_timeout``、
     ``tensor_min_download_timeout``、``PEER_READ_TIMEOUT``:
     ``recipe.add_client_config({...})`` で設定します。``max_resends`` はデフォルトで有限値
     ``3`` であり、同じ方法でオーバーライドできます —
     :ref:`timeout_troubleshooting` を参照してください。

高度なコントローラー設定については、``server_config_overrides`` と
``client_config_overrides`` が、名前付きパラメータの後に ``SwarmServerConfig`` と
``SwarmClientConfig`` にシャローマージされます。したがって、重複する辞書値は
ドキュメント化された名前付きAPIよりも優先されます。クライアント側のオーバーライドでは、
レシピが管理するエグゼキューター、アグリゲーター、パーシスター、shareableジェネレーター、
``min_responses_required`` を置き換えることはできません。カスタムコンポーネントやクォーラム設定には
``BaseSwarmLearningRecipe`` を使用してください。サーバー側のオーバーライドでは ``min_clients`` を
置き換えることはできません。スケジューラーとワークフローのすべてのクォーラム設定の整合性を保つため、
名前付きパラメータで設定してください。


エッジレシピ
========================

エッジデバイスの連合学習向けレシピです。

EdgeFedBuffRecipe
-----------------

.. code-block:: python

    from nvflare.edge.tools.edge_fed_buff_recipe import (
        EdgeFedBuffRecipe,
        ModelManagerConfig,
        DeviceManagerConfig,
    )

    recipe = EdgeFedBuffRecipe(
        job_name="edge-fedavg",
        model=MyModel(),
        model_manager_config=ModelManagerConfig(max_num_active_model_versions=3, max_model_version=20),
        device_manager_config=DeviceManagerConfig(device_selection_size=100),
        initial_ckpt="/path/to/pretrained.pt",  # Optional: pre-trained weights
    )

**例:**

- `examples/advanced/edge/jobs <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge/jobs>`_


ユーティリティ関数
==================================

実験トラッキングの追加
--------------------------------------------

任意のレシピに実験トラッキング(MLflow、TensorBoard、W&B)を追加します。

.. code-block:: python

    from nvflare.recipe.utils import add_experiment_tracking

    add_experiment_tracking(recipe, tracking_type="tensorboard")
    # or
    add_experiment_tracking(recipe, tracking_type="mlflow")
    # or
    add_experiment_tracking(recipe, tracking_type="wandb")

クロスサイト評価の追加
--------------------------------------------

任意のトレーニングレシピにクロスサイト評価を追加します。

.. code-block:: python

    from nvflare.recipe.utils import add_cross_site_evaluation

    add_cross_site_evaluation(recipe)
    # or limit evaluation to selected clients
    add_cross_site_evaluation(recipe, participating_clients=["site-1", "site-3"])
