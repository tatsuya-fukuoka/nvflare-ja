Hello Differential Privacy
===========================

この例では、NVIDIA FLARE を PyTorch および **差分プライバシー（Differential Privacy, DP）** と組み合わせて、プライバシー保証を伴うフェデレーテッドアベレージング（FedAvg）により不正検知モデルを学習する方法を示します。この例では `Opacus <https://opacus.ai>`_ を使用して、各クライアントでのローカル学習中に DP-SGD（差分プライベート確率的勾配降下法）を実装します。これによりサンプルレベルの差分プライバシーを実現します。完全なサンプルコードは `hello-dp directory <examples/hello-world/hello-dp/>`_ にあります。仮想環境を作成し、その中ですべてを実行することを推奨します。

差分プライバシーとは？
------------------------------

`Differential Privacy (DP) <https://en.wikipedia.org/wiki/Differential_privacy>`_ は、機微なデータを扱う際に強力なプライバシー保証を提供する数学的なフレームワークです。フェデレーテッドラーニングにおいて、DP はモデルの学習過程に慎重に調整されたノイズを加えることでユーザー情報を保護します。

**DP-SGD** は各最適化ステップでノイズを加えます。

1. **勾配クリッピング**: 感度を抑えるために勾配をクリッピングします
2. **ノイズの付加**: クリッピングされた勾配にガウスノイズを加えます
3. **プライバシー会計**: プライバシー予算 (ε, δ) を追跡します

プライバシーと有用性のトレードオフはイプシロン (ε) によって制御されます。

- **ε が小さい** = プライバシーがより強力、ノイズがより多い、精度がより低い
- **ε が大きい** = プライバシーがより弱い、ノイズがより少ない、精度がより高い

代表的な値は次のとおりです。

- **ε ≤ 1.0**: 強力なプライバシー（機微なデータに推奨）
- **ε = 1.0-3.0**: 中程度のプライバシー（バランスが良い） - デフォルトは 1.0
- **ε > 10**: 弱いプライバシー（保護は最小限）

NVIDIA FLARE のインストール
------------------------------------

インストール手順の詳細については `Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください。

.. code-block:: bash

   pip install nvflare

まず GitHub からサンプルコードを取得します。

.. code-block:: bash

   git clone https://github.com/NVIDIA/NVFlare.git

次に hello-dp ディレクトリに移動します。

.. code-block:: bash

   git switch <release branch>
   cd examples/hello-world/hello-dp

依存関係をインストールします。

.. code-block:: bash

   pip install -r requirements.txt

コード構造
--------------

.. code-block:: bash

   hello-dp
   |
   |-- client.py             # client training script with DP-SGD using Opacus
   |-- model.py              # MLP model definition for tabular data
   |-- job.py                # job recipe that defines client and server configurations
   |-- requirements.txt      # dependencies

データ
--------

この例では、OpenML の `Credit Card Fraud Detection dataset <https://www.openml.org/d/1597>`_ を使用します。これは不正なクレジットカード取引を検出する二値分類問題です。

**データセットの特徴:**

- 約 284,000 サンプル（正常: 284,315、不正: 492）
- 29 個の特徴量（匿名化された取引特徴量 V1-V28、Amount）
- 2 クラス: 正常 (0) と不正 (1)
- **非常に不均衡**: 約 99.8% が正常、約 0.17% が不正

**重要な注意**: このデータセットは 284,807 件の取引のうち不正がわずか 492 件しかなく、極めて不均衡です。これは学習において次のような追加の課題をもたらします。

- 標準的な accuracy は誤解を招く可能性があります（常に「正常」と予測するだけで 99.8% の accuracy になります）
- 不正検知においては **F1 スコアおよび Precision/Recall** の方がより意味のある指標です
- モデルは不均衡にもかかわらず、まれな不正クラスを検出できるように学習する必要があります

これは **プライバシーに配慮が必要な** ユースケースです。クレジットカード取引データは強力なプライバシー保護を必要とするため、フェデレーテッドラーニングにおける差分プライバシーを示す題材として最適です。

**データ分布**: 実際の FL 実験では、各クライアントが独自のデータセットを持ちます。この例では、データセットは単純な分割によって **クライアント間で自動的に分割** され、各クライアントは **重複しないサブセット** を持ちます。これは、データが複数の機関に分散している基本的なフェデレーションのシナリオをシミュレートしています。

モデル
--------

モデルは二値分類のためのシンプルな多層パーセプトロン（MLP）です。実装は `model.py <model.py>`_ にあります。

.. code-block:: python

   import torch.nn as nn

   class TabularMLP(nn.Module):
       """Simple Multi-Layer Perceptron for tabular data classification"""

       def __init__(self, input_dim=29, hidden_dims=[64, 32], output_dim=2):
           super(TabularMLP, self).__init__()

           layers = []
           prev_dim = input_dim

           # Build hidden layers
           for hidden_dim in hidden_dims:
               layers.append(nn.Linear(prev_dim, hidden_dim))
               layers.append(nn.ReLU())
               layers.append(nn.Dropout(0.2))
               prev_dim = hidden_dim

           # Output layer
           layers.append(nn.Linear(prev_dim, output_dim))

           self.model = nn.Sequential(*layers)

アーキテクチャは次のとおりです。

- **入力層**: 29 個の特徴量（取引データ）
- **隠れ層**: ReLU 活性化関数とドロップアウトを伴う 64 → 32 ニューロン
- **出力層**: 2 ニューロン（正常 vs 不正）

差分プライバシーを用いたクライアントコード
--------------------------------------------------------

クライアントコード `client.py <client.py>`_ は **Opacus** を使用して DP-SGD を実装します。標準的な学習との主な違いは ``PrivacyEngine`` を追加する点です。

.. code-block:: python

   from opacus import PrivacyEngine
   import nvflare.client as flare

   # Initialize NVFlare client
   flare.init()

   # Initialize privacy engine once (in first round only)
   privacy_engine = None

   while flare.is_running():
       input_model = flare.receive()
       model.load_state_dict(input_model.params)

       # === Apply Differential Privacy (First Round Only) ===
       # Privacy budget accumulates across ALL federated rounds
       if input_model.current_round == 0:
           # Calculate total epochs across all rounds for privacy accounting
           total_epochs = args.epochs * input_model.total_rounds

           privacy_engine = PrivacyEngine()
           model, optimizer, train_loader = privacy_engine.make_private_with_epsilon(
               module=model,
               optimizer=optimizer,
               data_loader=train_loader,
               epochs=total_epochs,                    # Total across ALL rounds
               target_epsilon=args.target_epsilon,     # Target privacy budget
               target_delta=args.target_delta,         # Failure probability
               max_grad_norm=args.max_grad_norm,       # Gradient clipping
           )
           # Noise multiplier is computed automatically
           print(f"Noise multiplier: {optimizer.noise_multiplier:.4f}")
       # ==================================

       # Train as usual - PrivacyEngine handles gradient clipping & noise
       for epoch in range(args.epochs):
           for data, target in train_loader:
               optimizer.zero_grad()
               loss = criterion(model(data), target)
               loss.backward()
               optimizer.step()

       # Check cumulative privacy budget spent
       epsilon = privacy_engine.get_epsilon(args.target_delta)
       print(f"Cumulative privacy spent: (ε = {epsilon:.2f}, δ = {args.target_delta})")

``PrivacyEngine.make_private_with_epsilon()`` メソッドは次を行います。

1. サンプルごとの勾配計算を有効にするためにモデルをラップします
2. 目標のイプシロンに対するノイズ乗数を自動的に計算します
3. 勾配をクリッピングしノイズを加えるようにオプティマイザを変更します
4. プライバシー会計のためにデータローダーをラップします
5. すべてのフェデレーテッドラウンドにわたってプライバシー予算を累積的に追跡します

サーバ側のワークフロー
------------------------------

この例では `FedAvg <https://proceedings.mlr.press/v54/mcmahan17a>`_ アルゴリズムを実装した `FedAvgRecipe <https://nvflare.readthedocs.io/en/main/apidocs/nvflare.app_opt.pt.recipes.fedavg.html>`_ を使用します。Recipe API がサーバ側のロジックをすべて自動的に処理します。

1. グローバルモデルを初期化します
2. 各学習ラウンドで次を行います。

   - 利用可能なクライアントをサンプリングします
   - 選択されたクライアントにグローバルモデルを送信します
   - クライアントからの更新を待ちます
   - クライアントのモデルを集約して新しいグローバルモデルを生成します

Recipe API を使用すると、**カスタムのサーバコードを書く必要はありません**。フェデレーテッドアベレージングのワークフローは NVFlare によって提供されます。

Job Recipe のコード
-----------------------

``FedAvgRecipe`` はクライアントの学習スクリプトと DP パラメータを組み合わせます。

.. code-block:: python

   from nvflare.apis.dxo import DataKind

   recipe = FedAvgRecipe(
       name="hello-dp",
       min_clients=n_clients,
       num_rounds=num_rounds,
       # Model can be class instance or dict config
       # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
       model=TabularMLP(input_dim=29, hidden_dims=[64, 32], output_dim=2),
       train_script="client.py",
       train_args=f"--batch_size {batch_size} --target_epsilon {target_epsilon} --n_clients {n_clients}",
       aggregator_data_kind=DataKind.WEIGHT_DIFF,
   )

   env = SimEnv(num_clients=n_clients)
   recipe.execute(env=env)

DP-SGD は各クライアントのローカル学習を保護します。``client.py`` では、クライアントが明示的に
ローカルパラメータからグローバルパラメータを引いた値を計算し、``FLModel(params_type=ParamsType.DIFF)`` を返します。この
``params_type`` はクライアント結果を規定する正式な記述です。recipe は構築時に任意の学習スクリプトから
結果の種類を推論することはできません。recipe は、カスタムアグリゲータが宣言する ``expected_data_kind`` を含め、
自身が管理するサーバ側の設定を検証します。

**重要**: プライバシー予算 (ε) はすべてのフェデレーテッドラウンドにわたって累積されます。``target_epsilon`` パラメータは、ラウンドごとではなく学習プロセス全体に対する合計のプライバシー予算を指定します。

ジョブの実行
--------------

ターミナルから job スクリプトを実行するだけで、シミュレーション環境でジョブを実行できます。

.. code-block:: bash

   python job.py

パラメータをカスタマイズするには次のようにします。

.. code-block:: bash

   python job.py --n_clients 2 --num_rounds 10 --target_epsilon 1.0

パラメータ:

- ``--n_clients``: フェデレーテッドクライアントの数（デフォルト: 2）
- ``--num_rounds``: フェデレーテッドラウンドの数（デフォルト: 10）
- ``--batch_size``: 学習のバッチサイズ（デフォルト: 64）
- ``--target_epsilon``: 全ラウンドにわたる **合計** プライバシー予算 - **小さいほどプライバシーが強力**（デフォルト: 1.0）

.. note::
   job スクリプトの一部として ``add_experiment_tracking(recipe, tracking_type="tensorboard")`` を使用すると、`client.py <client.py>`_ 内で NVIDIA FLARE の `SummaryWriter <https://nvflare.readthedocs.io/en/main/apidocs/nvflare.client.tracking.html#nvflare.client.tracking.SummaryWriter>`_ を用いて学習メトリクスをサーバへストリーミングできます。

結果の可視化
-----------------

TensorBoard で学習メトリクスとプライバシー予算を確認します。

.. code-block:: bash

   tensorboard --logdir /tmp/nvflare/simulation/hello-dp

http\://localhost:6006 を開くと、次の内容を確認できます。

- 時間経過に伴う学習損失
- **Accuracy** と **F1 スコア**（不正検知のメトリクス）
- クライアントごとに消費されたプライバシーイプシロン

プライバシーと有用性のトレードオフ
--------------------------------------------

差分プライバシーには、プライバシーとモデルの有用性の間のトレードオフが伴います。プライバシー予算 (ε) はすべてのフェデレーテッドラウンドにわたって累積されます。

.. list-table:: プライバシーレベル
   :widths: 15 20 20 45
   :header-rows: 1

   * - イプシロン (ε)
     - プライバシーレベル
     - モデルの精度
     - ユースケース
   * - ε ≤ 0.5
     - 非常に強力
     - 低い
     - 極めて機微（医療）
   * - ε = 0.5-1.0
     - 強力
     - 中程度
     - 機微（金融）
   * - ε = 1.0-3.0
     - 中程度
     - 良好（デフォルト）
     - 一般的なプライベートデータ
   * - ε = 3.0-10
     - 弱い
     - より良い
     - 軽度に機微
   * - ε > 10
     - 最小限
     - 最良
     - プライバシー目的には非推奨

**重要な注意点:**

- イプシロンの値はすべてのフェデレーテッドラウンドにわたって **累積** されます
- イプシロンが小さいほどプライバシーは強力ですが、より多くのラウンドが必要になったり精度が低下したりする場合があります
- ノイズ乗数は、目標のイプシロンを満たすように自動的に計算されます

**推奨事項:**

- プライバシーと有用性のバランスを取るため、まずは ``--target_epsilon 1.0``（デフォルト）から始めてください
- 極めて機微なデータ（医療、金融）の場合は ε ≤ 1.0 を使用してください
- 必要に応じて ``max_grad_norm``（勾配クリッピング）を調整してください
- プライベートデータでファインチューニングする前に、公開データで事前学習することを検討してください
- ラウンドをまたいだ累積イプシロンを監視してください

出力の概要
--------------

初期化
~~~~~~~~~~~~~~

* **TensorBoard**: ログは /tmp/nvflare/simulation/hello-dp/server/simulate_job/tb_events で確認できます
* **ワークフロー**: DP を有効にしたクライアントで FedAvg コントローラが初期化されます
* **プライバシー**: プライバシーエンジンはラウンド 0 で初期化され、累積予算を追跡します

各ラウンド
~~~~~~~~~~~~~~

* **モデルの配布**: グローバルモデルがクライアントに送信されます
* **ローカル学習**: 各クライアントは Opacus を用いた DP-SGD で学習します
* **プライバシーの追跡**: クライアントごとに累積イプシロン (ε) が記録されます
* **集約**: DP で学習された重み差分がサーバ上で集約されます

完了
~~~~~~~~

* **最終モデル**: プライバシー保証を備えた学習済みモデル
* **プライバシー予算**: 最終的な累積プライバシー予算が報告されます（target_epsilon 以下になるはずです）
* **想定される性能**（デフォルトの ε=1.0、5 ラウンド、2 クライアントの場合）:

  - グローバルテスト Accuracy: **99.95%**
  - グローバルテスト F1 スコア: **81.58%**
  - これらの結果は、強力なプライバシー保護を維持しながら効果的な不正検知が行えることを示しています

参考文献
------------

1. Abadi, M., et al. (2016). `Deep Learning with Differential Privacy <https://arxiv.org/abs/1607.00133>`_. ACM CCS 2016.
2. McMahan, B., et al. (2017). `Communication-Efficient Learning of Deep Networks from Decentralized Data <https://proceedings.mlr.press/v54/mcmahan17a>`_. AISTATS 2017.
3. `Opacus: User-friendly library for training PyTorch models with differential privacy <https://opacus.ai/>`_
4. `NVIDIA FLARE Documentation <https://nvflare.readthedocs.io/>`_
