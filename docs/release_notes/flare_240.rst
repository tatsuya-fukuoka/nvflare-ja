**************************
FLARE v2.4.0 の新機能
**************************

ユーザビリティの改善
======================

Client API
~~~~~~~~~~
新しい Client API を導入しました。これは、集中学習のコードからフェデレーテッドラーニングのコードへの変換プロセスを効率化するものです。
Client API を使用する場合、コードの再構成や新しいクラスの実装は不要で、わずか数行のコード変更だけで済みます。
ユーザーは既存の集中型ディープラーニングコードにこの小さな変更を加えるだけで、簡単にフェデレーテッドラーニングのコードへと変換できます。
PyTorch-Lightning については緊密な統合を提供しており、さらに少ないコード変更で済みます。
さらに、Client API によってユーザーが FLARE 固有の概念を深く理解する必要性が大幅に減り、全体的なユーザー体験の簡素化に役立ちます。

以下は、クライアントのトレーナーで Client API を使用する際の一般的なパターンの簡単な例です。

.. code-block:: python

    # import nvflare client API
    import nvflare.client as flare

    # initialize NVFlare client API
    flare.init()

    # run continuously when launching once
    while flare.is_running():

      # receive FLModel from NVFlare
      input_model = flare.receive()

      # loads model from NVFlare
      net.load_state_dict(input_model.params)

      # perform local training and evaluation on received model
      {existing centralized deep learning code} ...

      # construct output FLModel
      output_model = flare.FLModel(
          params=net.cpu().state_dict(),
          metrics={"accuracy": accuracy},
          meta={"NUM_STEPS_CURRENT_ROUND": steps},
      )

      # send model back to NVFlare
      flare.send(output_model)

Client API に関するより詳細な情報については、:ref:`client_api` のドキュメントおよび :github_nvflare_link:`サンプル <examples/hello-world/ml-to-fl>` を参照してください。

サードパーティ統合パターン
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
既存の ML/DL 学習システム基盤が存在するために、学習ロジックを FLARE のクライアント側へ移行しようとする際に課題に直面するケースがあります。
2.4.0 リリースでは、サードパーティ統合パターンを導入しました。これにより、FLARE システムとサードパーティの外部学習システムが、緊密に統合されたシステムを必要とせずにモデルパラメータをシームレスに交換できるようになります。

詳細は :ref:`3rd_party_integration` のドキュメントを参照してください。


Job テンプレートと CLI
~~~~~~~~~~~~~~~~~~~~~~~~
新たに追加された Job テンプレートは、ジョブ構成の作成および調整のプロセスを改善するために設計された、事前定義済みのジョブ構成です。
新しい Job CLI を使用することで、ユーザーは既存の Job テンプレートを簡単に活用し、必要に応じて変更を加え、新しいテンプレートを生成できます。
さらに Job CLI は、Admin コンソールを起動することなく、コマンドラインから直接ジョブを送信する便利な方法も提供します。

``nvflare job list_templates|create|submit|show_variables``

また、よく使われる構成のために作成し、継続的に拡充している :github_nvflare_link:`Job テンプレートディレクトリ <job_templates>` もぜひご覧ください。
Job テンプレートと Job CLI に関するより詳細な情報については、:ref:`job_cli` のドキュメントおよび :github_nvflare_link:`CLI チュートリアル <examples/tutorials/nvflare_cli.ipynb>` を参照してください。

ModelLearner
~~~~~~~~~~~~
ModelLearner は、Learner パターンを必要とするケースにおいて、簡素化されたユーザー体験を提供するために導入されました。
ユーザーは、重み、オプティマイザ、メトリクス、メタデータを含む FLModel オブジェクトのみを扱い、FLARE 固有の概念はユーザーから隠蔽されたままとなります。
ModelLearner は ``train()``、``validate()``、``submit_model()`` といった標準的な学習用の関数を定義しており、サブクラス化することで簡単に適応させることができます。

詳細については、:ref:`model_learner` のドキュメントと、:github_nvflare_link:`ModelLearner <nvflare/app_common/abstract/model_learner.py>` および
:github_nvflare_link:`FLModel <nvflare/app_common/abstract/fl_model.py>` の API 定義を参照してください。

ステップバイステップのサンプルシリーズ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ユーザーが FLARE をすばやく使い始められるように、Jupyter Notebook を用いた包括的な :github_nvflare_link:`ステップバイステップのサンプルシリーズ <examples/hello-world/step-by-step>` を導入しました。
従来のサンプルとは異なり、各ステップバイステップのサンプルでは一貫性のために 2 つのデータセット、すなわち画像データには CIFAR10、表形式データには HIGGS データセットのみを使用します。
各サンプルは前のサンプルの上に積み重ねる形で、さまざまな機能、ワークフロー、API を紹介しており、ユーザーは FLARE の機能を包括的に理解できます。

**CIFAR10 のサンプル:**

- image_stats: CIFAR10 のフェデレーテッド統計 (ヒストグラム)。
- sag: Client API を用いた PyTorch による scatter and gather (SAG) ワークフロー。
- sag_deploy_map: deploy_map 構成を用いた scatter and gather ワークフロー。Client API を使用して異なるサイトへアプリをデプロイします。
- sag_model_learner: ModelLearner を使ったクライアントコードの書き方を示す scatter and gather ワークフロー。
- sag_executor: クライアント側の Executor の書き方を示す scatter and gather ワークフロー。
- sag_mlflow: scatter & gather ワークフローにおける、Client API を用いた MLflow の実験トラッキングログ。
- sag_he: Client API と POC の -he モードを用いた準同型暗号。
- cse: Client API を用いたクロスサイト評価。
- cyclic: サーバー側 Controller による cyclic weight transfer ワークフロー。
- cyclic_ccwf: クライアント側 Controller によるクライアント制御型の cyclic weight transfer ワークフロー。
- swarm: Client API を用いたスウォームラーニングとクライアント側クロスサイト評価。

**HIGGS のサンプル:**

- tabular_stats: フェデレーテッド統計による表形式データのヒストグラム計算。
- scikit_learn: 表形式データに対するフェデレーテッド線形モデル (二値分類のロジスティック回帰) の学習。
- sklearn_svm: 表形式データに対するフェデレーテッド SVM モデルの学習。
- sklearn_kmeans: 表形式データに対するフェデレーテッド k-Means クラスタリング。
- xgboost: bagging コラボレーションによる表形式データに対するフェデレーテッド水平 xgboost 学習。

ストリーミング API
------------------
大規模言語モデル (LLM) をサポートするため、2.4.0 リリースでは、gRPC が課す 2 GB のサイズ上限を超えるオブジェクトの転送を容易にするストリーミング API を導入しました。
大きなオブジェクトを扱うために設計された新しいストリーミングレイヤーの追加により、大きなモデルを 1M のチャンクに分割してターゲットへストリーミングできるようになりました。
Object、Bytes、File、Blob 向けの組み込みストリーマーを提供しており、異なるエンドポイント間での効率的なオブジェクトストリーミングに対する多用途なソリューションとなります。

詳細については :mod:`nvflare.fuel.f3.stream_cell` の API を、FLARE で大きなモデルを扱う際の知見については :ref:`notes_on_large_models` のドキュメントを参照してください。

フェデレーテッドラーニングワークフローの拡張
--------------------------------------------
2.4.0 リリースでは、既存のサーバー側制御ワークフローの代替として :ref:`client_controlled_workflows` を導入しました。

サーバーサイド制御ワークフロー
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- サーバーは、学習プロセス、ジョブ管理、および最終的なモデルの重みの取り扱いについて、すべてのクライアントから信頼されています
- サーバー Controller がジョブのライフサイクル (クライアントサイトの健全性、ジョブステータスの監視など) を管理します
- サーバー Controller が学習プロセス (タスク割り当て、モデル初期化、集約、分散された最終モデルの取得など) を管理します

クライアントサイド制御ワークフロー
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- クライアントは学習プロセスの取り扱いについてサーバーを信頼しません。代わりに、タスク割り当て、モデル初期化、集約、最終モデルの配布はクライアントによって処理されます。
- サーバー Controller は依然としてジョブのライフサイクル (クライアントサイトの健全性、ジョブステータスの監視など) を管理します
- **セキュアメッセージング:** ピアツーピアのクライアントは TLS 暗号化を用いてメッセージを交換します。送信者は受信した証明書から受信者の公開鍵を使用し、AES256 鍵でメッセージを暗号化します。
  送信者とクライアントのみがメッセージを閲覧できます。クライアント間に直接の接続がなく、メッセージがサーバー経由でルーティングされる場合でも、サーバーはメッセージを復号できません。

一般的によく使われる 3 種類のクライアント側制御ワークフローが提供されます。

- :ref:`ccwf_cyclic_learning`: モデルがクライアントからクライアントへ受け渡されます。
- :ref:`ccwf_swarm_learning`: クライアント側 Controller およびアグリゲータとしてクライアントをランダムに選出し、その上で FedAvg による Scatter and Gather を実行します。
- :ref:`ccwf_cross_site_evaluation`: クライアントが他サイトのモデルを評価できるようにします。

これらのクライアント制御ワークフローを使用したサンプルについては、:github_nvflare_link:`スウォームラーニング <examples/advanced/swarm_learning>` および :github_nvflare_link:`クライアント制御型 cyclic <examples/hello-world/step-by-step/cifar10/cyclic_ccwf>` を参照してください。

MLFlow と Weights & Biases による実験トラッキングのサポート
------------------------------------------------------------
MLFlow および Weights & Biases のシステムによる実験トラッキングのサポートを拡充しました。
これらの機能に関する詳細なドキュメントは :ref:`experiment_tracking` にあり、サンプルは
:github_nvflare_link:`MLFlow <examples/advanced/experiment-tracking/mlflow>` および
:github_nvflare_link:`wandb <examples/advanced/experiment-tracking/wandb>` による FL 実験トラッキングにあります。

構成の強化
--------------------------

複数の構成ファイル形式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
2.4.0 リリースでは、複数の構成形式のサポートを追加しました。
このリリース以前は、構成ファイル形式は JSON のみであり、柔軟ではあるものの、コメント、変数の置換、継承といった便利な機能を欠いていました。

2 つの新しい構成形式を追加しました。

- `Pyhocon <https://github.com/chimpler/pyhocon>`_ - JSON のバリアントであり、多くの望ましい機能を備えた Python 向けの HOCON (Human-Optimized Config Object Notation) パーサー
- `OmegaConf <https://omegaconf.readthedocs.io/en/2.3_branch/>`_ - YAML ベースの階層的構成

ユーザーは単一の形式を使用することも、config_fed_client.conf と config_fed_server.json のように複数の形式を組み合わせることもできます。
複数の構成形式が併存する場合、次の検索順に基づいて優先的に使用されます: .json -> .conf -> .yml -> .yaml

ジョブ構成ファイル処理の改善
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
- 変数解決 - 構成ファイル内でのユーザー定義変数の定義と変数参照
- 組み込みシステム変数 - 構成ファイル内で使用できる事前定義済みのシステム変数
- OS 環境変数 - OS の環境変数をドル記号で参照可能
- パラメータ化された変数定義 - 再利用可能で、異なる具体的な構成へと解決できる構成テンプレートの作成

詳細は :ref:`configurations` のドキュメントを参照してください。

POC コマンドのアップグレード
----------------------------
POC コマンドを拡張し、ユーザーが実際のデプロイプロセスへ一歩近づけるようにしました。
この変更により、ユーザーはローカルでデプロイオプションを試すことができ、実験と本番の両方で同じ project.yaml ファイルを使用できます。

POC コマンドのモードは、本番環境のシミュレーションをより適切に反映するため、「ローカル、非セキュア」から「ローカル、セキュア、本番」へと変更されました。
最後に、POC コマンドは一般的な構文により合致するようになりました。
``nvflare poc -<action>`` => ``nvflare poc <action>``

詳細は :ref:`poc_command` のドキュメントまたは :github_nvflare_link:`チュートリアル <examples/tutorials/setup_poc.ipynb>` を参照してください。

セキュリティの強化
---------------------

安全でないコンポーネントの検出
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ユーザーは安全でないコンポーネントのチェッカーを定義できるようになり、構築されるコンポーネントを検証するためにそのチェッカーが呼び出されます。
チェッカーはコンポーネントの検証に失敗した場合に UnsafeJob 例外を送出し、それによってジョブが中止されます。

詳細については :ref:`unsafe_component_detection` のドキュメントを参照してください。

イベントベースのセキュリティプラグイン
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
ジョブレベルの機能認可のためのプラグインを構築するのに使用できる、追加の FL イベントを導入しました。

詳細については、:ref:`site_specific_auth` のドキュメント、およびこれらの機能に関する詳細を示す
:github_nvflare_link:`カスタム認証のサンプル <examples/advanced/custom_authentication>` を参照してください。

FL HUB: 階層的統合ブリッジ
---------------------------------------
FL HUB は、複数の FLARE システムが階層的に連携して動作することをサポートするために設計された、新しい実験的機能です。
フェデレーテッドコンピューティングでは、エッジデバイスの数が通常は非常に多い一方でサーバーは 1 台だけということが多く、パフォーマンス上の問題を引き起こす可能性があります。
この問題に対する解決策の 1 つが階層的な FLARE システムの利用であり、階層化された FLARE システムが互いに接続してツリー状の構造を形成します。
各クライアント (エッジデバイス) のリーフは自身のサーバーにのみ接続し、そのサーバーは同時に上位階層の FLARE システムのクライアントとしても機能します。

想定されるユースケースの 1 つはグローバルスタディであり、クライアントマシンが異なる地域にまたがって配置される場合です。
各地域のクライアントマシンをその地域内の単一の FL サーバーにのみ接続させるのではなく、FL HUB により、よりパフォーマンスの高い階層型のマルチサーバー構成を実現できます。

FL Hub の詳細については、:ref:`階層的統合ブリッジ <hierarchy_unification_bridge>` のドキュメントおよび :github_nvflare_link:`コード <nvflare/app_common/hub>` を参照してください。

その他の機能
--------------
- FLARE API のパリティ

  - FLARE API は Admin Client と同じ API セットを備えるようになりました。
  - ユーザーは Python API やノートブックからほぼすべてのコマンドを使用できます。

- Docker のサポート

  - NVFLARE のクラウド CSP 起動スクリプトは、VM へのデプロイに加えて Docker コンテナでのデプロイをサポートするようになりました。
  - provision コマンドは、対話的な docker run に加えて、デタッチされた docker run もサポートするようになりました。

- Flare Dashboard

  - 2.4.0 より前では、Flare ダッシュボードは Docker コンテナ内でしか実行できませんでした。
  - 2.4.0 では、Flare ダッシュボードを開発用に Docker なしでローカル実行できるようになりました。

- 学習なしでのモデル評価の実行

  - 2.4.0 リリースでは、学習を再実行することなくクロスバリデーションを実行できるようになりました。
  - :github_nvflare_link:`学習なしでのクロスサイトバリデーションの実行 <examples/hello-world/hello-numpy-cross-val#run-cross-site-validation-using-the-previous-trained-results>` のサンプルを参照してください。

- 通信の強化

  - gRPC のタイムアウトを置き換えるため、クライアントのジョブプロセスとサーバーの親プロセスの間にアプリケーション層の ping を追加しました。
    以前は、gRPC のタイムアウトを長く設定しすぎると、クラウドプロバイダー (Azure Cloud など) が 4 分後に接続を切断してしまうことに気づいていました。
    一方でタイムアウトの設定が短すぎる場合 (2 分など)、下層の gRPC が ping が多すぎると報告してしまいます。
    アプリケーションレベルの ping はこの両方の問題を回避し、サーバー/クライアントがプロセスの状態を確実に把握できるようにします。
  - FLARE は gRPC ベースの通信のために 2 つのドライバ、すなわち asyncio (AIO) 版と通常 (非 AIO) 版の gRPC ライブラリを提供します。
    AIO gRPC の顕著な利点の 1 つは、サーバー側でより多くの同時接続を処理できる能力です。
    しかし、AIO gRPC はクライアント側で厳しいネットワーク状況においてクラッシュする可能性がある一方、非 AIO gRPC はより安定しています。
    そのため FLARE 2.4.0 では、より高い安定性のためにデフォルト構成で非 AIO 版の gRPC ライブラリを使用します。

    - ドライバの選択を変更するには、ワークスペースの local ディレクトリにある ``comm_config.json`` を更新し、
      ``use_aio_grpc`` の設定変数を設定します。

新しいサンプル
--------------

フェデレーテッド大規模言語モデル (LLM) のサンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

フェデレーテッド LLM の扱い方を示すために、いくつかのサンプルを追加しました。

- :github_nvflare_link:`パラメータ効率的ファインチューニング <integration/nemo/examples/peft>`: NeMo の PEFT 手法を利用して LLM を下流タスクに適応させます。
- `プロンプトチューニングのサンプル <https://github.com/NVIDIA/NVFlare/tree/dev_deprecated/integration/nemo/examples/prompt_learning>`__: FLARE を NeMo と組み合わせてプロンプト学習を行います。
- :github_nvflare_link:`教師ありファインチューニング (SFT) <integration/nemo/examples/supervised_fine_tuning>`: 教師ありデータ上で LLM の全パラメータをファインチューニングします。
- :github_nvflare_link:`HuggingFace SFT Trainer による LLM チューニング <examples/advanced/llm_hf>`: FLARE を HuggingFace の trainer と組み合わせて LLM のチューニングタスクを行います。

垂直フェデレーテッド XGBoost
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
`XGBoost <https://github.com/dmlc/xgboost>`_ の 2.0 リリースにより、:github_nvflare_link:`垂直 xgboost のサンプル <examples/advanced/vertical_xgboost>` を示すことができるようになりました。
Private Set Intersection と XGBoost の新しいフェデレーテッドラーニングサポートを使用して、垂直分割された HIGGS データ (各サイトが重複するデータサンプルを共有しつつ、異なる特徴量を持つ) に対する分類を行います。

グラフニューラルネットワーク (GNN)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
GraphSage を用いた 2 つのサンプルを追加し、:github_nvflare_link:`帰納的学習を用いたグラフデータセット上でのフェデレーテッド GNN <examples/advanced/gnn#federated-gnn-on-graph-dataset-using-inductive-learning>` の学習方法を示しました。

**タンパク質分類:** 遺伝子オントロジーに基づく細胞機能から、タンパク質の役割を分類します。
使用しているデータセットは PPI (`protein-protein interaction <http://snap.stanford.edu/graphsage/#code>`_) グラフで、各グラフは特定のヒト組織を表します。
タンパク質間相互作用 (PPI) データセットは、グラフベースの機械学習タスク、特にバイオインフォマティクスの分野でよく使用されます。
このデータセットはタンパク質間の相互作用をグラフとして表現しており、ノードはタンパク質を、エッジはそれらの間の相互作用を表します。

**金融取引の分類:** 与えられた取引が合法か違法かを分類します。
この金融アプリケーションでは、`Elliptic++ <https://github.com/git-disl/EllipticPlusPlus>`_ データセットを使用します。このデータセットは
203k 件のビットコイン取引と 822k 件のウォレットアドレスから構成されており、グラフデータを活用することでビットコインネットワークにおける不正取引の検出と違法な
アドレス (行為者) の検出の両方を可能にします。詳細については、この `論文 <https://arxiv.org/pdf/2306.06108.pdf>`_ を参照してください。

金融アプリケーションのサンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
金融アプリケーションにおける不正検知の実施方法を示すため、`finance データセット <https://www.kaggle.com/datasets/mlg-ulb/creditcardfraud>`_ を用いて
XGBoost をさまざまな方法で使用しフェデレーテッドな形でモデルを学習する :github_nvflare_link:`サンプル <examples/advanced/finance>` を導入しました。
XGBoost による垂直および水平フェデレーテッドラーニングの両方を、ヒストグラムベースおよびツリーベースのアプローチとあわせて示しています。

KeyCloak サイト認証の統合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
FLARE はサードパーティの認証メカニズムに依存せず、各クライアントは独自の認証システムを持つことができます。
ここでは KeyCloak を使用して、FLARE のサイト固有認証のサポートを示します。
:github_nvflare_link:`KeyCloak サイト認証の統合 <examples/advanced/keycloak-site-authentication>` のサンプルは、admin ユーザーがジョブを送信・実行するために追加のユーザー認証を必要とするように構成されています。


**********************************
2.4.0 への移行: 注意点とヒント
**********************************

FLARE 2.4.0 では、いくつかの API と挙動の変更が導入されています。この移行ガイドは、以前の NVFLARE バージョンから現在のバージョンへ移行するのに役立ちます。

ジョブフォーマット: meta.json
-----------------------------
FLARE 2.4.0 では、ユーザーはジョブ内に meta.json 構成ファイルを定義する必要があります。
レガシーな app 定義は、デプロイマップと任意の数の app フォルダ (config/ と custom/ を含む) を持つ meta.json ファイルを含むジョブフォーマットへ更新する必要があります。
以下は、単一の app を持つ基本的なジョブ構造です。

.. code-block:: shell

  ├── my_job
  │   ├── app
  │   │   ├── config
  │   │   │   ├── config_client.json
  │   │   │   └── config_server.json
  │   │   └── custom
  │   └── meta.json

以下は、適宜編集できるデフォルトの meta.json です。

.. code-block:: json

  {
    "name": "my_job",
    "resource_spec": {},
    "min_clients" : 2,
    "deploy_map": {
      "app": [
        "@ALL"
      ]
    }
  }

FLARE API の同等性
------------------
FLARE 2.3.0 では、再設計された FLAdminAPI として FLARE API の初期バージョンが実装されましたが、含まれていたのは機能のサブセットのみでした。
FLARE 2.4.0 では、FLAdminAPI を終息できるようにするため、FLARE API を拡張して FLAdminAPI の残りの機能を含めました。

追加された機能の詳細については :ref:`FLARE API への移行 <migrating_to_flare_api>` を参照してください。

タイムアウト処理
~~~~~~~~~~~~~~~~

2.4.0 リリースでは、Admin Server が FL クライアントと通信して応答を待つコマンドに対するタイムアウト処理が改善されました。
以前は Admin Server 上で固定のグローバルなタイムアウト値が使用されていましたが、コマンドに長い時間がかかる場合にはこの値では不十分なことがありました
(例えば ``cat server log.txt`` コマンドは、大きなログファイルの転送に時間がかかることがあります)。
この場合、ユーザーは ``set_timeout`` コマンドを使って Admin Server のデフォルトのタイムアウト値を変更できましたが、このコマンドにはグローバルであり、すべてのユーザーに影響するという欠点がありました。
このコマンドのグローバルな影響により、あるユーザーが非常に小さなタイムアウト値を設定すると、すべてのユーザーのコマンドが失敗する可能性がありました。

これに対処するため、``set_timeout`` コマンドはセッション固有となるよう変更されました。
さらに、そのセッションについて Admin Server のデフォルトのタイムアウトの使用に戻すための新しい ``unset_timeout`` コマンドが追加されました。

``show_stats`` と ``show_errors`` の変更
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

古い構造では、サーバーの結果 dict は全体の結果 dict のトップレベルに直接配置される一方、各クライアントの結果 dict はクライアント名をキーとする項目として配置されていました。
サーバーとクライアントの結果の間で一貫性を持たせるため、サーバーの結果を "server" をキーとする項目として配置するよう変更しました。
FLAdminAPI の古い戻り値の構造に基づくコードがある場合は、それに応じて更新してください。

.. code-block::

    {
      "server": { # new "server" key for server result dict
        "ScatterAndGather": {
          "tasks": {
            "train": [
              "site-1",
              "site-2"
            ]
          },
          "phase": "train",
          "current_round": 2,
          "num_rounds": 50
        },
        "ServerRunner": {
          "job_id": "3ad5bdef-db12-4ffb-9362-0ff163973f7d",
          "status": "started",
          "workflow": "scatter_and_gather"
        }
      },
      "site-1": {
        "ClientRunner": {
          "job_id": "3ad5bdef-db12-4ffb-9362-0ff163973f7d",
          "current_task_name": "None",
          "status": "started"
        }
      },
      "site-2": {
        "ClientRunner": {
          "job_id": "3ad5bdef-db12-4ffb-9362-0ff163973f7d",
          "current_task_name": "train",
          "status": "started"
        }
      }
    }

POC コマンドのアップグレード
----------------------------
POC コマンドは 2.4.0 でアップグレードされました。

- アクションコマンドから ``--`` を削除し、サブコマンドへ変更
- POC は「本番モード」を使用するようになり、admin のユーザー名は以前のリリースの "admin" ではなく "admin@nvidia.com" になりました。
- 新しい ``-d`` (docker) および ``-he`` (準同型暗号) オプション
- ``nvflare poc prepare`` は POC ワークスペースの場所を保存する ``.nvflare/config.conf`` を生成し、これは環境変数 ``NVFLARE_POC_WORKSPACE`` より優先されます
- 以前のバージョンでは、スタートアップキットはデフォルトの POC ワークスペース ``/tmp/nvflare/poc`` の直下に配置されていました。2.4.0 では、本番のプロビジョニングのデフォルト構造に従うため、スタートアップキットは ``/tmp/nvflare/poc/example_project/prod_00/`` の下に配置されます。
- マルチ組織・マルチロールのサポート

.. code-block:: none

  nvflare poc -h
  usage: nvflare poc [-h] [--prepare] [--start] [--stop] [--clean] {prepare,prepare-jobs-dir,start,stop,clean} ...

  optional arguments:
    -h, --help            show this help message and exit
    --prepare             deprecated, suggest use 'nvflare poc prepare'
    --start               deprecated, suggest use 'nvflare poc start'
    --stop                deprecated, suggest use 'nvflare poc stop'
    --clean               deprecated, suggest use 'nvflare poc clean'

  poc:
    {prepare,prepare-jobs-dir,start,stop,clean}
                          poc subcommand
      prepare             prepare poc environment by provisioning local project
      prepare-jobs-dir    prepare jobs directory
      start               start services in poc mode
      stop                stop services in poc mode
      clean               clean up poc workspace

詳細は :ref:`poc_command` を参照してください。

セキュアメッセージング
----------------------

:class:`ServerEngineSpec<nvflare.apis.server_engine_spec.ServerEngineSpec>` および
:class:`ClientEngineExecutorSpec<nvflare.private.fed.client.client_engine_executor_spec.ClientEngineExecutorSpec>` の ``send_aux_request()`` に、新しい ``secure`` 引数が追加されました。

``secure`` は、aux リクエストをセキュアな方法で送信すべきかどうかを決定するオプションのブール値です。
そのようなユースケースの 1 つが、クライアント制御ワークフローのようなセキュアなピアツーピアメッセージングです。

.. code-block:: python

   @abstractmethod
    def send_aux_request(
        self,
        targets: Union[None, str, List[str]],
        topic: str,
        request: Shareable,
        timeout: float,
        fl_ctx: FLContext,
        optional=False,
        secure: bool = False,
    ) -> dict:
        """Send a request to Server via the aux channel.
        Implementation: simply calls the ClientAuxRunner's send_aux_request method.
        Args:
            targets: aux messages targets. None or empty list means the server.
            topic: topic of the request
            request: request to be sent
            timeout: number of secs to wait for replies. 0 means fire-and-forget.
            fl_ctx: FL context
            optional: whether the request is optional
            secure: should the request sent in the secure way
        Returns:
            a dict of reply Shareable in the format of:
                { site_name: reply_shareable }
        """
        pass

統計結果のフォーマット
----------------------
:class:`StatisticsController<nvflare.app_common.workflows.statistics_controller.StatisticsController>` では、
結果の辞書フォーマットは元来、可視化をサポートするために "site" と "dataset" を連結していました。
2.4.0 ではこれが変更され、"site" と "dataset" が結果辞書内でそれぞれ独自のキーを持つようになりました。

``result = {feature: {statistic: {site-dataset: value}}}``

から

``result =  feature: {statistic: {site: {dataset: value}}}}``

へ変更されました。

可視化のニーズを引き続きサポートするため、site-dataset の連結ロジックは代わりに
:class:`Visualization<nvflare.app_opt.statistics.visualization.statistics_visualization.Visualization>` へ移動されました。
