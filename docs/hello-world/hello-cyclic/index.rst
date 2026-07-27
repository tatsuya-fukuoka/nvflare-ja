Hello Cyclic Weight Transfer
============================

`Cyclic Weight Transfer <https://pubmed.ncbi.nlm.nih.gov/29617797/>`_ (CWT) は `FedAvg <https://arxiv.org/abs/1602.05629>`_ の代替手法です。CWT は `Cyclic Controller <https://nvflare.readthedocs.io/en/main/apidocs/nvflare.app_common.workflows.cyclic.html>`_ を使用して、モデルの重みをあるサイトから次のサイトへ受け渡し、繰り返しファインチューニングを行います。

.. note::

   この例では `MNIST <http://yann.lecun.com/exdb/mnist/>`_ の手書き数字データセットを使用し、trainer のコード内でデータを読み込みます。

GPU で TensorFlow を実行する
------------------------------------

GPU を使用したい場合は `NVIDIA TensorFlow docker <https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow>`_ の利用を推奨します。
GPU を使って実行する必要がない場合は、Python の仮想環境をそのまま使用できます。

NVIDIA TensorFlow コンテナを実行する
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

まず `NVIDIA container toolkit <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html>`_ をインストールしてください。
その後、次のコマンドを実行します。

.. code-block:: bash

   docker run --gpus=all -it --rm -v [path_to_NVFlare]:/NVFlare nvcr.io/nvidia/tensorflow:xx.xx-tf2-py3

GPU を使って実行する際の注意点
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

GPU を使ってこの例を実行する場合、TensorFlow はデフォルトで開始時に利用可能な GPU メモリをすべて確保しようとする点に
注意することが重要です。
複数のクライアントが関与するシナリオでは、次のフラグを設定して TensorFlow がすべての GPU メモリを確保しないように
する必要があります。

.. code-block:: bash

   TF_FORCE_GPU_ALLOW_GROWTH=true TF_GPU_ALLOCATOR=cuda_malloc_async

NVFlare のインストール
------------------------------

インストール手順の詳細については `Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: text

   pip install nvflare

GitHub からサンプルコードを取得します。

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git
   git switch <release branch>
   cd examples/hello-world/hello-cyclic


依存関係をインストールします。

.. code-block:: text

   pip install -r requirements.txt

コード構造
--------------

コード構造は次のとおりです。

.. code-block:: text

   hello-cyclic
   |
   |-- client.py           # client local training script
   |-- model.py            # model definition
   |-- job.py              # job recipe that defines client and server configurations
   |-- prepare_data.sh     # scripts to download the data
   |-- requirements.txt    # dependencies

データ
--------

この例では、TensorFlow Keras API によって提供される MNIST データセットを使用します。

モデル
--------


model.py ファイルでは、TensorFlow の Keras API を使用してシンプルなニューラルネットワークを定義しています。Net モデルは画像分類向けに設計されたシーケンシャルなアーキテクチャで、次の要素を備えています。

- Flatten レイヤ: 全結合レイヤ向けに入力データを整形します。
- Dense レイヤ: 非線形性のための ReLU 活性化関数を持つ 128 ユニット。
- Dropout レイヤ: 過学習を抑制するための 20% のドロップアウト率。
- 出力レイヤ: MNIST の数字を分類するための 10 ユニット。


.. literalinclude:: ../../../examples/hello-world/hello-cyclic/model.py
    :language: python
    :linenos:
    :caption: Model (model.py)
    :lines: 14-


クライアントコード
--------------------

クライアントコード ``client.py`` は学習を担当します。学習コードが標準的な PyTorch の学習コードとほぼ同じである点に注目してください。
唯一の違いは、サーバとの間でデータを受信・送信するための数行を追加していることです。


.. literalinclude:: ../../../examples/hello-world/hello-cyclic/client.py
    :language: python
    :linenos:
    :caption: Client Code (client.py)
    :lines: 14-

サーバコード
--------------

cyclic transfer では、サーバコードはあるクライアントから別のクライアントへモデル更新を順次受け渡す役割を担います。ここでは
NVFlare が提供するデフォルトのフェデレーテッド cyclic アルゴリズムをそのまま使用します。

Job Recipe
----------


.. literalinclude:: ../../../examples/hello-world/hello-cyclic/job.py
    :language: python
    :linenos:
    :caption: job recipe (job.py)
    :lines: 14-


実験を実行する
------------------

まずデータを準備します。

.. code-block:: bash

   bash ./prepare_data.sh
   python job.py

ログと結果を確認する
------------------------------

実行時のログと結果は、シミュレータのワークスペース内で確認できます。

.. code-block:: bash

   $ ls "/tmp/nvflare/simulation/cyclic"
