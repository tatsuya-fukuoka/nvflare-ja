Hello JAX
=========

.. warning::

   この例は、次期リリースに向けた NVFlare の開発ブランチである ``main`` に追随しています。
   ``main`` では、例の ``requirements.txt`` ファイルが、ある機能をサポートする最初の
   今後リリース予定の NVFlare バージョンを、そのパッケージが PyPI で公開される前から
   固定(pin)している場合があります。
   固定された ``nvflare`` バージョンがまだ入手できない場合は、PyPI からではなく
   このリポジトリから NVFlare をインストールしてください。

この例では、NVIDIA FLARE と JAX、Flax、Optax を使用して、連合平均(FedAvg)により MNIST 分類器をトレーニングする方法を示します。``hello-pt`` と同じ hello-world レシピ構造に従いますが、JAX のクライアントトレーニングループと、転送用に平坦化されたパラメータベクトルを使用します。

NVFLAREと依存関係のインストール
------------------------------------------------------------

完全なインストール手順については、`Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: bash

   pip install nvflare

まず GitHub からサンプルコードを取得します:

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git

次に hello-jax ディレクトリに移動します:

.. code-block:: bash

   git switch <release branch>
   cd examples/hello-world/hello-jax

依存関係をインストールします:

.. code-block:: bash

   pip install -r requirements.txt

コード構造
--------------------

.. code-block:: text

   hello-jax
   |
   |-- client.py         # client local training script
   |-- model.py          # JAX/Flax model helpers
   |-- prepare_data.py   # helper that downloads MNIST and writes .npy files
   |-- prepare_model.py  # helper that writes the initial flattened checkpoint
   |-- job.py            # job recipe that defines client and server configurations
   |-- requirements.txt  # dependencies

データ
------------

この例では `MNIST <https://www.tensorflow.org/datasets/catalog/mnist>`_ データセットを使用します。ジョブスクリプトは、シミュレーターの開始前に生の MNIST ファイルを一度ダウンロードし、``.npy`` ファイルに変換します。その後、各クライアントは準備されたキャッシュから読み込みます。

モデル
------------

:github_nvflare_link:`model.py <examples/hello-world/hello-jax/model.py>` のモデルは、Flax で実装された小さな畳み込みニューラルネットワークです。

.. literalinclude:: ../../../examples/hello-world/hello-jax/model.py
    :language: python
    :linenos:
    :caption: model code (model.py)
    :lines: 14-

クライアントコード
------------------------------------

クライアントコード(:github_nvflare_link:`client.py <examples/hello-world/hello-jax/client.py>`)は、ローカルトレーニングループを JAX のまま維持しながら、NVFlare の Client API を使用して現在のグローバルモデルを受け取り、更新されたパラメータを返します。

.. literalinclude:: ../../../examples/hello-world/hello-jax/client.py
    :language: python
    :linenos:
    :caption: client code (client.py)
    :lines: 14-

サーバーコード
----------------------------

この例では、NumPy パラメータ交換用に構成されたベースの ``FedAvgRecipe`` を使用します。JAX のパラメータツリーは、サーバーと交換される前に単一の NumPy ベクトルに平坦化され、各トレーニングラウンドの前にクライアント側で再構築されます。

ジョブを実行する前に、2つのリソースを準備します:

- 初期の平坦化されたチェックポイントは ``prepare_model.py`` によって生成され、``initial_ckpt`` を通じて ``FedAvgRecipe`` に渡されます。
- 共有の MNIST ``.npy`` キャッシュは ``prepare_data.py`` によって一度だけ準備されます。これにより、2つのシミュレートされたクライアントが同時にデータセットをダウンロードしようとしたり、TensorFlow 専用のデータユーティリティに依存したりすることを防ぎます。

アセットの準備
----------------------------

``/tmp/nvflare/data/hello-jax`` 配下のデフォルトの場所を使用して、初期チェックポイントとデータセットを準備します:

.. code-block:: bash

   python prepare_model.py
   python prepare_data.py

カスタムの場所に準備することもできます:

.. code-block:: bash

   python prepare_model.py --output /path/to/initial_model.npy
   python prepare_data.py --data_dir /path/to/mnist

ジョブレシピコード
------------------------------------

.. literalinclude:: ../../../examples/hello-world/hello-jax/job.py
    :language: python
    :linenos:
    :caption: job recipe (job.py)
    :lines: 14-

ジョブの実行
------------------------

アセットの準備ができたら、ジョブスクリプトを実行して、シミュレーション環境でジョブを実行します。

.. code-block:: bash

   python job.py

必要に応じて、コマンドラインから主要なハイパーパラメータを調整できます:

.. code-block:: bash

   python job.py --n_clients 2 --num_rounds 3 --epochs 1 --batch_size 128

デフォルト以外の場所にアセットを準備した場合は、それらを明示的に渡してください:

.. code-block:: bash

   python job.py --initial_ckpt /path/to/initial_model.npy --data_dir /path/to/mnist

出力の概要
--------------------

- **初期化**: ``BaseModelController`` が FedAvg ワークフローを開始し、初期の平坦化されたチェックポイントを読み込み、``/tmp/nvflare/simulation/hello-jax`` 配下にシミュレーション出力を書き込みます。
- **ラウンド0**: ``site-1`` と ``site-2`` がサンプリングされ、受信したモデルを精度 ``0.0527`` と ``0.0398`` で評価した後、1エポックのトレーニングを行い、トレーニング精度 ``0.8887`` / ``0.8857``、損失 ``0.3616`` / ``0.3778`` に到達します。クライアントログには、更新がサーバーに送り返される前に、``trained_model_eval_loss`` と ``accuracy`` を含むトレーニング後の評価行も記録されます。
- **ラウンド1**: 両サイトが再度サンプリングされ、受信したモデルの精度は ``0.9545`` と ``0.9799`` に向上し、ローカルトレーニングは精度 ``0.9702`` / ``0.9686``、損失 ``0.0990`` / ``0.0999`` に達します。クライアントログには、トレーニング済みローカルモデルに対する2回目の評価パスが記録され、集約された検証メトリクスは新たなベストの ``0.9671875`` になります。
- **ラウンド2**: 受信したモデルの精度は再び ``0.9762`` と ``0.9900`` に向上し、ローカルトレーニングは精度 ``0.9795`` / ``0.9790``、損失 ``0.0671`` / ``0.0683`` で終了します。クライアントログには、ローカルトレーニング後の ``trained_model_eval_loss`` と ``accuracy`` が再び報告され、集約された検証メトリクスは新たなベストの ``0.98310546875`` になります。
- **完了**: FedAvg は3ラウンドの後に終了し、最終的な NumPy チェックポイントを ``/tmp/nvflare/simulation/hello-jax/server/simulate_job/models/server.npy`` に永続化し、シミュレーション結果ディレクトリ ``/tmp/nvflare/simulation/hello-jax`` を報告します。
