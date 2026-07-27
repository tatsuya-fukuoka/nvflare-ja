2次のNewton-Raphson最適化による連合ロジスティック回帰
==========================================================================================================

この例では、2次のNewton-Raphson最適化を用いたロジスティック回帰による、
連合二値分類の実装方法を示します。

NVFLAREと依存関係のインストール
------------------------------------------------------------

完全なインストール手順については、`Installation <https://nvflare.readthedocs.io/en/main/installation.html>`_ を参照してください

.. code-block:: text

    pip install nvflare


GitHubからサンプルコードを取得します:

.. code-block:: text

    git clone https://github.com/NVIDIA/NVFlare.git

次に hello-lr ディレクトリに移動します:

.. code-block:: text

    git switch <release branch>
    cd examples/hello-world/hello-lr


依存関係をインストールします

.. code-block:: text

    pip install -r requirements.txt


コード構造
--------------------

.. code-block:: text

    hello-lr
    |
    |-- client.py         # client local training script
    |-- job.py            # job recipe that defines client and server configurations
    |-- download_data.py  # download dataset
    |-- prepare_data.py   # prepare data to convert to numpy
    |-- requirements.txt  # dependencies


データ
------------

この例では `UCI Heart Disease データセット <https://archive.ics.uci.edu/dataset/45/heart+disease>`_ を
使用します。

すべての属性は数値です。各データベースは同じインスタンス形式を持ちます。データベースには76の
生の属性がありますが、実際に使用されるのはそのうち14のみです。

データベースの作成者は次のように要請しています:

.. code-block:: text

      "...that any publications resulting from the use of the data include the
      names of the principal investigator responsible for the data collection
      at each institution.  They would be:

       1. Hungarian Institute of Cardiology. Budapest: Andras Janosi, M.D.
       2. University Hospital, Zurich, Switzerland: William Steinbrunn, M.D.
       3. University Hospital, Basel, Switzerland: Matthias Pfisterer, M.D.
       4. V.A. Medical Center, Long Beach and Cleveland Clinic Foundation:
	  Robert Detrano, M.D., Ph.D. "


データセットには4つのサイトからのサンプルが含まれており、以下のとおり
トレーニングセットとテストセットに分割されています:

.. list-table::
   :widths: 30 70
   :header-rows: 1

   * - サイト
     - サンプル分割
   * - Cleveland
     - train: 199サンプル, test: 104サンプル
   * - Hungary
     - train: 172サンプル, test: 89サンプル
   * - Switzerland
     - train: 30サンプル, test: 16サンプル
   * - Long Beach V
     - train: 85サンプル, test: 45サンプル

各サンプルの特徴量の数は13です。


特徴量
^^^^^^^^^^^^

.. list-table::
   :widths: 15 9 9 9 39 10 9
   :header-rows: 1

   * - 変数名
     - 役割
     - 型
     - 属性
     - 説明
     - 単位
     - 欠損値
   * - age
     - 特徴量
     - 整数
     - 年齢
     - 年数
     -
     - なし
   * - sex
     - 特徴量
     - カテゴリ
     - 性別
     -
     -
     - なし
   * - cp
     - 特徴量
     - カテゴリ
     -
     -
     -
     - なし
   * - trestbps
     - 特徴量
     - 整数
     -
     - 安静時血圧(入院時)
     - mm Hg
     - なし
   * - chol
     - 特徴量
     - 整数
     -
     - 血清コレステロール
     - mg/dl
     - なし
   * - fbs
     - 特徴量
     - カテゴリ
     -
     - 空腹時血糖 > 120 mg/dl
     -
     - なし
   * - restecg
     - 特徴量
     - カテゴリ
     -
     -
     -
     - なし
   * - thalach
     - 特徴量
     - 整数
     -
     - 最大心拍数
     -
     - なし
   * - exang
     - 特徴量
     - カテゴリ
     -
     - 運動誘発性狭心症
     -
     - なし
   * - oldpeak
     - 特徴量
     - 整数
     -
     - 安静時に対する運動誘発性ST低下
     -
     - なし
   * - slope
     - 特徴量
     - カテゴリ
     -
     -
     -
     - なし
   * - ca
     - 特徴量
     - 整数
     -
     - 透視法で着色された主要血管の数(0-3)
     -
     - あり
   * - thal
     - 特徴量
     - カテゴリ
     -
     -
     -
     - あり
   * - num
     - ターゲット
     - 整数
     -
     - 心疾患の診断
     -
     - なし

モデル
------------

`Newton-Raphson最適化 <https://en.wikipedia.org/wiki/Newton%27s_method>`_ の問題は、
次のように記述できます。

ロジスティック回帰による二値分類タスクにおいて、データサンプル :math:`x` が
陽性に分類される確率は次のように定式化されます:

.. math::

    p(x) = \sigma(\beta \cdot x + \beta_{0})

ここで :math:`\sigma(.)` はシグモイド関数を表します。:math:`\beta_{0}` と
:math:`\beta` は、単一のパラメータベクトル :math:`\theta =
( \beta_{0},  \beta)` にまとめることができます。:math:`d` を各データサンプル
:math:`x` の特徴量の数、:math:`N` をデータサンプルの数とすると、上記の確率の式の
行列版は次のようになります:

.. math::

    p(X) = \sigma( X \theta )

ここで :math:`X` はすべてのサンプルの行列で、形状は :math:`N \times (d+1)` であり、
切片 :math:`\theta_{0}` を考慮するために、最初の列は値1で埋められています。

目標は、以下の尤度関数を最大化するパラメータベクトル :math:`\theta` を
計算することです:

.. math::

    L_{\theta} = \prod_{i=1}^{N} p(x_i)^{y_i} (1 - p(x_i)^{1-y_i})

Newton-Raphson法は、2次近似によって尤度関数を最適化します。数学的な詳細を
省略すると、パラメータベクトル :math:`\theta` の理論上の更新式は次のとおりです:

.. math::

    \theta^{n+1} = \theta^{n} - H_{\theta^{n}}^{-1} \nabla L_{\theta^{n}}

ここで

.. math::

    \nabla L_{\theta^{n}} = X^{T}(y - p(X))

は尤度関数の勾配であり、:math:`y` はサンプルデータ行列 :math:`X` に対する
正解ラベルのベクトルです。また、

.. math::

    H_{\theta^{n}} = -X^{T} D X

は尤度関数のヘッシアンであり、:math:`D` は対角行列で、位置 :math:`(i,i)` の
対角値は :math:`D(i,i) = p(x_i) (1 - p(x_i))` です。

連合Newton-Raphson最適化では、各クライアントはローカルのトレーニングサンプルに
基づいて、それぞれの勾配 $\nabla L_{\theta^{n}}$ とヘッシアン $H_{\theta^{n}}$
を計算します。サーバーはすべてのクライアントで計算された勾配とヘッシアンを
集約し、上記の理論上の更新式に基づいてパラメータ $\theta$ の更新を実行します。

クライアント側
----------------------------

クライアント側では、ローカルトレーニングのロジックは :github_nvflare_link:`client.py <examples/hello-world/hello-lr/client.py>` に実装されています。

この実装は `Client API <https://nvflare.readthedocs.io/en/main/programming_guide/execution_api_type.html#client-api>`_ に基づいています。
これにより、ユーザーは最小限の `nvflare` 固有のコードを追加するだけで、典型的な
集中型トレーニングスクリプトを、連合学習のクライアント側ローカルトレーニング
スクリプトに変換できます。

- ローカルトレーニング中、各クライアントは `flare.receive()` API を使用して、
  サーバーから送信されたグローバルモデルのコピーを受け取ります。受け取った
  グローバルモデルは `FLModel` のインスタンスです。
- まずローカル検証が実行され、検証メトリクスが得られます

- 次に、各クライアントは、上で説明したそれぞれの理論式を使用して、ローカルの
  トレーニングデータに基づいて勾配とヘッシアンを計算します。これは
  :github_nvflare_link:`train_newton_raphson() <examples/hello-world/hello-lr/client.py>` メソッドに実装されています。その後、各クライアントは
  `flare.send()` API を使用して、計算結果(常に `FLModel` 形式)を集約のために
  サーバーに送信します。

各クライアントサイトは、上のデータテーブルに記載されたサイトに対応します。

トレーニングロジックは集中型のロジックとほぼ同じです。すなわち、データを読み込み、
トレーニング(Newton-Raphson更新)を実行し、トレーニング済みモデルを検証します。
連合学習のコードで追加された唯一の違いは、`FLModel` の受信や送信など、
FLシステムとのやり取りに関連するものです。


.. literalinclude:: ../../../examples/hello-world/hello-lr/client.py
    :language: python
    :linenos:
    :caption: Client code (client.py)
    :lines: 14-

サーバー側
------------------------

FLARE 組み込みの Newton-Raphson 法によるロジスティック回帰を活用します。
サーバー側の fedavg クラスは `nvflare.app_common.workflows.lr.fedavg.FedAvgLR` にあります

ジョブ
------------

.. literalinclude:: ../../../examples/hello-world/hello-lr/job.py
    :language: python
    :linenos:
    :caption: Job Recipe (job.py)
    :lines: 14-


データのダウンロードと準備
----------------------------------------------------

以下のスクリプトを実行します

.. code-block:: text

    python download_data.py
    python prepare_data.py


これにより、心疾患データセットが以下の場所にダウンロードされます

.. code-block:: text

    /tmp/flare/dataset/heart_disease_data/


ジョブの実行
------------------------

以下のコマンドを実行して、連合ロジスティック回帰を起動します。これは nvflare のシミュレーションモードで実行されます。

.. code-block:: text

    python job.py

