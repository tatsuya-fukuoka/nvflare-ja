########################################
XGBoost のための連合学習
########################################

概要
================
このガイドでは、NVIDIA FLARE (NVFlare) を使用して連合学習環境で XGBoost モデルを学習する方法を説明します。セキュリティレベルの異なる複数のコラボレーション戦略を紹介します。

NVFlare は次の利点を提供します。

- 準同型暗号 (HE) によるセキュアな学習。ローカルヒストグラムと勾配を連合サーバーやパッシブパーティから保護します。
- XGBoost プロセスのライフサイクル管理
- ネットワークの一時的な不調を克服できる信頼性のあるメッセージング
- リレーを用いた複雑なネットワーク上での学習

このガイドでは、次のような複数の連合 XGBoost 構成を扱います。

- **水平コラボレーション** : ヒストグラムベースおよびツリーベースのアプローチ（非セキュアおよびセキュア）
- **垂直コラボレーション** : ヒストグラムベースのアプローチ（非セキュアおよび準同型暗号によるセキュア）

XGBoost とは？
--------------------
XGBoost (eXtreme Gradient Boosting) は、分類および回帰タスクに決定木／回帰木を用いる強力な機械学習アルゴリズムです。特にテーブルデータで優れた性能を発揮し、次の理由から現在も広く使われています。

- 構造化データに対する **高い性能**
- 予測の **説明可能性**
- **計算効率の高さ**

これらの例では `DMLC XGBoost <https://github.com/dmlc/xgboost>`_ を使用します。これは次の機能を提供します。

- GPU アクセラレーション機能
- 分散学習および連合学習のサポート
- 最適化された勾配ブースティングの実装

連合学習のモード
====================

水平連合学習
--------------------
水平コラボレーションでは、各参加者は次の状態にあります。

- すべてのサイトで **同じ特徴量** （列）を持つ
- サイトごとに **異なるデータサンプル** （行）を持つ
- ラベル所有者として **対等な立場** にある

**例** : 複数の病院がそれぞれ完全な患者記録（すべての特徴量）を持っているが、患者は異なる。

垂直連合学習
--------------------
垂直コラボレーションでは、各参加者は次の状態にあります。

- サイトごとに **異なる特徴量** （列）を持つ
- すべてのサイトで **同じデータサンプル** （行）を持つ
- **1 つの「アクティブパーティ」** （ラベル所有者）と複数の「パッシブパーティ」が存在する

**例** : 銀行と小売業者が同じ顧客に関するデータを持っているが、属性が異なる（財務情報と購買行動）。

サポートされる学習モード
----------------------------
NVFlare 上で実行する場合、XGBoost の通信はすべてローカルで行われ、メッセージは NVFlare の通信インフラを通じて転送されます。暗号化は XGBoost 内で暗号化プラグインによって処理されます。これは実行時にインストールできる外部コンポーネントです。

NVFlare は次の 4 つのモードで連合学習をサポートします。

1. **HE ベースのセキュリティ保護なしの水平** - ヒストグラムベースまたはツリーベース（ツリーベースは送信前に "sum_hessian" 値を削除することでセキュア化されます）
2. **HE ベースのセキュリティ保護なしの垂直** - ヒストグラムベース
3. **HE ありの水平** - ヒストグラムベース（ヒストグラムを連合サーバーから保護）
4. **HE ありの垂直** - ヒストグラムベース（勾配をパッシブパーティから保護）

セキュリティリスクと緩和策
==============================

リスク
------------

連合 XGBoost には、主に 3 つのセキュリティリスクがあります。

1. **モデル統計の漏洩** : デフォルトの XGBoost JSON モデルには "sum_hessian" 統計量が含まれており、モデル反転攻撃によってデータ分布を復元できてしまいます。（参考: `TimberStrike <https://arxiv.org/abs/2506.07605>`_ ）

2. **ヒストグラムの漏洩** : 勾配ヒストグラムを悪用してデータ分布を再構成できます。ヒストグラムからも同じ "sum_hessian" のモデル統計量を導出できます。（参考: `TimberStrike <https://arxiv.org/abs/2506.07605>`_ ）

3. **勾配の漏洩** : サンプル単位の勾配はラベル情報を明らかにする可能性があります。（参考: `SecureBoost <https://arxiv.org/abs/1901.08755>`_ ）

攻撃対象領域
----------------

攻撃対象領域は、コラボレーションモードとパーティの役割によって異なります。

**サーバー** : コラボレーションモードに応じて、サーバーは次の情報にアクセスできる可能性があります。

1. ローカルモデル:

   - 水平ツリーベース:

      - 各クライアントのデータ分布に対する **モデル統計の漏洩**

2. ローカルヒストグラム:

   - 水平ヒストグラムベース／垂直ヒストグラムベース:

      - 各クライアント／パッシブパーティのデータ分布に対する **ヒストグラムの漏洩**

3. サンプル単位の勾配:

   - 垂直ヒストグラムベース:

      - アクティブパーティのラベル情報に対する **勾配の漏洩**

**クライアント** : コラボレーションモードに応じて、クライアントは次の情報にアクセスできる可能性があります。

1. 集約されたグローバルモデル:

   - 水平ツリーベース:

      - グローバルなデータ分布に対する **モデル統計の漏洩**

2. グローバルヒストグラム:

   - 水平ヒストグラムベース:

      - グローバルなデータ分布に対する **ヒストグラムの漏洩**

3. ローカルヒストグラム:

   - 垂直ヒストグラムベース:

      - アクティブパーティ上での各パッシブパーティのデータ分布に対する **ヒストグラムの漏洩**

4. サンプル単位の勾配:

   - パッシブパーティ上でのアクティブパーティのラベル情報に対する **勾配の漏洩**

緩和策
------------

次の表は、さまざまなコラボレーションシナリオで利用可能な緩和策をまとめたものです。

.. list-table:: コラボレーションモード別の緩和策
   :widths: 15 12 28 18 20 20
   :header-rows: 1

   * - コラボレーションモード
     - アルゴリズム
     - データ交換
     - 緩和されるリスク
     - セキュリティ対策
     - 実装
   * - **水平**
     - ツリーベース
     - クライアントがローカルでブーストしたツリーをサーバーに送信し、サーバーがそれらを統合してクライアントに配布します
     - サーバーとクライアントの両方における **モデル統計の漏洩**
     - JSON モデルから "sum_hessian" 値を削除
     - クライアントがローカルツリーをサーバーに送信する前に削除されます
   * - **水平**
     - ヒストグラムベース
     - クライアントがローカルヒストグラムをサーバーに送信し、サーバーがグローバルヒストグラムに集約してクライアントに配布します
     - サーバーにおける **ヒストグラムの漏洩** （クライアント側は残存）
     - ヒストグラムを暗号化
     - 送信前にローカルヒストグラムを暗号化します
   * - **垂直**
     - ヒストグラムベース
     - アクティブパーティが勾配を計算し、サーバーによってルーティングされ、パッシブパーティが勾配を受け取ってヒストグラムを計算し、サーバー経由でアクティブパーティに返します
     - サーバーにおける **ヒストグラムの漏洩** （アクティブパーティ側は残存）、サーバーとパッシブパーティの両方における **勾配の漏洩**
     - **主目的** : 勾配を暗号化、 **副次目的** : 分割値における特徴量の所有者をマスク
     - パッシブパーティへ送信する前に勾配を暗号化します

**注記:**

- **垂直ヒストグラムベース** :

  - **主目的** : パッシブパーティからサンプル勾配を保護する（重要）
  - **副次目的** : 特徴量の非所有者から分割値を隠す（望ましいがリスクは低い）

- **残る 2 つのリスク** については `高度なトピック: 将来のセキュリティシナリオ`_ の節で説明します。

TimberStrike 攻撃の分析
----------------------------

TimberStrike は、 ``sum_hessian`` 値とツリー構造を悪用して学習データの分布を推定するモデル反転攻撃です。実験結果はデータセットの規模によって大きく異なります。

.. list-table:: 再構成精度の結果
   :widths: 20 15 15 25
   :header-rows: 1

   * - データセット
     - サンプル数
     - 特徴量数
     - 再構成精度
   * - Diabetes（トイデータ）
     - 768
     - 8
     - 65.80%
   * - CreditCard（現実的なデータ）
     - 284,807
     - 30
     - 8.72%

.. note::

   上記の結果は、NVFlare による ``sum_hessian`` の削除 **以前** に得られたものです。つまり、攻撃者が完全なモデル統計量を利用できる状態でのものです。NVFlare の組み込み保護を有効にすると（下記参照）、TimberStrike の主要な情報源が排除されるため、攻撃性能は大幅に低下すると期待されます。「再構成精度」は距離の許容度に基づく指標であり（厳密な復元ではありません）、正確な定義については `TimberStrike の論文 <https://arxiv.org/abs/2506.07605>`_ を参照してください。

リスク評価
~~~~~~~~~~~~~~

実用的なデータセット（ `CreditCard <https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud>`_ ）では、 ``sum_hessian`` が利用可能であっても TimberStrike の精度は 10% 未満です。これを相対的に理解するため、参考として `NeMo SafeSynthesizer <https://docs.nvidia.com/nemo/microservices/latest/studio/safe-synthesizer.html>`_ を用います。SafeSynthesizer は、コンプライアンス (GDPR、HIPAA) を目的として設計されたプライバシー重視の合成データ生成ツールであり、メンバーシップ推論攻撃への保護を組み込み、オプションで差分プライバシー保証も提供します。こうしたプライバシー保護策を備えていてもなお、その合成データは実サンプルに対して 51.98% の近接度を達成します。これは、データの有用性を保つにはある程度の統計的類似性が必要だからです。TimberStrike の 8.72% は、この参照点を大きく下回ります。許容できるプライバシー水準は本質的にデータ依存です。ユーザーの皆さんには、ご自身のデータセットで同様の比較を実施することをお勧めします。

保護
~~~~~~~~

- **組み込み** : NVFlare は、水平ツリーベースモードにおいてモデル送信から ``sum_hessian`` を削除し、この攻撃の主要な情報源を排除します。
- **追加対策** : ``min_child_weight`` を大きくして、リーフごとに必要となるインスタンス重み（ヘシアン）の最小合計を引き上げます。これにより、分割の少ない粗いツリー構造になります。 `TimberStrike の論文 <https://arxiv.org/abs/2506.07605>`_ では、ツリーの深さ（ひいては分割数）が再構成精度に直接影響することが示されており、ツリーの粒度を粗くすることで情報の露出を抑えられると期待されます。最適な値はタスク依存です。プライバシーと有用性のトレードオフの分析については論文を参照してください。このパラメータはレシピの ``xgb_params`` に追加できます。

  .. code-block:: python

     from nvflare.recipe import set_per_site_config

     recipe = XGBHorizontalRecipe(
         name="xgb_higgs_horizontal",
         min_clients=2,
         num_rounds=100,
         xgb_params={
             "max_depth": 8,
             "eta": 0.1,
             "objective": "binary:logistic",
             "eval_metric": "auc",
             "min_child_weight": 100,  # increase for coarser trees and reduced privacy exposure
         },
     )
     set_per_site_config(recipe, per_site_config)

最も近い再構成サンプル (CreditCard)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下の各例は、それぞれの手法における最も近い一致（最小距離）を示しています。これらは異なる元レコードであり、各手法の再構成品質を独立に示すために掲載している点に注意してください。

*TimberStrike（精度 8.72%）*:

.. code-block::

   Original:      [-27.0, -25.3, -12.1, -1.53, -3.67, -1.82, -3.34, -26.6, 1.08, -0.42, 3.61, -5.42, ...]
   Reconstructed: [-30.0, -29.2, -10.5, 7.60, 2.20, -0.11, 4.55, -5.84, 5.50, 4.38, 3.07, 1.26, ...]

*SafeSynthesizer（精度 51.98%）*:

.. code-block::

   Original:      [2.06, -0.03, -1.06, 0.42, -0.13, -1.21, 0.20, -0.35, 0.51, 0.07, -0.70, 0.54, ...]
   Reconstructed: [2.06, -0.05, -1.07, 0.41, -0.12, -1.20, 0.20, -0.34, 0.50, 0.06, -0.68, 0.53, ...]

TimberStrike は、最も近い一致においてさえ大きな乖離を示します（例: 特徴量 4: -1.53 → 7.60）。一方、SafeSynthesizer の最も近い一致は特徴量ごとに 0.01〜0.02 しか差がないにもかかわらず、設計上プライバシー要件を満たしています。このことは、CreditCard のようなデータセットにおいて、TimberStrike の再構成が意味のあるプライバシーリスクを構成しない可能性を示唆しています。

GPU アクセラレーション
============================

連合 XGBoost は 2 段階の GPU アクセラレーションをサポートします。

1. XGBoost の GPU 学習
----------------------------
XGBoost モデルの初期化時に ``tree_method='gpu_hist'`` を設定することで、GPU による学習の高速化を有効にできます。

- **性能** : CPU 学習に対して最大 **4.15 倍の高速化** （ `GPU XGBoost Blog <https://developer.nvidia.com/blog/gradient-boosting-decision-trees-xgboost-cuda/>`_ ）

2. GPU アクセラレーションによる準同型暗号 (HE)
--------------------------------------------------
NVFlare は、専用の暗号化プラグインを用いて HE 演算の GPU アクセラレーションを提供します。

- **性能** : CPU 暗号化に対して最大 **36.5 倍の高速化** （ `NVFlare Secure XGBoost Blog <https://developer.nvidia.com/blog/security-for-data-privacy-in-federated-learning-with-cuda-accelerated-homomorphic-encryption-in-xgboost/>`_ ）

これらをそれぞれ「CPU/GPU XGBoost」および「CPU/GPU 暗号化」と呼ぶことにします。

セキュリティ実装マトリクス
==============================

次の表は、異なるハードウェア構成でどのセキュリティ対策がサポートされるかを示しています。

.. list-table:: セキュリティ実装マトリクス
   :widths: 18 30 13 13 13 13
   :header-rows: 1

   * - コラボレーションモード
     - セキュリティ目標
     - CPU XGBoost + CPU 暗号化
     - CPU XGBoost + GPU 暗号化
     - GPU XGBoost + CPU 暗号化
     - GPU XGBoost + GPU 暗号化
   * - **水平**
     - サーバーに対するヒストグラムの保護
     - ✅
     - N/A\*
     - ✅
     - N/A\*
   * - **垂直**
     - **主目的** : 勾配の保護
     - ✅
     - ✅
     - ✅
     - ✅
   * - **垂直**
     - **副次目的** : 分割値のマスク
     - ✅
     - ✅
     - ❌
     - ❌

**\*注記** : 水平ヒストグラムの暗号化は計算負荷が高くない（ヒストグラムベクトルを暗号化するだけ）ため、GPU 暗号化は必要ありません。

**実装上の注記** :

- **垂直モードの主目的** （勾配の保護）: すべての構成で完全にサポートされます
- **垂直モードの副次目的** （分割値のマスク）: CPU XGBoost でのみサポートされます

高度なトピック: 将来のセキュリティシナリオ
==============================================

以下のセキュリティシナリオは、現在の本ソリューションでは実装されていません。ユーザーは、 **平文でのヒストグラム通信** がデータ分布の情報を明らかにし、前述のようなデータ再構成攻撃を可能にし得ることを認識しておく必要があります。一方で、同様の統計量は `federated statistics <https://nvflare.readthedocs.io/en/main/examples/federated_statistics_overview.html>`_ のような一般的な手法からも導出できます。攻撃の有効性は、データの複雑さ、モデルのハイパーパラメータ、利用可能なデータ分布情報など複数の要因に依存するため、ある種の攻撃に対する示唆は大きく変動し得ます。これは依然として未解決かつ活発な研究領域です。

すべてのパーティに対する保護のための将来的な機能強化候補
------------------------------------------------------------

.. list-table:: 将来のセキュリティシナリオ
   :widths: 15 12 20 25 28
   :header-rows: 1

   * - コラボレーションモード
     - アルゴリズム
     - 残存するセキュリティリスク
     - 想定されるアプローチ
     - 課題
   * - **水平**
     - ヒストグラムベース
     - クライアントにおけるグローバルなデータ分布のヒストグラム漏洩（上記で対処済みのサーバーに加えて）
     - コンフィデンシャルコンピューティング、先進的な HE
     - サーバーが計算を行い最終的な分割のみを配布する方式における HE の互換性の課題 [*]_
   * - **垂直**
     - ヒストグラムベース
     - アクティブパーティにおける各パッシブパーティのデータ分布のヒストグラム漏洩（上記で対処済みのサーバーにおけるヒストグラム漏洩、およびサーバーとパッシブパーティにおける勾配漏洩に加えて）
     - ローカルでのデータ前処理と匿名化、コンフィデンシャルコンピューティング、先進的な HE
     - パッシブパーティが計算を行い最終的な分割のみを送信する方式における HE の互換性の課題 [*]_

.. [*] **HE の互換性に関する課題** : 現在の準同型暗号方式は、暗号文の除算や argmax といった演算を効率的にサポートしていません。これらは暗号化されたデータ上で分割計算を行うために必要です。「サーバー／パッシブパーティ上で分割まで計算を行う」アプローチをサポートするには、先進的な HE の機能が必要です。

前提条件
==============

必要な Python パッケージ
----------------------------

NVFlare 2.7.2 以上、

.. code-block:: bash

    pip install nvflare~=2.7.2

連合セキュア XGBoost。次のコマンドでバイナリビルドからインストールできます。

.. code-block:: bash

    pip install https://s3-us-west-2.amazonaws.com/xgboost-nightly-builds/federated-secure/xgboost-2.2.0.dev0%2B4601688195708f7c31fcceeb0e0ac735e7311e61-py3-none-manylinux_2_28_x86_64.whl

.. note::

   xgboost のビルド環境は、Python < 3.12 を必要とする特定の numpy バージョンに依存する場合があります。

または、最新の XGBoost ビルドを取得する必要がある場合は、

.. code-block:: bash

    pip install https://s3-us-west-2.amazonaws.com/xgboost-nightly-builds/federated-secure/`curl -s https://s3-us-west-2.amazonaws.com/xgboost-nightly-builds/federated-secure/meta.json | grep -o 'xgboost-2\.2.*whl'|sed -e 's/+/%2B/'`

水平セキュア学習には ``TenSEAL`` パッケージが必要です。

.. code-block:: bash

    pip install tenseal

**nvflare** プラグインを使用する場合、垂直セキュア学習には ``ipcl_python`` パッケージが必要です。 **cuda_paillier** プラグインを使用する場合、このパッケージは不要です。

.. code-block:: bash

    pip install ipcl-python

このパッケージは PyPI では Python 3.8 用のみ提供されています。他のバージョンの Python では、GitHub からインストールする必要があります。

.. code-block:: bash

    pip install git+https://github.com/intel/pailliercryptolib_python.git@development

システム環境
----------------
セキュア学習をサポートするため、いくつかの準同型暗号ライブラリが使用されます。これらのライブラリは Intel CPU または NVIDIA GPU を必要とします。

Linux が推奨 OS です。Ubuntu 22.4 で広範にテストされています。

GPU 学習には次の Docker イメージが推奨されます。

::

    nvcr.io/nvidia/pytorch:24.03-py3

暗号化プラグインのビルド
----------------------------

セキュア学習には暗号化プラグインが必要であり、これはご自身の環境に合わせてソースコードから
ビルドする必要があります。

プラグインをビルドするには、https://github.com/NVIDIA/NVFlare から NVFlare のソースコードをチェックアウトし、
:github_nvflare_link:`this document <integration/xgboost/encryption_plugins/README.md>` の手順に従ってください。

.. _xgb_provisioning:

NVFlare のプロビジョニング
------------------------------
水平セキュア学習では、NVFlare システムを準同型暗号コンテキスト付きでプロビジョニングする必要があります。これには ``project.yml`` の HEBuilder を使用します。
設定例は :github_nvflare_link:`secure_project.yml <examples/advanced/cifar10/cifar10-real-world/workspaces/secure_project.yml>` にあります。

以下は HEBuilder を含む ``secure_project.yml`` ファイルの抜粋です。

.. code-block:: yaml

    api_version: 3
    name: secure_project
    description: NVIDIA FLARE sample project yaml file for CIFAR-10 example

    participants:

    ...

    builders:
    - path: nvflare.lighter.impl.workspace.WorkspaceBuilder
        args:
        template_file: master_template.yml
    - path: nvflare.lighter.impl.template.TemplateBuilder
    - path: nvflare.lighter.impl.static_file.StaticFileBuilder
        args:
        config_folder: config
    - path: nvflare.lighter.impl.he.HEBuilder
        args:
        poly_modulus_degree: 8192
        coeff_mod_bit_sizes: [60, 40, 40]
        scale_bits: 40
        scheme: CKKS
    - path: nvflare.lighter.impl.cert.CertBuilder
    - path: nvflare.lighter.impl.signature.SignatureBuilder


データの準備
================
データは、コラボレーションモードに応じて連合 XGBoost 学習用に適切な形式にしておく必要があります。

水平学習
--------------
水平学習では、すべてのクライアント上のデータセットが同じ列（特徴量）を共有している必要があります。各クライアントは異なるデータサンプル（行）を持ちます。

垂直学習
--------------
垂直学習では、すべてのクライアント上のデータセットは異なる列（特徴量）を含みますが、重複する行（データサンプル）を共有している必要があります。ラベル列は通常、デフォルトで site-1（「アクティブパーティ」）に割り当てられます。

垂直分割の前処理の詳細については、 :github_nvflare_link:`Vertical XGBoost Example <examples/advanced/vertical_xgboost>` を参照してください。

XGBoost プラグインの設定
============================
XGBoost はセキュア学習を扱うために暗号化プラグインを必要とします。利用可能なプラグインは 2 つあります。

- **cuda_paillier** : デフォルトのプラグインです。このプラグインは暗号演算に GPU を使用します。
- **nvflare** : このプラグインはデータをローカルの NVFlare プロセスに転送して暗号化を行います。

.. note::

   すべてのクライアントが同じプラグインを使用しなければなりません。クライアントごとに異なるプラグインを使用した場合、
   連合 XGBoost の挙動は不定となり、ジョブがクラッシュする可能性があります。

**cuda_paillier** プラグインは、compute capability 7.0 以上をサポートする NVIDIA GPU を必要とします。また、CUDA
12.2 または 12.4 がインストールされている必要があります。詳細は https://developer.nvidia.com/cuda-gpus を参照してください。

同梱される 2 つのプラグインは、垂直セキュア学習においてのみ異なります。水平セキュア学習では、どちらのプラグインも
データを NVFlare に転送して暗号化するという点でまったく同じ動作をします。

学習モード別のプラグイン設定
--------------------------------

垂直（非セキュア）
~~~~~~~~~~~~~~~~~~~~~~
プラグインは不要です。

水平（非セキュア）
~~~~~~~~~~~~~~~~~~~~~~
プラグインは不要です。

垂直セキュア
~~~~~~~~~~~~~~~~
垂直セキュア学習には、どちらのプラグインも使用できます。

暗号演算をより高速に行うために GPU を使用するため、デフォルトの cuda_paillier プラグインが推奨されます。

.. note::

    **cuda_paillier** プラグインは、compute capability 7.0 以上をサポートする NVIDIA GPU を必要とします。詳細は https://developer.nvidia.com/cuda-gpus を参照してください。

ログに次のエラーが表示される場合、GPU が検出されていないか、GPU が要件を満たしていないことを意味します。

::

    CUDA runtime API error no kernel image is available for execution on the device at line 241 in file /my_home/nvflare-internal/processor/src/cuda-plugin/paillier.h
    2024-07-01 12:19:15,683 - SimulatorClientRunner - ERROR - run_client_thread error: EOFError:


この場合、CPU 上で暗号化を実行するために nvflare プラグインを使用できます。これには ipcl-python パッケージが必要です。
プラグインはクライアント上の ``local/resources.json`` ファイルで設定できます。

.. code-block:: json

    {
        "federated_plugin": {
            "name": "nvflare",
            "path": "/opt/libs/libnvflare.so"
        }
    }

ここで **name** はプラグイン名、 **path** はライブラリファイル名を含むプラグインのフルパスです。
**path** は省略可能で、デフォルト値はそのプラグイン用に NVFlare に同梱されているライブラリです。

次の環境変数を使用して、JSON 内の値を上書きできます。

.. code-block:: bash

    export NVFLARE_XGB_PLUGIN_NAME=nvflare
    export NVFLARE_XGB_PLUGIN_PATH=/opt/libs/libnvflare.so

.. note::

   NVFlare シミュレーターで実行する場合、resources.json がサポートされないため、
   プラグインは環境変数を使用して設定する必要があります。

水平セキュア
~~~~~~~~~~~~~~~~
プラグインのセットアップは垂直セキュアと同じです。

このモードでは、すべてのプラグインで tenseal パッケージが必要です。
NVFlare システムのプロビジョニングには tenseal コンテキストを含める必要があります。
詳細は :ref:`xgb_provisioning` を参照してください。

シミュレーターの場合、プロビジョニングで生成された tenseal コンテキストを startup フォルダにコピーする必要があります。

``simulator_workspace/startup/client_context.tenseal``

たとえば、

.. code-block:: bash

    nvflare provision -p secure_project.yml -w /tmp/poc_workspace
    mkdir -p /tmp/simulator_workspace/startup
    cp /tmp/poc_workspace/example_project/prod_00/site-1/startup/client_context.tenseal /tmp/simulator_workspace/startup

server_context.tenseal ファイルは不要です。

ジョブの設定
================
.. _secure_xgboost_controller:

コントローラー
------------------

サーバー側では、workflows に次のコントローラーを設定する必要があります。

``nvflare.app_opt.xgboost.histogram_based_v2.fed_controller.XGBFedController``

XGBoost の学習はクライアント上で実行されますが、すべてのクライアントが同じ設定を共有するように、パラメータはサーバー側で設定します。
XGBoost のパラメータは https://xgboost.readthedocs.io/en/stable/python/python_intro.html#setting-parameters で定義されています。

- **num_rounds** : 学習ラウンド数。
- **data_split_mode** : XGBoost の data_split_mode パラメータと同じで、水平の場合は 0、垂直の場合は 1 です。
- **secure_training** : true の場合、XGBoost はプラグインを使用してセキュアモードで学習します。
- **xgb_params** : この dict で定義された学習パラメータは、ブーストパラメータである **params** として XGBoost に渡されます。
- **xgb_options** : この dict には XGBoost に渡すその他のオプションパラメータが含まれます。現在は **early_stopping_rounds** のみがサポートされています。
- **client_ranks** : クライアント名をランクにマッピングする dict です。

エグゼキューター
--------------------

クライアント側では、executors に次のエグゼキューターを設定する必要があります。

``nvflare.app_opt.xgboost.histogram_based_v2.fed_executor.FedXGBHistogramExecutor``

エグゼキューターに必要なパラメータは 1 つだけです。

- **data_loader_id** : データローダーのコンポーネント ID

データローダー
------------------

クライアント側では、components にデータローダーを設定する必要があります。データが前処理済みであれば CSVDataLoader を使用できます。たとえば、

.. code-block:: json

    {
        "id": "dataloader",
        "path": "nvflare.app_opt.xgboost.histogram_based_v2.csv_data_loader.CSVDataLoader",
        "args": {
            "folder": "/opt/dataset/vertical_xgb_data"
        }
    }


データに特別な処理が必要な場合は、カスタムローダーを実装できます。ローダーは XGBDataLoader インターフェースを実装する必要があります。


ジョブの例
==============

垂直学習
--------------

以下は垂直セキュア学習ジョブの設定ファイルです。暗号化が不要な場合は、 ``secure_training`` 引数を false に変更するだけです。

.. code-block::

    :caption: config_fed_server.json

    {
        "format_version": 2,
        "num_rounds": 3,
        "workflows": [
            {
                "id": "xgb_controller",
                "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_controller.XGBFedController",
                "args": {
                    "num_rounds": "{num_rounds}",
                    "data_split_mode": 1,
                    "secure_training": true,
                    "xgb_options": {
                        "early_stopping_rounds": 2
                    },
                    "xgb_params": {
                        "max_depth": 3,
                        "eta": 0.1,
                        "objective": "binary:logistic",
                        "eval_metric": "auc",
                        "tree_method": "hist",
                        "nthread": 1
                    },
                    "client_ranks": {
                        "site-1": 0,
                        "site-2": 1
                    }
                }
            }
        ]
    }



.. code-block::

    :caption: config_fed_client.json

    {
        "format_version": 2,
        "executors": [
            {
                "tasks": [
                    "config",
                    "start"
                ],
                "executor": {
                    "id": "Executor",
                    "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_executor.FedXGBHistogramExecutor",
                    "args": {
                        "data_loader_id": "dataloader"
                    }
                }
            }
        ],
        "components": [
            {
                "id": "dataloader",
                "path": "nvflare.app_opt.xgboost.histogram_based_v2.csv_data_loader.CSVDataLoader",
                "args": {
                    "folder": "/opt/dataset/vertical_xgb_data"
                }
            }
        ]
    }


水平学習
--------------

水平学習の設定は、 ``data_split_mode`` が 0 であることと、データローダーが水平分割データを指す必要があることを除き、垂直学習と同じです。

.. code-block:: json
   :caption: config_fed_server.json

    {
        "format_version": 2,
        "num_rounds": 3,
        "workflows": [
            {
                "id": "xgb_controller",
                "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_controller.XGBFedController",
                "args": {
                    "num_rounds": "{num_rounds}",
                    "data_split_mode": 0,
                    "secure_training": true,
                    "xgb_options": {
                        "early_stopping_rounds": 2
                    },
                    "xgb_params": {
                        "max_depth": 3,
                        "eta": 0.1,
                        "objective": "binary:logistic",
                        "eval_metric": "auc",
                        "tree_method": "hist",
                        "nthread": 1
                    },
                    "client_ranks": {
                        "site-1": 0,
                        "site-2": 1
                    },
                    "in_process": true
                }
            }
        ]
    }




.. code-block:: json
   :caption: config_fed_client.json

    {
        "format_version": 2,
        "executors": [
            {
                "tasks": [
                    "config",
                    "start"
                ],
                "executor": {
                    "id": "Executor",
                    "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_executor.FedXGBHistogramExecutor",
                    "args": {
                        "data_loader_id": "dataloader",
                        "in_process": true
                    }
                }
            }
        ],
        "components": [
            {
                "id": "dataloader",
                "path": "nvflare.app_opt.xgboost.histogram_based_v2.csv_data_loader.CSVDataLoader",
                "args": {
                    "folder": "/data/xgboost_secure/dataset/horizontal_xgb_data"
                }
            }
        ]
    }

学習済みモデル
==================
学習済みモデルを使って学習を継続するには、そのモデルを ``custom/model.json`` というパスと名前で
ジョブフォルダに配置します。

すべてのサイトが同じ ``model.json`` を共有する必要があります。同じデータセットでの以前の学習結果を入力モデルとして使用できます。

学習済みモデルが検出されると、NVFlare はログに次の行を出力します。

::

    INFO - Pre-trained model is used: /tmp/nvflare/poc/example_project/prod_00/site-1/startup/../996ac44f-e784-4117-b365-24548f1c490d/app_site-1/custom/model.json


パフォーマンスチューニング
==============================
タイムアウト
----------------
セキュア学習では、HE 演算が非常に低速です。大きなデータセットを使用する場合、いくつかのタイムアウト値を
調整する必要があります。

XGBoost のメッセージは、Reliable Messages
(:class:`ReliableMessage<nvflare.apis.utils.reliable_message.ReliableMessage>`) を使用してクライアントとサーバーの間で転送されます。エグゼキューターの引数にある次のパラメータが
タイムアウトの挙動を制御します。

    - **per_msg_timeout** : 各メッセージのタイムアウト（秒）。
    - **tx_timeout** : トランザクション全体のタイムアウト（秒）。これは、すべての再試行を含めた応答待ちの合計時間です。

.. code-block::
   :caption: config_fed_client.json

    {
        "format_version": 2,
        "executors": [
            {
                "tasks": [
                    "config",
                    "start"
                ],
                "executor": {
                    "id": "Executor",
                    "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_executor.FedXGBHistogramExecutor",
                    "args": {
                        "data_loader_id": "dataloader",
                        "per_msg_timeout": 300.0,
                        "tx_timeout": 900.0,
                        "in_process": true
                    }
                }
            }
        ],
        ...
    }

クライアント数
------------------
デフォルトの設定では 20 クライアントまでしか扱えません。学習により多くのクライアントが参加する場合は、このパラメータを調整する必要があります。

.. code-block::
   :caption: config_fed_client.json

    {
        "format_version": 2,
        "num_rounds": 3,
        "rm_max_request_workers": 100,
        ...
    }


追加のリソース
==================

- `NVIDIA FLARE Documentation <https://nvflare.readthedocs.io/>`_
- `XGBoost Documentation <https://xgboost.readthedocs.io/>`_
- `GPU XGBoost Blog <https://developer.nvidia.com/blog/gradient-boosting-decision-trees-xgboost-cuda/>`_
- `NVFlare Secure XGBoost Blog <https://developer.nvidia.com/blog/security-for-data-privacy-in-federated-learning-with-cuda-accelerated-homomorphic-encryption-in-xgboost/>`_
