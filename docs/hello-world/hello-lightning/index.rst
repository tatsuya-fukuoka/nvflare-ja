
Hello Pytorch Lightning
=======================

この例では、NVIDIA FLARE と PyTorch Lightning を組み合わせて、連合平均(FedAvg)または SCAFFOLD を用いて
画像分類器をトレーニングする方法を示します。完全なサンプルコードは
:github_nvflare_link:`hello-lightning ディレクトリ <examples/hello-world/hello-lightning>` にあります。

.. note::

   Lightning の自動 SCAFFOLD サポートは NVFlare 2.9.0 で導入されます。そのパッケージが公開されるまでは、
   このリポジトリから NVFlare をインストールし、残りの例の依存関係を個別にインストールしてください。

仮想環境を作成し、すべてを virtualenv 内で実行することを推奨します。


NVIDIA FLAREのインストール
------------------------------------------------

完全なインストール手順については、:doc:`Installation </installation>` を参照してください。リリース済みブランチの場合:

.. code-block:: text

    pip install nvflare

現在の ``main`` ブランチの場合は、自動 SCAFFOLD サポートを利用できるように、リポジトリのルートからインストールしてください:

.. code-block:: text

    python -m pip install -e .
    python -m pip install torch torchvision "jsonargparse[signatures]>=4.17.0" pytorch_lightning tensorboard


``requirements.txt`` の ``nvflare~=2.9.0rc`` エントリは、最初の互換リリースを意図的に記録しています。
NVFlare 2.9.0 の公開後は、次のコマンドで完全な環境をインストールできます:

.. code-block:: bash

   python -m pip install -r requirements.txt

GitHubからサンプルコードを取得します:

.. code-block:: text

   git clone https://github.com/NVIDIA/NVFlare.git

次に hello-lightning ディレクトリに移動します:

.. code-block:: text

    git switch <release branch>
    cd examples/hello-world/hello-lightning

コード構造
--------------------

.. code-block:: text

    .
 hello-lightning
    |
    |-- client.py        # client local training script
    |-- model.py         # model definition
    |-- job.py              # job recipe that defines client and server configurations
    |-- requirements.txt    # dependencies

データ
------------
この例では `CIFAR-10 <https://www.cs.toronto.edu/~kriz/cifar.html>`_ データセットを使用します。

実際のFL実験では、各クライアントはローカルトレーニングに使用する独自のデータセットを持ちます。
CIFAR-10 データセットは、torchvision の datasets モジュールを介してインターネットからダウンロードできます。
各クライアントが独自のデータセットを持つように、データセットをクライアントごとに分割することもできます。
ここでは簡単のため、各クライアントで同じデータセットを使用します。

PyTorch のデータモジュールはデータセットを直接ダウンロードできます。すべてのサイトが同じデータセットを
ダウンロードするため、データの準備が完了する前にトレーニングが始まってしまい、エラーにつながるケースがあります。
トレーニングを開始する前に、ターミナルのコマンドラインから以下を実行して、事前にデータをダウンロードしておくことができます

.. code-block:: text

    ./prepare_data.sh


.. literalinclude:: ../../../examples/hello-world/hello-lightning/prepare_data.sh
    :language: bash
    :linenos:
    :caption: prepare_data.sh

PyTorch Lightning において、`LightningDataModule` はデータの読み込みと処理を扱うための標準化された方法です。トレーニング、検証、テスト用のデータを準備するために必要なすべての手順をカプセル化し、データセットとデータローダーをクリーンで整理された形で管理しやすくします。この抽象化により、データ関連のロジックがモデルやトレーニングコードから分離され、コードの構成と再利用性が向上します。

`LightningDataModule`
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

- **目的:** `LightningDataModule` は、データセットのダウンロード、変換、分割や、トレーニング・検証・テスト・予測用のデータローダーの提供など、データ関連のすべての操作をカプセル化するように設計されています。

- **主要メソッド:**
  - `prepare_data()`: データのダウンロードと準備に使用されます。このメソッドは一度だけ呼び出され、複数のGPUやノードに分散されません。
  - `setup(stage)`: さまざまなステージ('fit'、'validate'、'test'、'predict' など)用のデータセットをセットアップするために使用されます。このメソッドはすべてのGPUまたはノードで呼び出されます。
  - `train_dataloader()`、`val_dataloader()`、`test_dataloader()`、`predict_dataloader()`: これらのメソッドは、各ステージに対応するデータローダーを返します。

`DataModule` のセットアップ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

`CIFAR10DataModule` では、以下を実装しています:

- **初期化 (`__init__`):** コンストラクタは、データモジュール全体で使用されるデータディレクトリとバッチサイズを初期化します。

- **データ準備 (`prepare_data`):** このメソッドは、指定されたディレクトリに CIFAR-10 データセットがまだ存在しない場合にダウンロードします。トレーニングデータセットとテストデータセットの両方を準備します。

- **セットアップ (`setup`):** このメソッドは、ステージごとにデータセットを割り当てます:
  - 'fit' および 'validate' ステージでは、CIFAR-10 トレーニングデータセットをトレーニングセットと検証セットに分割します。
  - 'test' および 'predict' ステージでは、テストデータセットを割り当てます。

- **データローダー:** このモジュールは、トレーニング、検証、テスト、予測用のデータローダーを提供し、それぞれ指定されたバッチサイズで構成されます。

`LightningDataModule` を使用することで、データ処理ロジックがきれいにカプセル化され、トレーニングコードの他の部分に影響を与えることなく、データ関連の操作を管理・変更しやすくなります。

.. literalinclude:: ../../../examples/hello-world/hello-lightning/client.py
    :language: python
    :linenos:
    :caption: data module
    :lines: 14-70


モデル
------------
PyTorch Lightning において、`LightningModule` は PyTorch の上に構築された
高レベルの抽象化であり、モデルのトレーニングプロセスを効率化します。
モデルアーキテクチャ、トレーニング、検証、テストのロジックをカプセル化し、
PyTorch に通常付随するボイラープレートコードに煩わされることなく、
開発者がモデルの中核部分に集中できるようにします。

`LightningModule` の概要

- **モデル定義:** `LightningModule` は、PyTorch の `nn.Module` を使用して
  定義されたモデルアーキテクチャで初期化されます。これには、レイヤー、
  活性化関数、およびモデルに必要なその他のコンポーネントが含まれます。

- **順伝播(フォワードパス):** `forward` メソッドは、入力データがモデルを
  どのように流れるかを指定します。ここでモデルの中核となる計算が定義されます。

- **トレーニングロジック:** `training_step` メソッドは、1回のトレーニング
  イテレーションのロジックを含みます。損失や、精度など追跡したいメトリクスを
  計算します。このメソッドはトレーニングループ中に自動的に呼び出されます。

- **検証とテスト:** トレーニングステップと同様に、`validation_step` および
  `test_step` メソッドは、モデルがそれぞれ検証データセットとテストデータセットで
  どのように評価されるかを定義します。これらのメソッドは、モデルの性能と
  汎化性能のモニタリングに役立ちます。

- **オプティマイザーの構成:** `configure_optimizers` メソッドは、トレーニング中に
  使用されるオプティマイザーと学習率スケジューラーを指定します。これにより、
  柔軟でカスタマイズ可能なトレーニング戦略が可能になります。

`LightningModule` を使用することで、開発者は分散トレーニング、自動
チェックポイント、ロギングといった PyTorch Lightning の機能を活用でき、
実験のスケールや複雑なトレーニングワークフローの管理が容易になります。
この抽象化により、よりクリーンなコード、より良い構成、より容易なデバッグが
促進され、最終的にモデル開発プロセスが加速されます。

.. literalinclude:: ../../../examples/hello-world/hello-lightning/model.py
    :language: python
    :linenos:
    :caption: model.py
    :lines: 14-

--------------


クライアントコード
------------------------------------

トレーニングコードが PyTorch Lightning の標準的なトレーニングコードとほぼ同一であることに注目してください。
唯一の違いは、サーバーとデータを送受信するための数行を追加した点です。
理解しやすいように、変更したコードにはすべて0から4の番号を付けています。


.. literalinclude:: ../../../examples/hello-world/hello-lightning/client.py
    :language: python
    :linenos:
    :caption: client.py
    :lines: 71-


`client.py` ファイルのコードロジックの主な流れは、PyTorch Lightning と NVFlare を使用して、各クライアント上でローカルに連合学習(FL)トレーニングロジックを実行することです。
主要なステップの内訳は次のとおりです:

1. **引数の解析:**

   - `define_parser()` 関数は、コマンドライン引数、特にデータ読み込みのバッチサイズを設定する `--batch_size` 引数を解析するために使用されます。

2. **初期化:**

   - `main()` 関数は、まずコマンドライン引数を解析してバッチサイズを取得します。
   - `flare.init()` 関数が呼び出されて NVFlare クライアントが初期化されます。これは `flare.get_site_name()` などの特定の NVFlare 関数を使用するために必要です。

3. **モデルとデータモジュールのセットアップ:**

   - PyTorch Lightning モデルである `LitNet` のインスタンスが作成されます。
   - データの読み込みと処理を扱うために、指定されたバッチサイズで `CIFAR10DataModule` のインスタンスが作成されます。

4. **Trainerの構成:**

   - PyTorch Lightning の `Trainer` が構成されます。GPUが利用可能な場合はGPUを使用するように設定され、そうでない場合はCPUがデフォルトになります。

5. **NVFlare統合:**

   - `flare.patch(trainer)` 関数が呼び出され、NVFlare が PyTorch Lightning のトレーナーと統合されます。これにより、トレーナーは連合学習のタスクを処理できるようになります。
   - ``ScaffoldRecipe`` が SCAFFOLD コントロールを送信すると、このパッチは必要な
     ``PTScaffoldHelper`` の更新を自動的に適用し、コントロールの差分を返します。このパスには、
     1つのオプティマイザーによる Lightning の自動最適化が必要です。

6. **連合学習ループ:**

   - `flare.is_running()` が `True` を返す間、つまり連合学習ジョブがアクティブな間、ループが実行されます。
   - ループ内では:
      - `flare.receive()` を使用して、NVFlare サーバーからグローバルモデルを受信します。
      - ログ出力のために、現在のラウンドとサイト名が出力されます。
      - `trainer.validate()` を使用して、グローバルモデルを検証します。
      - 受信したグローバルモデルを起点として、`trainer.fit()` を使用してローカルトレーニングを実行します。
      - `trainer.test()` を使用して、ローカルモデルをテストします。
      - `trainer.predict()` を使用して、予測を行います。

7. **実行:**

   - スクリプトがメインモジュールとして実行された場合に `main()` 関数が実行され、プロセス全体が開始されます。


サーバーコード
----------------------------
連合平均では、サーバーコードはクライアントからのモデル更新の集約を担当し、
そのワークフローパターンは scatter-gather に似ています。
この例では、NVFlare が提供するデフォルトの連合平均アルゴリズムを直接使用します。
FedAvg クラスは `nvflare.app_common.workflows.fedavg.FedAvg` で定義されています。
この例では、カスタマイズしたサーバーコードを定義する必要はありません。


ジョブレシピコード
------------------------------------
ジョブレシピコードは、クライアントとサーバーの構成を定義するために使用されます。

.. literalinclude:: ../../../examples/hello-world/hello-lightning/job.py
    :language: python
    :linenos:
    :caption: Job Recipe (job.py)
    :lines: 14-

モデル入力オプション
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

``model`` パラメータは2つの形式を受け付けます:

1. **クラスインスタンス**: ``model=LitNet()`` - 便利でPythonic
2. **辞書設定**: ``model={"class_path": "model.LitNet", "args": {}}`` - 大規模モデルに適しています

事前学習済みの重みから再開するには:

.. code-block:: python

   recipe = FedAvgRecipe(
       model=LitNet(),
       initial_ckpt="/server/path/to/pretrained.pt",  # Absolute path
       ...
   )


FLジョブの実行
----------------------------

このセクションでは、上で定義したジョブレシピを使用して連合学習ジョブを
実行するためのコマンドを示します。ターミナルでこのコマンドを実行してください。
まず、次のコマンドを実行してデータをダウンロードします:

.. code-block:: text

  ./prepare_data.sh


**FLジョブを実行するコマンド**

指定したラウンド数、バッチサイズ、クライアント数でジョブを開始するには、
ターミナルで次のコマンドを使用します。


.. code-block:: text

  python job.py --num_rounds 2 --batch_size 16

FedAvg がデフォルトです。同じクライアントは、トレーニングループを変更することなく SCAFFOLD を実行できます:

.. code-block:: text

  python job.py --algorithm scaffold --num_rounds 2 --batch_size 16

Lightning の手動最適化を使用する場合は、``flare.patch(trainer)`` を使わずに明示的な receive/train/send ループを使用し、
``PTScaffoldHelper`` を直接統合してください。

自動パスは、``precision="32-true"`` または ``precision="bf16-mixed"`` を指定した1つのオプティマイザーと、
すべてのステップにおいてパラメータグループ間で等しい有限かつ非負の学習率をサポートします。NVFlare 2.9.0 以降、
PyTorch の SCAFFOLD コントロール差分にはトレーニング可能なパラメータのみが含まれ、BatchNorm の移動統計量などの
バッファは通常のモデル状態のままです。カスタムの SCAFFOLD アグリゲーターは、疎なコントロール辞書を受け付ける
必要があります。トレーニング可能性はラウンド間で変化する可能性があり、その場合、新たにトレーニング可能になった
ローカルコントロールはゼロにリセットされますが、``requires_grad`` はラウンド中に変化してはなりません。


出力

.. code-block:: text


.. code-block:: Python
   :dedent: 1

        # < ... skip few lines of logs ..>
        # 2025-07-22 18:45:45,758 - INFO - Start FedAvg.
        # 2025-07-22 18:45:45,759 - INFO - loading initial model from persistor
        # 2025-07-22 18:45:45,759 - INFO - Both source_ckpt_file_full_name and ckpt_preload_path are not provided. Using the default model weights initialized on the persistor side.
        # 2025-07-22 18:45:45,760 - INFO - Round 0 started.
        # 2025-07-22 18:45:45,760 - INFO - Sampled clients: ['site-1', 'site-2']
        # 2025-07-22 18:45:45,760 - INFO - Sending task train to ['site-1', 'site-2']
        #
        # < ... skip .. few lines of logs ..>
        #
        # 2025-07-22 18:45:50,507 - INFO - batch_size=16, site=site-1
        # 2025-07-22 18:45:50,543 - INFO -
        # [Current Round=0, Site = site-1]
        #
        # 2025-07-22 18:45:50,543 - INFO - --- validate global model ---
        # 2025-07-22 18:45:50,578 - INFO - batch_size=16, site=site-2
        # 2025-07-22 18:45:50,656 - INFO -
        # [Current Round=0, Site = site-2]
        #
        # 2025-07-22 18:45:50,656 - INFO - --- validate global model ---
        #
        # < ... skip .. few lines of logs ..>
        #
        # ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
        # ┃        Test metric        ┃       DataLoader 0        ┃
        # ┡━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
        # │      test_acc_epoch       │    0.44699999690055847    │
        # │         test_loss         │    1.5125484466552734     │
        # └───────────────────────────┴───────────────────────────┘
        # Testing DataLoader 0:  68%|████████████████████████████████▍               | 422/625 [00:01<00:00, 276.33it/s]2025-07-22 18:46:39,629 - INFO - --- prediction with new best model ---
        # Testing DataLoader 0:  76%|████████████████████████████████████▋           | 478/625 [00:01<00:00, 275.61it/s]2025-07-22 18:46:39,837 - INFO - Files already downloaded and verified
        # Testing DataLoader 0: 100%|████████████████████████████████████████████████| 625/625 [00:02<00:00, 275.79it/s]
        # ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
        # ┃        Test metric        ┃       DataLoader 0        ┃
        # ┡━━━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━┩
        # │      test_acc_epoch       │    0.44699999690055847    │
        # │         test_loss         │    1.5125484466552734     │
        # └───────────────────────────┴───────────────────────────┘
        # 2025-07-22 18:46:40,370 - INFO - --- prediction with new best model ---
        # 2025-07-22 18:46:40,431 - INFO - Files already downloaded and verified
        # 2025-07-22 18:46:40,577 - INFO - Files already downloaded and verified
        # Predicting DataLoader 0:  16%|███████▍                                     | 103/625 [00:00<00:01, 371.90it/s]2025-07-22 18:46:41,191 - INFO - Files already downloaded and verified
        # Predicting DataLoader 0: 100%|█████████████████████████████████████████████| 625/625 [00:01<00:00, 367.54it/s]
        # Predicting DataLoader 0:  53%|███████████████████████▊                     | 331/625 [00:00<00:00, 346.29it/s]2025-07-22 18:46:42,615 - WARNING - request to stop the job for reason END_RUN received
        # Predicting DataLoader 0: 100%|█████████████████████████████████████████████| 625/625 [00:01<00:00, 344.12it/s]
        # 2025-07-22 18:46:43,476 - WARNING - request to stop the job for reason END_RUN received
