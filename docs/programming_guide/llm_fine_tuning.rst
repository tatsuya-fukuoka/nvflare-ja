.. _llm_fine_tuning:

##################################################
フェデレーテッド LLM ファインチューニング
##################################################

NVIDIA FLARE は、HuggingFace Transformers や NVIDIA NeMo などの一般的な
フレームワークを用いた、大規模言語モデル (LLM) のフェデレーテッドな
ファインチューニングをサポートしています。複数のファインチューニング戦略が
サポートされています。

- **SFT (Supervised Fine-Tuning、教師ありファインチューニング)** -- タスク固有のデータによるモデル全体または一部のファインチューニング
- **PEFT (Parameter-Efficient Fine-Tuning、パラメータ効率的ファインチューニング)** -- パラメータのごく一部のみを学習する LoRA などのアダプタベースの手法

いずれのアプローチも FLARE Client API を使用するため、既存のシングルマシン用
ファインチューニングスクリプトを、最小限のコード変更でフェデレーテッドに
変換できます。

HuggingFace との統合
=====================

FLARE は ``nvflare.client.hf`` により、HuggingFace モデルのフェデレーテッドな
ファインチューニングを直接サポートしています。このファサードは HuggingFace の
``Trainer`` や TRL の ``SFTTrainer`` にパッチを適用し、既存の
``trainer.evaluate()`` や ``trainer.train()`` の呼び出しが FL ラウンドに
参加できるようにします。

**フェデレーテッド SFT** は、複数サイトにまたがってモデル全体(または選択した
レイヤ)をファインチューニングします。

.. code-block:: python

   # client.py -- standard HuggingFace training, federated via HuggingFace Client API
   import nvflare.client.hf as flare

   trainer = SFTTrainer(...)
   flare.patch(trainer)

   while flare.is_running():
       trainer.evaluate()
       trainer.train()

**フェデレーテッド PEFT (LoRA)** はアダプタのパラメータのみを学習し、通信コストを
劇的に削減します。重み全体の送信が現実的でない大規模モデルに最適です。

完全なサンプルは以下を参照してください。

- `Hello HuggingFace Qwen サンプル <https://github.com/NVIDIA/NVFlare/tree/main/examples/hello-world/hello-huggingface>`_
- :github_nvflare_link:`高度な LLM HuggingFace サンプル <examples/advanced/llm_hf>`
- :ref:`hf_client_api`

NVIDIA NeMo との統合
=====================

NVIDIA NeMo モデルについては、FLARE は複数のファインチューニング戦略に対する
緊密な統合を提供しています。

- `NeMo によるフェデレーテッド SFT <https://github.com/NVIDIA/NVFlare/tree/main/integration/nemo/examples/supervised_fine_tuning>`_ -- 複数サイトにまたがる NeMo モデルの教師ありファインチューニング
- `NeMo によるフェデレーテッド PEFT <https://github.com/NVIDIA/NVFlare/tree/main/integration/nemo/examples/peft>`_ -- NeMo によるパラメータ効率的な LoRA ファインチューニング

セルフペース学習
=================

フェデレーテッド LLM 学習を体系的に学べる学習パスは以下のとおりです。

- `第8章: フェデレーテッド LLM 学習 <https://github.com/NVIDIA/NVFlare/tree/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-8_federated_LLM_training>`_

  - 8.1 フェデレーテッド BERT
  - 8.2 フェデレーテッド SFT
  - 8.3 フェデレーテッド PEFT
  - 8.4 通信のための LLM 量子化
  - 8.5 LLM ストリーミング
