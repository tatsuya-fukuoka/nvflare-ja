.. _client_api:

##########
Client API
##########

.. note::
   **データサイエンティストの方へ:** 技術的な詳細を抑えた実践的なガイドが必要な場合は、ユーザーガイドの :ref:`client_api_usage` を参照してください。
   このページでは、研究者および開発者向けの包括的な技術ドキュメントを提供します。

   **FLAREが初めてですか?** まずは :ref:`quickstart` ガイドから始めてください。

FLARE Client API は、集中型のローカルトレーニングコードを連合学習(Federated Learning)コードへ簡単に変換する手段を提供し、以下の利点があります:

* 数行のコード変更のみで済み、コードの再構成や新しいクラスの実装が不要
* ユーザーに公開されるFLARE固有の新しい概念を削減
* 異なるフレームワーク(PyTorch、PyTorch Lightning、HuggingFace)を使用した既存のローカルトレーニングコードから容易に適応可能

コアコンセプト
==============

広く使われている連合学習(FL)ワークフロー「FedAvg」の一般的な構造は以下のとおりです:

#. FLサーバーが初期モデルを初期化する
#. 各ラウンド(グローバルイテレーション)で:

   #. FLサーバーがグローバルモデルをクライアントに送信する
   #. 各FLクライアントはこのグローバルモデルを起点として、自身のデータでトレーニングする
   #. 各FLクライアントはトレーニング済みモデルを送り返す
   #. FLサーバーがすべてのモデルを集約し、新しいグローバルモデルを生成する

クライアント側のトレーニングワークフローは以下のとおりです:

#. FLサーバーからモデルを受け取る
#. 受け取ったグローバルモデルに対してローカルトレーニングを実行する、および/またはモデル選択のために受け取ったグローバルモデルを評価する
#. 新しいモデルをFLサーバーに送り返す

集中型トレーニングコードを連合学習に変換するには、以下のステップを行うようにコードを適応させる必要があります:

#. 受信した :ref:`fl_model` から必要な情報を取得する
#. ローカルトレーニングを実行する
#. 結果を送り返すために新しい :ref:`fl_model` に格納する

一般的なユースケースでは、Client API には3つの必須メソッドがあります:

* ``init()``: NVFlare Client API 環境を初期化します。
* ``receive()``: NVFlare 側からモデルを受信します。
* ``send()``: NVFlare 側へモデルを送信します。

ユーザーは Client API を使用して、集中型トレーニングコードを連合学習に変更できます。例:

.. code-block:: python

    import nvflare.client as flare

    flare.init() # 1. Initializes NVFlare Client API environment.
    input_model = flare.receive() # 2. Receives model from NVFlare side.
    params = input_model.params # 3. Obtain the required information from received FLModel

    # original local training code begins
    new_params = local_train(params)
    # original local training code ends

    output_model = flare.FLModel(params=new_params) # 4. Put the results in a new FLModel
    flare.send(output_model) # 5. Sends the model to NVFlare side.

5行のコード変更で、集中型トレーニングコードを連合学習の設定に変換できました。

この後、ジョブレシピを利用して連合学習ジョブを定義・実行できます。詳細は :ref:`job_recipe` を参照してください。

Client API とジョブレシピの関係を理解する
=========================================================

Client API とジョブレシピ(Job Recipe)API は、連合学習ワークフローの中で異なる役割を担います:

* **Client API** (``nvflare.client``) - トレーニングスクリプト (``client.py``) で以下の用途に使用します:

  * FLサーバーからモデルを受信する
  * 更新したモデルをサーバーへ送り返す
  * FLシステム情報(ジョブID、サイト名など)にアクセスする
  * タスク種別(トレーニング、評価など)を判定する

* **ジョブレシピAPI** (``nvflare.recipe``) - ジョブ定義 (``job.py``) で以下の用途に使用します:

  * FLワークフロー(例: FedAvg、Cyclic、Swarm Learning)を定義する
  * ジョブパラメータ(クライアント数、ラウンド数、モデルなど)を指定する
  * 実行環境(シミュレーション、POC、本番環境)を設定する
  * 実験トラッキング、サイト横断評価などの機能を追加する

完全な動作例
=========================

Client API とジョブレシピがどのように連携するかを示す完全な例を示します:

**プロジェクト構成:**

.. code-block:: none

    my-fl-project/
    ├── job.py              # Job definition (Job Recipe API)
    ├── client.py           # Training script (Client API)
    ├── model.py            # Model definition (optional)
    └── requirements.txt    # Dependencies

**ステップ1: Client API を使用してトレーニングスクリプトを定義する** (``client.py``):

.. code-block:: python

    import nvflare.client as flare
    import torch
    from model import Net

    def train(net, train_loader, device):
        # Your existing training code
        net.train()
        for data, target in train_loader:
            # ... training logic ...
        return net

    def evaluate(net, test_loader, device):
        # Your existing evaluation code
        net.eval()
        accuracy = 0.0
        # ... evaluation logic ...
        return accuracy

    def load_data():
        # Your data loading logic here
        # This should return training and test data loaders
        from torchvision import datasets, transforms
        from torch.utils.data import DataLoader

        transform = transforms.Compose([
            transforms.ToTensor(),
            transforms.Normalize((0.5,), (0.5,))
        ])
        train_dataset = datasets.MNIST('./data', train=True, download=True, transform=transform)
        test_dataset = datasets.MNIST('./data', train=False, transform=transform)
        train_loader = DataLoader(train_dataset, batch_size=64, shuffle=True)
        test_loader = DataLoader(test_dataset, batch_size=64, shuffle=False)
        return train_loader, test_loader

    def main():
        # Initialize Client API
        flare.init()

        # Setup
        device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
        net = Net().to(device)
        train_loader, test_loader = load_data()

        # Federated Learning Loop
        while flare.is_running():
            # Receive global model from server
            input_model = flare.receive()

            # Load received weights
            net.load_state_dict(input_model.params)

            # Local training
            net = train(net, train_loader, device)

            # Evaluation
            accuracy = evaluate(net, test_loader, device)

            # Send updated model back to server
            output_model = flare.FLModel(
                params=net.state_dict(),
                metrics={"accuracy": accuracy}
            )
            flare.send(output_model)

    if __name__ == "__main__":
        main()

**ステップ2: ジョブレシピを使用してFLジョブを定義する** (``job.py``):

.. code-block:: python

    import argparse
    from model import Net
    from nvflare.app_opt.pt.recipes.fedavg import FedAvgRecipe
    from nvflare.recipe import SimEnv, add_experiment_tracking

    def main():
        parser = argparse.ArgumentParser()
        parser.add_argument("--n_clients", type=int, default=2)
        parser.add_argument("--num_rounds", type=int, default=5)
        parser.add_argument("--batch_size", type=int, default=32)
        args = parser.parse_args()

        # Create the FL job using Recipe API
        recipe = FedAvgRecipe(
            name="my-pytorch-job",
            min_clients=args.n_clients,
            num_rounds=args.num_rounds,
            # Model can be class instance or dict config
            # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
            model=Net(),
            train_script="client.py",
            train_args=f"--batch_size {args.batch_size}",
        )

        # Optional: Add experiment tracking
        add_experiment_tracking(recipe, tracking_type="tensorboard")

        # Execute in simulation environment
        env = SimEnv(num_clients=args.n_clients)
        run = recipe.execute(env)

        print(f"Job completed with status: {run.get_status()}")
        print(f"Results saved to: {run.get_result()}")

    if __name__ == "__main__":
        main()

**ステップ3: FLジョブを実行する:**

.. code-block:: bash

    # Run with default parameters
    python job.py

    # Run with custom parameters
    python job.py --n_clients 5 --num_rounds 10 --batch_size 64

同じジョブは、環境を変更するだけで異なる環境で実行できます:

.. code-block:: python

    # Simulation (single process, fast for debugging)
    from nvflare.recipe import SimEnv
    env = SimEnv(num_clients=2)

    # POC (multi-process, closer to production)
    from nvflare.recipe import PocEnv
    env = PocEnv(num_clients=2)

    # Production (distributed deployment)
    from nvflare.recipe import ProdEnv
    env = ProdEnv(startup_kit_location="/path/to/admin/startup")

このアプローチの主な利点
==============================

1. **関心の分離**: トレーニングロジック(Client API)とジョブ設定(ジョブレシピ)が分離されています
2. **最小限のコード変更**: わずか数行の変更で集中型トレーニングをFLに変換できます
3. **環境の柔軟性**: 同じコードがシミュレーション、POC、本番環境で動作します
4. **容易な実験**: トレーニングコードを変更せずにFLパラメータを変更できます
5. **組み込み機能**: トラッキングやサイト横断評価などを関数呼び出し1つで追加できます

Client API リファレンス
=======================

以下は主要な Client API の一覧表です。

.. list-table:: Client API
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - APIドキュメントへのリンク
   * - init
     - NVFlare Client API 環境を初期化します。
     - :func:`init<nvflare.client.api.init>`
   * - receive
     - NVFlare 側からモデルを受信します。
     - :func:`receive<nvflare.client.api.receive>`
   * - send
     - NVFlare 側へモデルを送信します。
     - :func:`send<nvflare.client.api.send>`
   * - system_info
     - NVFlare のシステム情報を取得します。
     - :func:`system_info<nvflare.client.api.system_info>`
   * - get_job_id
     - ジョブIDを取得します。
     - :func:`get_job_id<nvflare.client.api.get_job_id>`
   * - get_site_name
     - サイト名を取得します。
     - :func:`get_site_name<nvflare.client.api.get_site_name>`
   * - is_running
     - NVFlare システムが稼働中かどうかを返します。
     - :func:`is_running<nvflare.client.api.is_running>`
   * - is_train
     - 現在のタスクがトレーニングタスクかどうかを返します。
     - :func:`is_train<nvflare.client.api.is_train>`
   * - is_evaluate
     - 現在のタスクが評価(evaluate)タスクかどうかを返します。
     - :func:`is_evaluate<nvflare.client.api.is_evaluate>`
   * - is_submit_model
     - 現在のタスクが submit_model タスクかどうかを返します。
     - :func:`is_submit_model<nvflare.client.api.is_submit_model>`

.. list-table:: Lightning APIs
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - APIドキュメントへのリンク
   * - patch
     - PyTorch Lightning の Trainer に FLARE で使用するためのパッチを適用します。
     - :func:`patch<nvflare.app_opt.lightning.api.patch>`

.. list-table:: HuggingFace APIs
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - APIドキュメントへのリンク
   * - patch
     - HuggingFace の ``Trainer`` または TRL の ``SFTTrainer`` に FLARE で使用するためのパッチを適用します。
     - :func:`patch<nvflare.app_opt.hf.api.patch>`
   * - is_running
     - パッチ適用済みの HuggingFace トレーナーのFLループを調整します。
     - :func:`is_running<nvflare.client.hf.is_running>`

.. list-table:: Metrics Logger
   :widths: 25 25 50
   :header-rows: 1

   * - API
     - 説明
     - APIドキュメントへのリンク
   * - SummaryWriter
     - SummaryWriter は Tensorboard の SummaryWriter の使い方を模倣します。
     - :class:`SummaryWriter<nvflare.client.tracking.SummaryWriter>`
   * - WandBWriter
     - WandBWriter は Weights & Biases の使い方を模倣します。
     - :class:`WandBWriter<nvflare.client.tracking.WandBWriter>`
   * - MLflowWriter
     - MLflowWriter は MLflow の使い方を模倣します。
     - :class:`MLflowWriter<nvflare.client.tracking.MLflowWriter>`

Client API を使用すべきとき
===========================

Client API はほとんどのユーザーにとって\ **推奨される出発点**\ であり、特に以下の場合に適しています:

* 既存の集中型トレーニングコードがある場合
* 最小限のコード変更でFLを有効にしたい場合
* 一般的なフレームワーク(PyTorch、TensorFlow、NumPy など)を使用している場合
* 素早い実験やプロトタイピングが必要な場合
* シンプルで直感的なAPIを好む場合

その他の実行APIについては、:ref:`execution_api_type` を参照してください。

追加リソース
====================

**APIドキュメント:**

* Client API モジュール: :mod:`nvflare.client.api` - 完全なAPIリファレンス
* PyTorch Lightning API: :mod:`nvflare.app_opt.lightning.api` - Lightning 固有の連携
* HuggingFace Client API: :ref:`hf_client_api` - HuggingFace Trainer 連携ガイド
* ジョブレシピガイド: :ref:`job_recipe` - FLジョブの定義と実行方法

**ガイド:**

* :ref:`client_api_usage` - より多くの例を含むユーザーガイド
* :ref:`job_recipe` - ジョブレシピのチュートリアル
* :ref:`fl_simulator` - シミュレーション環境の詳細

Client API の通信パターン
=================================

.. image:: ../../resources/client_api.png
    :height: 300px

さまざまなシナリオに合わせた Client API の実装を複数提供しており、それぞれが異なる通信パターンと結び付いています。

インプロセス Client API
-----------------------

インプロセスエグゼキューターでは、トレーニングスクリプトとクライアントエグゼキューターの両方が同一プロセス内で動作します。
トレーニングスクリプトは START_RUN イベントの発生時に一度だけ起動され、END_RUN イベントまで実行し続けます。
両者の間の通信は、効率的なインメモリのデータバスを介して行われます。

トレーニングプロセスが単一GPUまたはGPUなしで行われ、トレーニングスクリプトがサードパーティのトレーニングシステムと統合していない場合は、(利用可能であれば)インプロセスエグゼキューターが望ましい選択です。

サブプロセス Client API
-----------------------

一方、LauncherExecutor は SubprocessLauncher を用いてサブプロセスでトレーニングスクリプトを実行します。その結果、クライアントエグゼキューターとトレーニングスクリプトは別々のプロセスに存在します。SubprocessLauncher には "launch_once" オプションが用意されており、サーバーからタスクを受け取るたびに外部スクリプトを起動するか、START_RUN イベントで一度だけスクリプトを起動して END_RUN イベントまで実行し続けるかを制御できます。両者の間の通信は、CellPipe(デフォルト)または FilePipe によって行われます。

マルチGPUトレーニングや外部トレーニングインフラを利用するシナリオでは、Launcher エグゼキューターを選択する方が適している場合があります。


さまざまな Pipe の選択
=========================

2.5.x リリースでは、ほとんどのユーザーに対して、インプロセスエグゼキューターのデフォルト設定(メモリベースのデータ交換がデフォルト)の利用を推奨します。
一方、2.4.x リリースでは、ほとんどのユーザーに CellPipe を用いたデフォルト設定の利用を推奨します。

CellPipe は、ローカルホスト上の Executor プロセスとトレーニングスクリプトプロセスの間で、TCPベースのセル間(cell-to-cell)接続を実現します。セル(cell)という用語は論理的なエンドポイントを表します。この通信により、2つのプロセス間でモデル、メトリクス、メタデータの交換が可能になります。

これに対して FilePipe は、Executor プロセスとトレーニングスクリプトプロセスの間でファイルベースの通信を提供し、ジョブ固有のファイルディレクトリを利用してファイル経由でモデルとメタデータを交換します。FilePipe は CellPipe よりもセットアップが容易ですが、高頻度のメトリクス交換には適していません。

例
========

さまざまなフレームワークで Client API とジョブレシピを使用した完全な動作例:

**Hello World の例**\ (初心者に推奨):

- PyTorch: :ref:`hello_pt_job_api` - CIFAR-10 画像分類
- NumPy: :github_nvflare_link:`hello-numpy <examples/hello-world/hello-numpy>` - FLの基本概念
- PyTorch Lightning: :github_nvflare_link:`hello-lightning <examples/hello-world/hello-lightning>` - Lightning 連携
- TensorFlow: :ref:`hello_tf_job_api` - MNIST 分類
- HuggingFace Trainer: :github_nvflare_link:`hello-huggingface <examples/hello-world/hello-huggingface>` - HuggingFace Client API による Qwen SFT/PEFT
- Flower: :github_nvflare_link:`hello-flower <examples/hello-world/hello-flower>` - FLARE 上の Flower

**高度な例:**

- HuggingFace LLM チューニング: :github_nvflare_link:`llm_hf <examples/advanced/llm_hf>` - 大規模モデルの SFT/PEFT、量子化、マルチGPUパターン
- XGBoost: :github_nvflare_link:`xgboost examples <examples/advanced/xgboost>` - ツリーベースの連合学習
- Scikit-learn: :github_nvflare_link:`sklearn-linear <examples/advanced/sklearn-linear>`、:github_nvflare_link:`sklearn-kmeans <examples/advanced/sklearn-kmeans>`、:github_nvflare_link:`sklearn-svm <examples/advanced/sklearn-svm>` - 従来型の機械学習アルゴリズム

**自己学習教材:**

段階的に学習するには、:ref:`self_paced_training` の教材を参照してください。
さまざまなFLアルゴリズム(FedAvg、Cyclic、Swarm Learning など)を、充実したチュートリアルと例とともに扱っています。


カスタムデータクラスのシリアライズ/デシリアライズ
===============================================================

カスタムクラスの形式でデータを渡すには、NVFlare 内のシリアライズツールを活用できます。

例:

.. code-block:: python

    class CustomClass:
        def __init__(self, x, y):
            self.x = 1
            self.y = 2

コードで ``Enum`` から派生したクラスやデータクラス(dataclass)を使用している場合、それらはデフォルトのデコンポーザーで処理されます。
その他のカスタムクラスについては、専用のカスタムデコンポーザーを作成し、サーバー側とクライアント側の両方、および train.py 内で fobs.register を使用して登録されていることを確認する必要があります。

なお、カスタムデータクラスを機能させるには、train.py とは別のファイルに配置する必要があります。

シリアライズの詳細については、:ref:`serialization` を参照してください。
