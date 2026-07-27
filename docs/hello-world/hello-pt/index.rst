Hello PyTorch
=============

この例では、NVIDIA FLARE を PyTorch と組み合わせて、フェデレーテッドアベレージング（FedAvg）により画像分類器を学習する方法を示します。完全なサンプルコードは `hello-pt directory <examples/hello-world/hello-pt/>` にあります。仮想環境を作成し、その中ですべてを実行することを推奨します。

NVFLARE と依存関係のインストール
------------------------------------------

インストール手順の詳細については `Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: text

    pip install nvflare

まず GitHub からサンプルコードを取得します。

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git

次に hello-pt ディレクトリに移動します。

.. code-block:: bash

   git switch <release branch>
   cd examples/hello-world/hello-pt


依存関係をインストールします。

.. code-block:: text

    pip install -r requirements.txt



コード構造
--------------

.. code-block:: bash

   hello-pt
   |
   |-- client.py             # client local training script
   |-- model.py              # model definition
   |-- job.py                # job recipe that defines client and server configurations
   |-- requirements.txt      # dependencies

NVIDIA FLARE のインストール
------------------------------------

ここでは PT 拡張付きの nvflare をインストールします。インストール手順の詳細については `Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: bash

   pip install nvflare[PT]

すべての依存関係をインストールします。

.. code-block:: bash

   pip install -r requirements.txt

データ
--------

この例では `CIFAR-10 <https://www.cs.toronto.edu/~kriz/cifar.html>`_ データセットを使用します。CIFAR10 データセットは torchvision の datasets モジュールを介してインターネットからダウンロードできます。

実際の FL 実験では、各クライアントがローカル学習に使用する独自のデータセットを持ちます。
各クライアントが自身のデータセットを持つように、データセットをクライアントごとに分割することもできます。
ここでは簡単のため、各クライアントで同じデータセットを使用します。

モデル
--------

PyTorch では、``nn.Module`` を継承したクラス（例: ``SimpleNetwork``）を定義することでニューラルネットワークを実装します。
ネットワークのアーキテクチャは __init__ メソッドで構築し、forward メソッドで入力データがレイヤをどのように流れるかを
決定します。計算を高速化するため、利用可能であればモデルはハードウェアアクセラレータ（NVIDIA GPU など）に転送され、そうでなければ CPU 上で実行されます。このモデルの実装は :github_nvflare_link:`model.py <examples/hello-world/hello-pt/model.py>` にあります。

.. code-block:: python

   import torch
   import torch.nn as nn
   import torch.nn.functional as F

   class SimpleNetwork(nn.Module):
       def __init__(self):
           super(SimpleNetwork, self).__init__()
           self.conv1 = nn.Conv2d(3, 6, 5)
           self.pool = nn.MaxPool2d(2, 2)
           self.conv2 = nn.Conv2d(6, 16, 5)
           self.fc1 = nn.Linear(16 * 5 * 5, 120)
           self.fc2 = nn.Linear(120, 84)
           self.fc3 = nn.Linear(84, 10)

       def forward(self, x):
           x = self.pool(F.relu(self.conv1(x)))
           x = self.pool(F.relu(self.conv2(x)))
           x = torch.flatten(x, 1)  # flatten all dimensions except batch
           x = F.relu(self.fc1(x))
           x = F.relu(self.fc2(x))
           x = self.fc3(x)
           return x

クライアントコード
--------------------

クライアント側の学習ワークフローは次のとおりです。

1. FL サーバからモデルを受信します。
2. 受信したグローバルモデルに対してローカル学習を行う、または／およびモデル選択のために受信したグローバルモデルを評価します。
3. 新しいモデルを FL サーバに送り返します。

クライアントコード（:github_nvflare_link:`client.py <examples/hello-world/hello-pt/client.py>`）は、この学習ワークフローの実装を担当します。学習コードが標準的な PyTorch の学習コードとほぼ同じである点に注目してください。
唯一の違いは、サーバとの間でデータを受信・送信するための数行を追加していることです。

NVFlare の Client API を使用すると、集中学習向けに書かれた機械学習コードを容易に適応させ、フェデレーテッドのシナリオに適用できます。
一般的なユースケースでは、Client API を使ってこれを実現するために必要なメソッドは 3 つです。

- ``init()``: NVFlare Client API 環境を初期化します。
- ``receive()``: FL サーバからモデルを受信します。
- ``send()``: FL サーバにモデルを送信します。

これらのシンプルなメソッドにより、開発者は Client API を使用して、
以下に示すように 5 行のコード変更で集中学習のコードを
FL のシナリオへ変更できます。

.. code-block:: python

   import nvflare.client as flare

   flare.init() # 1. Initializes NVFlare Client API environment.
   input_model = flare.receive() # 2. Receives model from the FL server.
   params = input_model.params # 3. Obtain the required information from the received model.

   # original local training code
   new_params = local_train(params)

   output_model = flare.FLModel(params=new_params) # 4. Put the results in a new `FLModel`
   flare.send(output_model) # 5. Sends the model to the FL server.

サーバコード
--------------

フェデレーテッドアベレージングでは、サーバコードはグローバルモデルの配布と、クライアントからのモデル更新の集約を担当します。

まず、NVFlare による `FedAvg <https://proceedings.mlr.press/v54/mcmahan17a?ref=https://githubhelp.com>`_ アルゴリズムの堅牢な実装を提供します。

サーバは次の主要なステップを実行します。

1. FL サーバが初期モデルを初期化します。
2. 各ラウンド（グローバルイテレーション）で次を行います。
   - FL サーバが利用可能なクライアントをサンプリングします。
   - FL サーバがグローバルモデルをクライアントに送信し、その更新を待ちます。
   - FL サーバがすべての ``results`` を集約し、新しいグローバルモデルを生成します。

この例では、PyTorch 向けの `FedAvgRecipe <https://nvflare.readthedocs.io/en/main/apidocs/nvflare.app_opt.pt.recipes.fedavg.html#nvflare.app_opt.pt.recipes.fedavg.FedAvgRecipe>`_ を利用し、NVFlare が提供するデフォルトのフェデレーテッドアベレージングアルゴリズムをそのまま使用します。

この例では、カスタマイズしたサーバコードを定義する必要はありません。

Job Recipe のコード
-----------------------

Job Recipe は ``client.py`` を指定し、組み込みのフェデレーテッドアベレージングアルゴリズムを選択します。

.. code-block:: python

   recipe = FedAvgRecipe(
       name="hello-pt",
       min_clients=n_clients,
       num_rounds=num_rounds,
       # Model can be specified as class instance or dict config:
       model=SimpleNetwork(),
       # Alternative: model={"class_path": "model.SimpleNetwork", "args": {}},
       # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt",
       train_script="client.py",
       train_args=f"--batch_size {batch_size}",
   )

   env = SimEnv(num_clients=n_clients, num_threads=n_clients)
   recipe.execute(env=env)

モデル入力のオプション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``model`` パラメータは 2 つの形式を受け付けます。

1. **クラスインスタンス**（上記の例）: ``model=SimpleNetwork()`` - 手軽で Python らしい書き方です
2. **dict 設定**: ``model={"class_path": "model.SimpleNetwork", "args": {}}`` - 大きなモデルに適しています

事前学習済みの重みから学習を再開するには ``initial_ckpt`` を使用します。

.. code-block:: python

   recipe = FedAvgRecipe(
       model=SimpleNetwork(),
       initial_ckpt="/server/path/to/pretrained.pt",  # Absolute path, must exist on server
       ...
   )

.. note::

   クラスインスタンスは、ジョブ送信前に設定ファイルへ変換されます。大きなモデルの場合は、不要なインスタンス化のオーバーヘッドを避けるために dict 設定を使用してください。

ジョブの実行
--------------

ターミナルから job スクリプトを実行するだけで、シミュレーション環境でジョブを実行できます。

.. code-block:: bash

   python job.py

.. note::
   job スクリプトの一部として ``add_experiment_tracking(recipe, tracking_type="tensorboard")`` を使用すると、:github_nvflare_link:`client.py <examples/hello-world/hello-pt/client.py>` 内で NVIDIA FLARE の `SummaryWriter <https://nvflare.readthedocs.io/en/main/apidocs/nvflare.client.tracking.html#nvflare.client.tracking.SummaryWriter>`_ を用いて学習メトリクスをサーバへストリーミングできます。

ノートブック
--------------

この例のインタラクティブ版については、Google Colab で実行できる :github_nvflare_link:`notebook <examples/hello-world/hello-pt/hello-pt.ipynb>` を参照してください。

出力の概要
--------------

初期化
~~~~~~~~~~~~~~~

- **TensorBoard**: ログは /tmp/nvflare/simulation/hello-pt/server/simulate_job/tb_events で確認できます。
- **ワークフロー**: BaseModelController が初期化されます。

ラウンド 0
~~~~~~~~~~~~

- **モデルの読み込み**: persistor から初期モデルが読み込まれます。
- **サンプリングされたクライアント**: site-1、site-2。
- **学習**:
  - 両方のサイトにタスクが送信されます。
  - 2 エポックが完了し、損失が報告されます。
- **集約**: モデルが集約され、サーバ上に永続化されます。

ラウンド 1
~~~~~~~~~~~~

- **サンプリングされたクライアント**: site-1、site-2。
- **学習**:
  - ラウンド 0 と同様のプロセスです。
  - **集約**: モデルが集約され、永続化されます。

完了
~~~~~~~~

- **FedAvg プロセス**: 最終モデルが永続化され、正常に完了します。
