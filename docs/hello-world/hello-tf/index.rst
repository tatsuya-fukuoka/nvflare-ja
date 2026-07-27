Hello TensorFlow
================

この例では、`NVIDIA FLARE <https://nvflare.readthedocs.io/en/main/index.html>`_ を TensorFlow と組み合わせて、フェデレーテッドアベレージング（`FedAvg <https://arxiv.org/abs/1602.05629>`_）により画像分類器を学習する方法を示します。この例では TensorFlow がディープラーニングの学習フレームワークとして機能します。

詳細なドキュメントについては `Hello TensorFlow <https://www.tensorflow.org/datasets/catalog/mnist>`_ のサンプルページを参照してください。

GPU をサポートするには `NVIDIA TensorFlow docker <https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow>`_ の利用を推奨します。GPU が不要であれば、Python の仮想環境で十分です。

この例を FLARE API で実行するには、:github_nvflare_link:`hello_world notebook <examples/hello-world/hello_world.ipynb>` を参照してください。

NVIDIA TensorFlow コンテナの実行
------------------------------------------

`NVIDIA container toolkit <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>`_ がインストールされていることを確認してください。その後、次のコマンドを実行します。

.. code-block:: bash

   docker run --gpus=all -it --rm -v [path_to_NVFlare]:/NVFlare nvcr.io/nvidia/tensorflow:xx.xx-tf2-py3

NVIDIA FLARE のインストール
------------------------------------

インストール手順の詳細については `Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: bash

   pip install nvflare

GitHub からサンプルコードをクローンします。

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git

hello-tf ディレクトリに移動します。

.. code-block:: bash

   git switch <release branch>
   cd examples/hello-world/hello-tf

依存関係をインストールします。

.. code-block:: bash

   pip install -r requirements.txt

コード構造
--------------

.. code-block:: text

   hello-pt
   |
   |-- client.py         # client local training script
   |-- model.py          # model definition
   |-- job.py            # job recipe that defines client and server configurations
   |-- requirements.txt  # dependencies

データ
--------

この例では `MNIST <https://www.tensorflow.org/datasets/catalog/mnist>`_ の手書き数字データセットを使用し、trainer のコード内で読み込みます。

モデル
--------

`model.py` ファイルでは、TensorFlow の Keras API を使用してシンプルなニューラルネットワークを定義しています。`Net` モデルは画像分類向けに設計されたシーケンシャルなアーキテクチャで、次の要素を備えています。

- **Flatten レイヤ**: 全結合レイヤ向けに入力データを整形します。
- **Dense レイヤ**: 非線形性のための ReLU 活性化関数を持つ 128 ユニット。
- **Dropout レイヤ**: 過学習を抑制するための 20% のドロップアウト率。
- **出力レイヤ**: MNIST の数字を分類するための 10 ユニット。

このモデルは NVIDIA FLARE によるフェデレーテッドラーニングで使用され、FedAvg アルゴリズムを用いて複数のクライアントにまたがって学習されます。


.. literalinclude:: ../../../examples/hello-world/hello-tf/model.py
    :language: python
    :linenos:
    :caption: model code (model.py)
    :lines: 14-


クライアントコード
--------------------

クライアントコード ``client.py`` は学習を担当します。学習コードは標準的な PyTorch の学習コードとよく似ており、サーバとのデータ交換を扱う行が追加されています。

.. literalinclude:: ../../../examples/hello-world/hello-tf/client.py
    :language: python
    :linenos:
    :caption: client code (client.py)
    :lines: 14-


サーバコード
--------------

フェデレーテッドアベレージングでは、サーバコードは scatter-gather のワークフローパターンに従って、クライアントからのモデル更新を集約します。この例では NVFlare が提供するデフォルトのフェデレーテッドアベレージングアルゴリズムを使用するため、カスタムのサーバコードは不要です。

Job Recipe のコード
-----------------------

job recipe には `client.py` と組み込みの FedAvg アルゴリズムが含まれます。


.. literalinclude:: ../../../examples/hello-world/hello-tf/job.py
    :language: python
    :linenos:
    :caption: job recipe (job.py)
    :lines: 14-

モデル入力のオプション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``model`` パラメータは 2 つの形式を受け付けます。

1. **クラスインスタンス（サブクラス化した Keras モデル）**: ``model=Net()`` - 手軽で Python らしい書き方です
2. **dict 設定**: ``model={"class_path": "model.Net", "args": {}}`` - 大きなモデルに適しています

事前学習済みの重みから再開するには、次のようにします。

.. code-block:: python

   recipe = FedAvgRecipe(
       model=Net(),
       initial_ckpt="/server/path/to/pretrained.h5",  # Absolute path
       ...
   )

.. note::

   TensorFlow/Keras の場合、``model`` にはサブクラス化した Keras クラスのインスタンス（例: ``Net()``）または dict 設定を使用してください。
   SavedModel や .h5 ファイルにはアーキテクチャと重みの両方が含まれるため、``model`` を指定せずに ``initial_ckpt`` を使用できます。

実験を実行する
------------------

job API を使用してジョブを作成し、シミュレータで実行するスクリプトを実行します。

.. code-block:: bash

   TF_FORCE_GPU_ALLOW_GROWTH=true TF_GPU_ALLOCATOR=cuda_malloc_async python3 job.py

ログと結果を確認する
------------------------------

実行時のログと結果は、シミュレータのワークスペース内で確認できます。

.. code-block:: bash

   $ ls /tmp/nvflare/jobs/workdir

GPU を使って実行する際の注意点
------------------------------------------

GPU を使用する場合、TensorFlow は起動時に利用可能な GPU メモリをすべて確保しようとします。複数クライアントのシナリオでこれを防ぐには、次のフラグを設定します。

.. code-block:: bash

   TF_FORCE_GPU_ALLOW_GROWTH=true TF_GPU_ALLOCATOR=cuda_malloc_async

クライアント数よりも GPU の数が多い場合は、シミュレーション時に `--gpu` 引数を使用して GPU ごとに 1 クライアントを実行することを検討してください（例: `nvflare simulator -n 2 --gpu 0,1 [job]`）。
