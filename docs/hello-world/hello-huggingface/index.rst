.. _hello_huggingface:

Hello HuggingFace
=================

この例では、NVIDIA FLARE と HuggingFace Client API を使用して、Qwen 因果言語モデルの
連合 PEFT/LoRA ファインチューニングを実行する方法を示します。
完全なサンプルコードは
:github_nvflare_link:`examples/hello-world/hello-huggingface <examples/hello-world/hello-huggingface>` にあります。

NVFLAREと依存関係のインストール
------------------------------------------------------------

完全なインストール手順については、:doc:`Installation </installation>` を参照してください。
リリース済みブランチの場合:

.. code-block:: text

   pip install nvflare

HuggingFace Client API は NVFlare 2.9.0 で導入されます。そのパッケージが公開される
までは、このリポジトリから NVFlare をインストールし、残りの例の依存関係を個別に
インストールしてください:

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git
   cd NVFlare
   python -m pip install -e .
   python -m pip install torch transformers accelerate datasets peft trl safetensors

``requirements.txt`` の ``nvflare~=2.9.0rc`` エントリは、最初の互換リリースを
記録しています。NVFlare 2.9.0 の公開後は、
``python -m pip install -r requirements.txt`` で完全な環境をインストールできます。

コード構造
--------------------

.. code-block:: text

   hello-huggingface
   |
   |-- client.py        # HuggingFace/TRL local training script
   |-- model.py         # Qwen LoRA server-side model
   |-- prepare_data.py  # writes synthetic per-site JSONL data
   |-- job.py           # job recipe for simulation
   |-- requirements.txt
   |-- README.md

データ
------------

デフォルトの2クライアント用合成JSONLデータセットを準備します:

.. code-block:: bash

   python prepare_data.py

デフォルトでは以下の場所に書き込まれます:

.. code-block:: text

   /tmp/nvflare/hello-huggingface/data
   |
   |-- site-1
   |   |-- train.jsonl
   |   |-- valid.jsonl
   |-- site-2
   |   |-- train.jsonl
   |   |-- valid.jsonl

``prepare_data.py`` と ``job.py`` に ``--data_root`` を渡すことで、独自に準備した
データを使用できます。

クライアントコード
------------------------------------

クライアントスクリプトは、通常の HuggingFace/TRL ``SFTTrainer`` スクリプトです。
連合学習向けの変更は意図的に最小限に抑えられています:

.. code-block:: python

   import nvflare.client.hf as flare

   flare.init()
   site_name = flare.get_site_name()

   flare.patch(trainer)

   while flare.is_running():
       trainer.evaluate()
       trainer.train()

``flare.patch(trainer)`` はトレーナーのメソッドをラップし、スクリプトがグローバル
パラメータを受け取り、評価を実行し、ローカルトレーニングの割り当て分を実行し、
結果を FL サーバーに送り返せるようにします。

ジョブの実行
------------------------

データを準備した後、シミュレーションを実行します:

.. code-block:: bash

   python job.py

このジョブは、PEFT/LoRA を使用して、2つのシミュレートされたクライアントで2回の
FLラウンドを実行します。各クライアントは、Client API の初期化後に
``<data_root>/<site_name>/`` から自身のデータを解決します。フルモデルおよび
マルチノードの Qwen ワークフローについては、
:github_nvflare_link:`examples/advanced/llm_hf <examples/advanced/llm_hf>` を参照してください。

さらに学ぶ
--------------------

HuggingFace Client API のコントラクトとオプションについては、:ref:`hf_client_api` を参照してください。
