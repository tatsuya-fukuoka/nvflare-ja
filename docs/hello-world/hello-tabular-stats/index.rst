.. _hello_tabular_stats:

表形式データのフェデレーテッド統計
=========================================

この例では、Pandas の DataFrame として表現できる表形式データに対して、フェデレーテッド統計を生成する方法を示します。


NVIDIA FLARE のインストール
------------------------------------
インストール手順の詳細については :doc:`Installation </installation>` を参照してください。

.. code-block:: text

    pip install nvflare


GitHub からサンプルコードを取得します。

.. code-block:: text

    git clone https://github.com/NVIDIA/NVFlare.git

次に hello-tabular-stats ディレクトリに移動します。

.. code-block:: text

    git switch <release branch>
    cd examples/hello-world/hello-tabular-stats


依存関係のインストール
------------------------------

    pip install -r requirements.txt


オプションの分位数依存関係のインストール -- fastdigest
------------------------------------------------------------

分位数（quantile）を計算する場合は ``fastdigest==0.4.0`` をインストールしてください。

分位数の統計が不要な場合は、この手順をスキップしてください。

.. code-block:: text

    pip install fastdigest==0.4.0


Ubuntu では、次のようなエラーが表示されることがあります。

.. code-block:: text

  Cargo, the Rust package manager, is not installed or is not on PATH.
  This package requires Rust and Cargo to compile extensions. Install it through
  the system's package manager or via https://rustup.rs/

  Checking for Rust toolchain....

これは、fastdigest（またはその依存関係）のビルドに Rust と Cargo が必要なためです。

Ubuntu システムに Rust と Cargo をインストールする必要があります。次の手順に従ってください。
Rust と Cargo をインストールする
rustup を使って Rust をインストールするには、次のコマンドを実行します。

.. code-block:: text

    ./install_cargo.sh

その後、再度 fastdigest をインストールできます。

.. code-block:: text

    pip install fastdigest==0.4.0


コード構造
--------------

.. code-block:: text

    hello-tabular-stats
    |
    ├── client.py         # client local training script
    ├── job.py            # job recipe that defines client and server configurations
    ├── prepare_data.py   # utilities to download data
    ├── install_cargo.sh  # scripts to install rust and cargo needed for quantil dependency, only needed if you plan to install quantile dependency
    └── requirements.txt  # dependencies
    ├── demo
    │   └── visualization.ipynb # Visualization Notebook


データ
--------

この例では、UCI（University of California, Irvine）の `adult dataset <https://archive.ics.uci.edu/dataset/2/adult>`_ を使用します。

元のデータセットにはすでに「training」と「test」のデータセットが含まれています。ここでは単純に、学習データとテストデータが異なるクライアントに属していると仮定します。
そこで、学習データとテストデータを 2 つのクライアントに割り当てます。

ここでは、データユーティリティを使用して UCI データセットをダウンロードし、/tmp/nvflare/data/ ディレクトリ配下のクライアントごとのパッケージディレクトリに配置します。

UCI のウェブサイトは一時的に停止する場合がある点にご注意ください。

.. code-block:: text

    python prepare_data.py

次のような出力が表示されるはずです。

prepare data for data directory /tmp/nvflare/df_stats/data

.. code-block:: text

    download to /tmp/nvflare/df_stats/data/site-1/data.csv
    skip empty line


    download to /tmp/nvflare/df_stats/data/site-2/data.csv
    skip empty line

    done with prepare data


クライアントコード
--------------------

ローカルの統計ジェネレータです。統計ジェネレータ `AdultStatistics` は `Statistics` の仕様を実装しています。

.. literalinclude:: ../../../examples/hello-world/hello-tabular-stats/client.py
    :language: python
    :linenos:
    :caption: Client Code (client.py)
    :lines: 14-


表形式統計に必要な関数の多くは、すでに DFStatisticsCore に実装されています。

`AdultStatistics` クラスで実際に必要となるのは、次の内容です。

- data_features -- ここでは特徴量名の配列をハードコードしています。
- `load_data() -> Dict[str, pd.DataFrame]` 関数の実装。このメソッドは、
  各データソース（"train"、"test"）ごとに 1 つずつの pandas DataFrame を持つ辞書を返します
- `data_path = <data_root_dir>/<site-name>/<filename>`

サーバコード
--------------
サーバ側の集約はすでに Statistics Controller に実装されています。

Job Recipe
----------

ジョブは recipe を介して定義され、Simulation Execution Env で実行します。

.. literalinclude:: ../../../examples/hello-world/hello-tabular-stats/job.py
    :language: python
    :linenos:
    :caption: job Recipe (job.py)
    :lines: 14-



統計の設定によって、生成する統計量が決まります。
以下は一例です。

.. code-block:: text

    statistic_configs = {
        "count": {},
        "mean": {},
        "sum": {},
        "stddev": {},
        "histogram": {"*": {"bins": 20}, "Age": {"bins": 20, "range": [0, 100]}},
        "quantile": {"*": [0.1, 0.5, 0.9]},
    }


ジョブの実行
--------------
ターミナルからコードを実行してみてください。

.. code-block:: text

    python job.py


次のような出力が表示されるはずです。

.. code-block:: text

    2025-09-03 20:42:03,392 - INFO - save statistics result to persistence store
    2025-09-03 20:42:03,392 - INFO - job dir = /tmp/nvflare/simulation/stats_df/server/simulate_job
    2025-09-03 20:42:03,395 - INFO - trying to save data to /tmp/nvflare/simulation/stats_df/server/simulate_job/statistics/adults_stats.json
    2025-09-03 20:42:03,395 - INFO - file /tmp/nvflare/simulation/stats_df/server/simulate_job/statistics/adults_stats.json saved


結果はワークスペース "/tmp/nvflare" に保存されます。

.. code-block:: text

    /tmp/nvflare/simulation/stats_df/server/simulate_job/statistics/adults_stats.json


可視化
--------

JSON 形式であれば、Pandas DataFrame とプロットを用いてデータを簡単に可視化できます。
出力された ``adults_stats.json`` ファイルをダウンロードして demo ディレクトリにコピーし、Jupyter ノートブック ``visualization.ipynb`` を実行してください。




