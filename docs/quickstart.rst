.. _quickstart:
.. _get_started:
.. _getting_started:

######################################################
クイックスタートシリーズ
######################################################

NVIDIA FLAREクイックスタートシリーズへようこそ! このガイドでは、NVIDIA FLAREを使った連合学習(Federated Learning)プログラムの構築方法を素早く学べるように、一連のHello Worldサンプルを提供します。

先に進む前に、:ref:`installation` の手順が完了していることを確認してください。

実行モード
====================

FLAREは、ワークフローの各段階に応じた3つのモードをサポートしています:

- **シミュレーター** (:ref:`fl_simulator`) -- 高速なテストとアルゴリズム開発のために、単一のシステム上でジョブを実行します。
- **POC** (:ref:`poc_command`) -- クライアントとサーバーを別々のプロセスとして、1台のホスト上でデプロイメントをシミュレートします。
- **本番環境** (:ref:`provisioned_setup`) -- プロビジョニングで生成されたスタートアップキットを使用した分散デプロイメントです。

開発には **シミュレーター** から始め、**POC** で検証してから **本番環境** に進みます。


MLコードを連合学習に変換する
============================================================

既存のトレーニングコードを連合学習に変換するのに必要な変更は、次の3つだけです:

**ステップ1: トレーニングスクリプトにFLAREのインポートを追加する**

.. code-block:: python

    import nvflare.client as flare

**ステップ2: FLAREを初期化し、トレーニングループをラップする**

.. code-block:: python

    flare.init()

    while flare.is_running():
        input_model = flare.receive()           # receive global model
        model.load_state_dict(input_model.params)

        # ... your existing training code here ...

        output_model = flare.FLModel(
            params=model.cpu().state_dict(),
            metrics={"accuracy": accuracy},
        )
        flare.send(output_model)                # send updated model back

**ステップ3: FLワークフローを定義するジョブレシピを作成する**

.. code-block:: python

    from model import MyModel
    from nvflare.app_opt.pt.recipes import FedAvgRecipe
    from nvflare.recipe import SimEnv

    recipe = FedAvgRecipe(
        name="my-fedavg-job",
        min_clients=2,
        num_rounds=5,
        model=MyModel(),
        train_script="train.py",
    )
    env = SimEnv(num_clients=2)
    run = recipe.execute(env)

これだけです。トレーニングロジックはそのまま変わりません -- 通信、集約、オーケストレーションはFLAREが処理します。
Client APIの完全なリファレンスは :ref:`Client API <client_api>` を参照してください。事前構築済みレシピについては :ref:`利用可能なレシピ <available_recipes>` を参照してください。

Hello Worldサンプル
========================================

以下のHello Worldサンプルは、さまざまな連合学習アルゴリズムとワークフローを実演します。各サンプルには、始めるための手順とコードが含まれています。

1. **Hello PyTorch** - PyTorchのモデルとトレーニングループを使った連合平均。 :doc:`hello-world/hello-pt/index`

2. **Hello Lightning** - 効率的なモデルトレーニングのためにPyTorch Lightningを使用するサンプル。 :doc:`hello-world/hello-lightning/index`

3. **Hello Differential Privacy** - `PyTorchとOpacusを使った差分プライバシーによる、プライバシーを保護する連合学習トレーニング。 <hello-world/hello-dp/index.html>`_

4. **Hello TensorFlow** - `TensorFlowモデルを使った連合平均。 <hello-world/hello-tf/index.html>`_

5. **Hello JAX** - `MNISTでJAX、Flax、Optaxを使った連合平均。 <hello-world/hello-jax/index.html>`_

6. **Hello HuggingFace** - `HuggingFace TrainerとTRLを使った連合Qwen SFT/PEFT。 <hello-world/hello-huggingface/index.html>`_

7. **Hello Logistic Regression** - `scikit-learnを使った連合ロジスティック回帰のサンプル。 <hello-world/hello-lr/index.html>`_

8. **Hello Cyclic** - `Cyclic連合学習ワークフローのサンプル。 <hello-world/hello-cyclic/index.html>`_

9. **Hello Tabular Statistics** - `連合統計計算のサンプル。 <hello-world/hello-tabular-stats/index.html>`_

10. **Hello Flower** - `FLARE内でのFlowerアプリの実行。 <hello-world/hello-flower/index.html>`_

11. **Hello XGBoost** - `連合学習環境におけるテーブルデータの勾配ブースティングを実演する連合XGBoostのサンプル。 <hello-world/hello-xgboost/index.html>`_

まずはHello PyTorchから始めましょう: :doc:`hello-world/hello-pt/index`

.. toctree::
   :maxdepth: 1
   :hidden:

   hello-world/hello-pt/index
   hello-world/hello-tf/index
   hello-world/hello-jax/index
   hello-world/hello-huggingface/index
   hello-world/hello-lightning/index
   hello-world/hello-xgboost/index
   hello-world/hello-dp/index
   hello-world/hello-flower/index
   hello-world/hello-lr/index
   hello-world/hello-tabular-stats/index
   hello-world/hello-cyclic/index
