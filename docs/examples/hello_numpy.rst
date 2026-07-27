.. _hello_numpy:

Hello NumPy
===========

この例では、NVIDIA FLARE を NumPy と組み合わせて使い、フェデレーテッドアベレージング (FedAvg) でシンプルなモデルを学習する方法を示します。完全なサンプルコードは :github_nvflare_link:`hello-numpy ディレクトリ <examples/hello-world/hello-numpy>` にあります。
仮想環境を作成し、その仮想環境内ですべてを実行することを推奨します。

NVIDIA FLARE のインストール
----------------------------
完全なインストール手順については :doc:`../installation` を参照してください。

.. code-block:: text

    pip install nvflare

依存関係をインストールします。

.. code-block:: text

    pip install -r requirements.txt


コード構成
--------------

GitHub からサンプルコードを取得します。

.. code-block:: text

    git clone https://github.com/NVIDIA/NVFlare.git

hello-numpy ディレクトリに移動します。

.. code-block:: text

    git switch <release branch>
    cd examples/hello-world/hello-numpy


.. code-block:: text

    hello-numpy
        |
        |-- client.py             # client local training script
        |-- model.py              # model definition
        |-- job.py                # job recipe that defines client and server configurations
        |-- requirements.txt      # dependencies


データ
-----------------
この例では、簡略化された合成データセットを使用します。各クライアントは3x3の重み行列に対して基本的な演算を行い、学習中に各重みに小さなデルタを加えます。この方法により、実データの読み込みや前処理の複雑さを伴わずに、フェデレーテッドラーニングの集約プロセスを明確に観察できます。

実際のFL実験では、各クライアントがローカル学習に使う独自のデータセットを持つことになります。
ここでは簡潔さのために、フェデレーテッドラーニングの集約プロセスをはっきりと確認し理解できるような合成データを使用しています。

モデル
------------------
この例では、クライアントコード内にスクリプト形式で記述された NumPy のモデル更新ループを使用し、シンプルな3x3の重み行列によって
集約の挙動を分かりやすく示します。

完全な実装は次を参照してください。

- :github_nvflare_link:`client.py <examples/hello-world/hello-numpy/client.py>`



クライアントコード
------------------
クライアントの学習スクリプトは、標準的なFLクライアントのパターンに従います。

クライアント側では、学習ワークフローは次のようになります。

   1. FLサーバーからモデルを受け取る
   2. 受け取ったグローバルモデルに対して学習を行う
   3. 更新したモデルをFLサーバーに送り返す

NVFlare の Client API を使う場合、このワークフローを実現するための必須メソッドが3つあります。

   - ``flare.init()``: NVFlare Client API 環境を初期化します
   - ``flare.receive()``: FLサーバーからモデルを受け取ります
   - ``flare.send()``: FLサーバーへモデルを送信します

次のコードスニペットは、これらのメソッドの使い方を示しています。

.. code-block:: python

   import nvflare.client as flare

   flare.init()  # 1. Initialize NVFlare Client API
   input_model = flare.receive()  # 2. Receive model from server
   params = input_model.params  # 3. Extract model parameters

   # Your training code here
   new_params = train(params)

   output_model = flare.FLModel(params=new_params)  # 4. Package results
   flare.send(output_model)  # 5. Send updated model to server

完全な実装は次を参照してください。

- :github_nvflare_link:`client.py <examples/hello-world/hello-numpy/client.py>`


サーバーコード
------------------
フェデレーテッドアベレージングでは、サーバーコードはクライアントからのモデル更新を集約する役割を担います。ワークフローのパターンは scatter-gather に似ています。
この例では、NVFlare が提供するデフォルトのフェデレーテッドアベレージングアルゴリズムをそのまま使用します。
FedAvg クラスは `nvflare.app_common.workflows.fedavg.FedAvg` に定義されています。
この例では、カスタマイズしたサーバーコードを定義する必要はありません。


ジョブレシピのコード
--------------------
ジョブレシピには client.py と組み込みの fedavg アルゴリズムが含まれます。


ジョブレシピの実装は次を参照してください。

- :github_nvflare_link:`job.py <examples/hello-world/hello-numpy/job.py>`


モデル入力のオプション
^^^^^^^^^^^^^^^^^^^^^^^
NumPy のレシピでは、``model`` には NumPy 配列またはリストを指定できます。事前学習済みの重みから再開するには次のようにします。

.. code-block:: python

   recipe = NumpyFedAvgRecipe(
       model=None,  # Optional when using initial_ckpt
       initial_ckpt="/server/path/to/model.npy",  # Absolute path
       ...
   )

.. note::

   NumPy のチェックポイントにはモデルデータ全体が含まれるため、``initial_ckpt`` は ``model`` なしで使用できます。


FLジョブの実行
---------------
このセクションでは、上で定義したジョブレシピを使ってフェデレーテッドラーニングのジョブを実行するコマンドを示します。このコマンドをターミナルで実行してください。

.. note::

    モデルは重み ``[[1, 2, 3], [4, 5, 6], [7, 8, 9]]`` から始まり、各クライアントは学習中に各重みに1を加えます。
    集約後、ラウンドごとに重みが1ずつ増えていくのが確認でき、フェデレーテッドラーニングのプロセスが実感できます。



FLジョブを実行するコマンド
---------------------------

ターミナルで次のコマンドを使い、指定したラウンド数とクライアント数でジョブを開始します。

.. code-block:: text

   python job.py --num_rounds 3 --n_clients 2

この演習の完全なソースコードは
:github_nvflare_link:`examples/hello-world/hello-numpy <examples/hello-world/hello-numpy/>` にあります。

この例の過去バージョン
------------------------

以前のバージョンの NVIDIA FLARE を使用しているユーザー向けに、この例は異なる名前で提供されていました。

**"Hello Scatter and Gather" (バージョン 2.0-2.4):**

   - `hello-numpy-sag for 2.0 <https://github.com/NVIDIA/NVFlare/tree/2.0/examples/hello-numpy-sag>`_
   - `hello-numpy-sag for 2.1 <https://github.com/NVIDIA/NVFlare/tree/2.1/examples/hello-numpy-sag>`_
   - `hello-numpy-sag for 2.2 <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/hello-numpy-sag>`_
   - `hello-numpy-sag for 2.3 <https://github.com/NVIDIA/NVFlare/tree/2.3/examples/hello-world/hello-numpy-sag>`_
   - `hello-numpy-sag for 2.4 <https://github.com/NVIDIA/NVFlare/tree/2.4/examples/hello-world/hello-numpy-sag>`_

**"Hello FedAvg NumPy" (バージョン 2.5-2.6):**

   - `hello-fedavg-numpy for 2.5 <https://github.com/NVIDIA/NVFlare/tree/2.5/examples/hello-world/hello-fedavg-numpy>`_
   - `hello-fedavg-numpy for 2.6 <https://github.com/NVIDIA/NVFlare/tree/2.6/examples/hello-world/hello-fedavg-numpy>`_
