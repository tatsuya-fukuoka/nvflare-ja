.. _mobile_training:

###############################################
モバイル連合トレーニング (iOS / Android)
###############################################

FLARE 2.7 では、`ExecuTorch <https://github.com/pytorch/executorch>`_ を利用して
モバイルデバイス (iOS および Android) 上で連合学習を実行できるようになりました。最大の利点は、
**デバイス側のプログラミングが不要である** ことです -- 標準的な PyTorch でモデルを開発すれば、
エクスポート、デプロイ、連合トレーニングのオーケストレーションはすべて FLARE が処理します。

動作の仕組み
==============

1. **モデルを設計する** -- 標準的な PyTorch で設計します (モバイル向けに軽量に保ってください)
2. **モデルをラップする** -- ExecuTorch 用の損失計算と予測ロジックを含む ``DeviceModel`` でラップします
3. **ETFedBuffRecipe を使用する** -- FLARE ジョブを作成します [1]_ -- 残りはすべて FLARE が処理します

モバイル SDK (Android および iOS) は、:ref:`エッジデバイス連携プロトコル (EDIP) <flare_edge>` に従って
HTTP 経由で FLARE サーバーと通信します。

ステップ 1 -- モデルアーキテクチャの設計
------------------------------------------

シングルマシンでのトレーニングと同じ要領で、PyTorch を使ってモデルを設計します。ただし、
モバイルデバイスの計算リソースには限りがあることに留意してください。サポートされるレイヤーは
標準的な PyTorch とは異なる場合があるため、
`ExecuTorch のドキュメント <https://github.com/pytorch/executorch>`_ を参照してください。

ステップ 2 -- DeviceModel の作成
----------------------------------

ExecuTorch では、トレーニング中にモデルが損失と予測の両方を返す必要があります。
モデルを ``DeviceModel`` でラップしてください。

.. code-block:: python

   from nvflare.edge.models.model import DeviceModel

   class TrainingNet(DeviceModel):
       def __init__(self):
           super().__init__(MyCifar10Net())

``DeviceModel`` 基底クラスには、デフォルトで ``CrossEntropyLoss`` が含まれています。必要に応じて
損失関数をオーバーライドできます。

ステップ 3 -- ETFedBuffRecipe による FLARE ジョブの作成
--------------------------------------------------------

モバイルデバイス向けの連合トレーニングジョブを作成するには、``ETFedBuffRecipe`` を使用します。

.. code-block:: python

   recipe = ETFedBuffRecipe(
       job_name=job_name,
       device_model=device_model,
       input_shape=input_shape,
       output_shape=output_shape,
       model_manager_config=ModelManagerConfig(
           max_model_version=3,
           update_timeout=1000.0,
           num_updates_for_model=total_num_of_devices,
       ),
       device_manager_config=DeviceManagerConfig(
           device_selection_size=total_num_of_devices,
           min_hole_to_fill=total_num_of_devices,
       ),
       evaluator_config=evaluator_config,
       simulation_config=(
           SimulationConfig(
               task_processor=task_processor,
               num_devices=num_of_simulated_devices_on_each_leaf,
           )
           if num_of_simulated_devices_on_each_leaf > 0
           else None
       ),
       device_training_params={"epoch": 3, "lr": 0.0001, "batch_size": batch_size},
   )

主なパラメータ:

- **device_model**: ステップ 2 で作成した ``DeviceModel`` ラッパー
- **input_shape, output_shape**: ExecuTorch モデルをエクスポートする際のテンソル形状
- **device_training_params**: 各デバイスに渡されるトレーニングのハイパーパラメータ

モバイル SDK ガイド
=====================

SDK の統合方法と API リファレンスの詳細については、以下を参照してください。

- :doc:`FLARE モバイル開発ガイド <flare_mobile>` -- Android と iOS の両方に対応した SDK アーキテクチャ、セットアップ、ベストプラクティス
- :doc:`Android SDK API リファレンス <mobile_android>` -- Android 向けの Kotlin/Java API リファレンス

サンプル
==========

モバイル連合トレーニングの完全な動作サンプルについては、
`エッジのサンプル <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ を参照してください。

参考文献
==========

.. [1] Nguyen, J., Malik, K., Zhan, H., Yousefpour, A., Rabbat, M., Malek, M., & Huba, D. (2023).
   Asynchronous Federated Learning with Bidirectional Quantized Communications and Buffered Aggregation.
   arXiv preprint arXiv:2308.00263. https://arxiv.org/pdf/2308.00263

.. toctree::
   :maxdepth: 1
   :hidden:

   mobile_android
