**************************
FLARE v2.5.0 の新機能
**************************

ユーザーエクスペリエンスの改善
==============================
NVFlare 2.5.0 では、エンドツーエンドでの使いやすさを実現するいくつかの新しい API セットを提供しており、研究者やデータ
サイエンティストが FLARE を扱う体験を大きく向上させます。この新しい API は、クライアント、サーバー、およびジョブの構築を、エンドツーエンドで Python らしいユーザー体験とともにカバーします。

Model Controller API
~~~~~~~~~~~~~~~~~~~~
新しい :ref:`model_controller` は、新たなフェデレーテッドラーニングワークフローを開発する体験を大幅に簡素化します。ユーザーは
ModelController をサブクラス化するだけで新しいワークフローを開発できます。この新しい API では、FLModel クラスを除き、
NVFlare の構成要素の詳細を知る必要はありません。FLModel はモデルの重み、最適化パラメータ、メタデータを含む単なるデータ構造です。

基本的な Python コードで新しいワークフローを簡単に構築でき、準備ができたら、クライアントとサーバー間の通信には
send_and_wait() という通信関数だけがあれば十分です。

Client API
~~~~~~~~~~
もう 1 つの :ref:`client_api` の実装として、
:class:`InProcessClientAPIExecutor<nvflare.app_common.executors.in_process_client_api_executor.InProcessClientAPIExecutor>` を導入しました。
これは :class:`SubprocessLauncher<nvflare.app_common.launchers.subprocess_launcher.SubprocessLauncher>` を使用する従来の Client API と
同じインターフェースおよび構文を持ちますが、すべての通信がメモリ内で行われる点が異なります。

このインプロセスの Client API を使用して :class:`ScriptExecutor<nvflare.app_common.executors.script_executor.ScriptExecutor>` を構築しており、
これは新しい Job API で直接使用されます。

SubProcessLauncherClientAPI と比較して、インプロセスの Client API はより高い効率性を提供し、構成も容易です。
すべての操作は Executor のメモリ空間内で実行されます。

SubProcessLauncherClientAPI は、別個の学習プロセスが必要となるケースで使用できます。

Job API
~~~~~~~
新しい Job API、すなわち :ref:`fed_job_api` は、Client API および Model Controller API と組み合わせることで、エンドツーエンドで Python らしい
ユーザー体験をもたらします。現行リリース以前は必要だったジョブ構成が、直接自動生成できるようになったため、
ユーザーは構成ファイルを手作業で編集する必要がなくなりました。

新しい Job API の威力を示す多数のサンプルを提供しており、新しいフェデレーテッドラーニングアルゴリズムを試したり、
新しいアプリケーションを作成したりすることが非常に容易になります。

Flower との統合
------------------
NVFlare と `Flower <https://flower.ai/>`_ フレームワークの統合は、Flower のプロジェクトを NVFlare 上でシームレスに実行できるようにすることで、
研究者が両フレームワークの強みを活用できるようにすることを目指しています。Flower と FLARE のシームレスな
統合により、Flower フレームワーク内で作成されたアプリケーションは、いかなる変更も必要とせずに FLARE のランタイム
環境内で難なく動作します。この初期統合はプロセスを効率化し、複雑さを取り除き、
2 つのプラットフォーム間の円滑な相互運用性を確保することで、FL アプリケーション全体の効率性とアクセシビリティを高めます。
詳細は `こちら <https://arxiv.org/abs/2407.00031>`__ を参照してください。hello-world のサンプルは
:github_nvflare_link:`こちら <examples/hello-world/hello-flower>` にあります。

セキュア XGBoost
----------------
XGBoost の最新機能により、準同型暗号を用いたセキュアなフェデレーテッドラーニングのサポートが導入されました。垂直フェデレーテッド
XGBoost 学習では、各サンプルの勾配が暗号化によって保護されるため、ラベル情報が
意図しない当事者に漏洩することはありません。一方、水平フェデレーテッド XGBoost 学習では、ローカルの勾配ヒストグラムが
中央の集約サーバーに知られることはありません。

XGBoost と連携する当社の暗号化プラグインにより、NVFlare は CPU と GPU の両方で、XGBoost モデル学習のためのすべての
セキュアなフェデレーテッド方式をサポートするようになりました。

`nvflare によるフェデレーテッド xgboost のユーザーガイド <https://nvflare.readthedocs.io/en/2.5/user_guide/federated_xgboost.html>`
および :github_nvflare_link:`サンプル <examples/advanced/xgboost_secure>` を参照してください。

Tensorflow のサポート
---------------------
コミュニティからの貢献により、Tensorflow を用いた FedOpt、FedProx、Scaffold のアルゴリズムを追加しました。
コードは :github_nvflare_link:`こちら <nvflare/app_opt/tf>`、サンプルは :github_nvflare_link:`こちら <examples/getting_started/tf>` で確認できます。

FOBS の自動登録
----------------------
NVFlare がメッセージのシリアライズおよびデシリアライズに使用するセキュアな仕組みである FOBS が、新しい自動登録機能によって強化されました。
これらの変更により、ユーザーが登録しなければならないデコンポーザの数が削減されます。変更点は以下のとおりです。

  - デシリアライズ時のデコンポーザの自動登録。デコンポーザのクラスはシリアライズされたデータ内に保存され、デシリアライズ時に
    デコンポーザが自動的に登録されます。コンポーネントがシリアライズされたデータを受け取るだけでシリアライズを行わない場合、
    デコンポーザの登録はもはや不要です。

  - シリアライズ時のデータクラスデコンポーザの自動登録。あるクラスに対するデコンポーザが見つからない場合、FOBS はそのクラスを
    データクラスとして扱い、DataClassDecomposer を登録しようとします。これはほとんどのケースで機能しますが、すべてではありません。


新しいサンプル
--------------
セキュアなフェデレーテッド Kaplan-Meier 解析
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`時間ビニングと準同型暗号によるセキュアなフェデレーテッド Kaplan-Meier 解析のサンプル <examples/advanced/kaplan-meier-he>`
は、2 つの機能を示しています。

  - 時間ビニングと準同型暗号 (HE) によるセキュアな機能を用いる場合と用いない場合の、フェデレーテッドな設定での Kaplan-Meier 生存解析の実施方法。
  - シミュレータモードで HE を実現するためのワークフローを構成するために、Flare の ModelController API を使用する方法。

創薬向けの BioNemo サンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
`BioNeMo <https://www.nvidia.com/en-us/clara/bionemo/>`_ は、創薬のための NVIDIA の生成 AI プラットフォームです。
NVFlare を使用してフェデレーテッドラーニング環境で BioNeMo を実行するサンプルをいくつか含めています。

  - :github_nvflare_link:`タスクフィッティングのサンプル <examples/advanced/bionemo/task_fitting/README.md>` には、ESM-1nv の事前学習済みモデルを使用して、埋め込みの形でタンパク質の学習済み表現を取得する方法を示すノートブックが含まれています。
  - :github_nvflare_link:`下流タスクのサンプル <examples/advanced/bionemo/downstream/README.md>` では、BioNeMo の ESM スタイルのモデルをファインチューニングするための 3 つの異なる下流タスクを示しています。

NR 最適化によるフェデレーテッドロジスティック回帰
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`2 次のニュートン・ラフソン最適化によるフェデレーテッドロジスティック回帰のサンプル <examples/advanced/lr-newton-raphson>`
では、2 次のニュートン・ラフソン最適化を用いたロジスティック回帰による、フェデレーテッドな二値分類の実装方法を示します。

階層的フェデレーテッド統計
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`階層的フェデレーテッド統計 <examples/advanced/federated-statistics/hierarchical_stats>` は、複数の組織が
関与する場合に役立ちます。例えば医療機器のアプリケーションでは、医療機器の使用統計をデバイス、
デバイスをホストするサイト、そして病院やメーカーのそれぞれの観点から見ることができます。
メーカーは、異なるサイトや病院における自社製品 (デバイス) の使用統計を見たいと考えるでしょう。病院は、
異なるメーカーの異なる製品を含む、デバイス全体の統計を見たいと考えるかもしれません。このような場合に、階層的な
フェデレーテッド統計は非常に役立ちます。

FedAvg 早期停止のサンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`FedAvg 早期停止のサンプル <examples/hello-world/hello-fedavg>` は、新しいサーバー側の Model
Controller API を使えば、数行の Python コードで制御条件を変更したりワークフローを調整したりすることが非常に容易であることを示そうとしています。

Tensorflow のアルゴリズムとサンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Tensorflow 向けの FedOpt、FedProx、Scaffold の実装です。

FedBN: ローカルバッチ正規化による非 IID 特徴量上でのフェデレーテッドラーニング
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`FedBN のサンプル <research/fed-bn>` は、異なるデータ分布をまたいでモデルを集約する際の
特徴シフト問題に対処するために設計されたフェデレーテッドラーニングアルゴリズムを紹介しています。

この研究では、モデルを平均化する前に特徴シフトを緩和するために、ローカルバッチ正規化を使用する効果的な手法を提案します。
FedBN と呼ばれるこの結果として得られる方式は、私たちの広範な実験において、古典的な FedAvg と FedProx の両方を上回りました。これらの実験的結果は、
簡略化された設定において FedBN が FedAvg よりも速い収束率を持つことを示す収束解析によって裏付けられています。


エンドツーエンドのフェデレーテッド XGBoost サンプル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:github_nvflare_link:`このサンプル <examples/advanced/finance-end-to-end/xgboost.ipynb>` では、
フェデレーテッドな設定における特徴量エンジニアリング、前処理、学習のエンドツーエンドのプロセスを示そうとしています。
FLARE を使用してフェデレーテッド ETL を実行し、その後に学習を行うことができます。

開発者向けチュートリアルページ
------------------------------
ユーザーが FLARE によるフェデレーテッドラーニングをすばやく学べるように、コードと動画の両方を備えた `チュートリアル Web ページ <https://nvidia.github.io/NVFlare>`_ を作成し、
数分で FL への変換と実行の方法を対話的に学べるようにしました。また、
興味のあるサンプルを簡単に検索して見つけられるように、チュートリアルカタログも作成しました。

**********************************
2.5.0 への移行: 注意点とヒント
**********************************

FLARE 2.5.0 では、いくつかの API と挙動の変更が導入されています。この移行ガイドは、以前の NVFlare バージョンから
現在のバージョンへ移行するのに役立ちます。

"name" を非推奨とし "path" のみを使用
-------------------------------------
2.5.0 では、構成内の "name" フィールドは非推奨となりました。"name" フィールドを "path" に変更し、フルパスを使用する必要があります。
例えば、

.. code-block:: json

  "name": "TBAnalyticsReceiver"

は次のように更新する必要があります。

.. code-block:: json

  "path": "nvflare.app_opt.tracking.tb.tb_receiver.TBAnalyticsReceiver"

XGBoost v1 - v2
---------------

2.5.0 では XGBoost のサポートが強化され、準同型暗号 (HE) を用いたセキュアな学習をサポートするようになりました。また、Controller 側で
XGBoost のパラメータを設定することで、すべてのクライアントが同じパラメータを取得できるようになり、ユーザーインターフェースも簡素化されました。

主な変更点は以下のとおりです。

  - xgboost のパラメータがクライアントの構成からサーバーへ移動しました。
  - 新しい split_mode および secure_training パラメータ
  - 新しい :class:`CSVDataLoader<nvflare.app_opt.xgboost.histogram_based_v2.csv_data_loader.CSVDataLoader>`

2.5.0 向けのサンプル構成ファイル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

config_fed_server.json
""""""""""""""""""""""

.. code-block:: json

  {
      "format_version": 2,
      "num_rounds": 3,
      "workflows": [
          {
              "id": "xgb_controller",
              "path": "nvflare.app_opt.xgboost.histogram_based_v2.fed_controller.XGBFedController",
              "args": {
                  "num_rounds": "{num_rounds}",
                  "split_mode": 1,
                  "secure_training": false,
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

config_fed_client.json
""""""""""""""""""""""

.. code-block:: json

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
              "path": "nvflare.app_opt.xgboost.histogram_based_v2.secure_data_loader.SecureDataLoader",
              "args": {
                  "rank": 0,
                  "folder": "/tmp/nvflare/dataset/vertical_xgb_data"
              }
          }
      ]
  }

シミュレータのワークスペース構造
--------------------------------

2.4.0 では、サーバーとすべてのクライアントが ``simulate_job`` という同じシミュレータワークスペースのルートを共有していました。サーバーと各クライアントは
それぞれ独自の app_XXXX ジョブ定義を持っていましたが、ワークスペースのルートフォルダが同じであるため、モデルファイルの場所が競合する可能性がありました。

.. raw:: html

   <details>
   <summary><a>2.4.0 のフォルダ構造の例</a></summary>

.. code-block:: none

  simulator/
  ├── local
  │   └── log.config
  ├── simulate_job
  │   ├── app_server
  │   │   ├── FL_global_model.pt
  │   │   ├── __init__.py
  │   │   ├── config
  │   │   │   ├── config_fed_client.json
  │   │   │   ├── config_fed_server.json
  │   │   │   ├── config_train.json
  │   │   │   ├── config_validation.json
  │   │   │   ├── dataset_0.json
  │   │   │   └── environment.json
  │   │   ├── custom
  │   │   │   ├── __init__.py
  │   │   │   ├── add_shareable_parameter.py
  │   │   │   ├── client_aux_handler.py
  │   │   │   ├── client_send_aux.py
  │   │   │   ├── client_trainer.py
  │   │   │   ├── fed_avg_responder.py
  │   │   │   ├── model_shareable_manager.py
  │   │   │   ├── print_shareable_parameter.py
  │   │   │   ├── server_aux_handler.py
  │   │   │   ├── server_send_aux.py
  │   │   │   └── supervised_fitter.py
  │   │   ├── docs
  │   │   │   ├── Readme.md
  │   │   │   └── license.txt
  │   │   ├── eval
  │   │   └── models
  │   ├── app_site-1
  │   │   ├── __init__.py
  │   │   ├── config
  │   │   │   ├── config_fed_client.json
  │   │   │   ├── config_fed_server.json
  │   │   │   ├── config_train.json
  │   │   │   ├── config_validation.json
  │   │   │   ├── dataset_0.json
  │   │   │   └── environment.json
  │   │   ├── custom
  │   │   │   ├── __init__.py
  │   │   │   ├── add_shareable_parameter.py
  │   │   │   ├── client_aux_handler.py
  │   │   │   ├── client_send_aux.py
  │   │   │   ├── client_trainer.py
  │   │   │   ├── fed_avg_responder.py
  │   │   │   ├── model_shareable_manager.py
  │   │   │   ├── print_shareable_parameter.py
  │   │   │   ├── server_aux_handler.py
  │   │   │   ├── server_send_aux.py
  │   │   │   └── supervised_fitter.py
  │   │   ├── docs
  │   │   │   ├── Readme.md
  │   │   │   └── license.txt
  │   │   ├── eval
  │   │   ├── log.txt
  │   │   └── models
  │   ├── app_site-2
  │   │   ├── __init__.py
  │   │   ├── config
  │   │   │   ├── config_fed_client.json
  │   │   │   ├── config_fed_server.json
  │   │   │   ├── config_train.json
  │   │   │   ├── config_validation.json
  │   │   │   ├── dataset_0.json
  │   │   │   └── environment.json
  │   │   ├── custom
  │   │   │   ├── __init__.py
  │   │   │   ├── add_shareable_parameter.py
  │   │   │   ├── client_aux_handler.py
  │   │   │   ├── client_send_aux.py
  │   │   │   ├── client_trainer.py
  │   │   │   ├── fed_avg_responder.py
  │   │   │   ├── model_shareable_manager.py
  │   │   │   ├── print_shareable_parameter.py
  │   │   │   ├── server_aux_handler.py
  │   │   │   ├── server_send_aux.py
  │   │   │   └── supervised_fitter.py
  │   │   ├── docs
  │   │   │   ├── Readme.md
  │   │   │   └── license.txt
  │   │   ├── eval
  │   │   ├── log.txt
  │   │   └── models
  │   ├── log.txt
  │   ├── meta.json
  │   └── pool_stats
  │       └── simulator_cell_stats.json
  └── startup
      ├── client_context.tenseal
      └── server_context.tenseal

.. raw:: html

   </details>
   <br />

2.5.0 では、サーバーとすべてのクライアントが、シミュレータワークスペースの下にそれぞれ独自のワークスペースのサブフォルダを持つようになります。``simulator_job``
は各サイトのワークスペース内にあります。これにより各サイトが完全に分離され、モデルファイルが競合することはありません。このワークスペース
構造は、POC の実世界アプリケーションのフォーマットと一貫しています。

.. raw:: html

   <details>
   <summary><a>2.5.0 のフォルダ構造の例</a></summary>

.. code-block:: none

  simulator/
  ├── server
  │   ├── local
  │   │   └── log.config
  │   ├── log.txt
  │   ├── pool_stats
  │   │   └── simulator_cell_stats.json
  │   ├── simulate_job
  │   │   ├── app_server
  │   │   │   ├── FL_global_model.pt
  │   │   │   └── config
  │   │   │       ├── config_fed_client.conf
  │   │   │       └── config_fed_server.conf
  │   │   ├── artifacts
  │   │   │   ├── 39d0b7edb17b437dbf77da2e402b2a4d
  │   │   │   │   └── artifacts
  │   │   │   │       └── running_loss_reset.txt
  │   │   │   └── b10ff3e54b0d464c8aab8cf0b751f3cf
  │   │   │       └── artifacts
  │   │   │           └── running_loss_reset.txt
  │   │   ├── cross_site_val
  │   │   │   ├── cross_val_results.json
  │   │   │   ├── model_shareables
  │   │   │   │   ├── SRV_FL_global_model.pt
  │   │   │   │   ├── site-1
  │   │   │   │   └── site-2
  │   │   │   └── result_shareables
  │   │   │       ├── site-1_SRV_FL_global_model.pt
  │   │   │       ├── site-1_site-1
  │   │   │       ├── site-1_site-2
  │   │   │       ├── site-2_SRV_FL_global_model.pt
  │   │   │       ├── site-2_site-1
  │   │   │       └── site-2_site-2
  │   │   ├── meta.json
  │   │   ├── mlruns
  │   │   │   ├── 0
  │   │   │   │   └── meta.yaml
  │   │   │   └── 470289463842501388
  │   │   │       ├── 39d0b7edb17b437dbf77da2e402b2a4d
  │   │   │       │   ├── artifacts
  │   │   │       │   ├── meta.yaml
  │   │   │       │   ├── metrics
  │   │   │       │   │   ├── running_loss
  │   │   │       │   │   ├── train_loss
  │   │   │       │   │   └── validation_accuracy
  │   │   │       │   ├── params
  │   │   │       │   │   ├── learning_rate
  │   │   │       │   │   ├── loss
  │   │   │       │   │   └── momentum
  │   │   │       │   └── tags
  │   │   │       │       ├── client
  │   │   │       │       ├── job_id
  │   │   │       │       ├── mlflow.note.content
  │   │   │       │       ├── mlflow.runName
  │   │   │       │       └── run_name
  │   │   │       ├── b10ff3e54b0d464c8aab8cf0b751f3cf
  │   │   │       │   ├── artifacts
  │   │   │       │   ├── meta.yaml
  │   │   │       │   ├── metrics
  │   │   │       │   │   ├── running_loss
  │   │   │       │   │   ├── train_loss
  │   │   │       │   │   └── validation_accuracy
  │   │   │       │   ├── params
  │   │   │       │   │   ├── learning_rate
  │   │   │       │   │   ├── loss
  │   │   │       │   │   └── momentum
  │   │   │       │   └── tags
  │   │   │       │       ├── client
  │   │   │       │       ├── job_id
  │   │   │       │       ├── mlflow.note.content
  │   │   │       │       ├── mlflow.runName
  │   │   │       │       └── run_name
  │   │   │       ├── meta.yaml
  │   │   │       └── tags
  │   │   │           └── mlflow.note.content
  │   │   └── tb_events
  │   │       ├── site-1
  │   │       │   ├── events.out.tfevents.1724447288.yuhongw-mlt.86138.3
  │   │       │   ├── metrics_running_loss
  │   │       │   │   └── events.out.tfevents.1724447288.yuhongw-mlt.86138.5
  │   │       │   └── metrics_train_loss
  │   │       │       └── events.out.tfevents.1724447288.yuhongw-mlt.86138.4
  │   │       └── site-2
  │   │           ├── events.out.tfevents.1724447288.yuhongw-mlt.86138.0
  │   │           ├── metrics_running_loss
  │   │           │   └── events.out.tfevents.1724447288.yuhongw-mlt.86138.2
  │   │           └── metrics_train_loss
  │   │               └── events.out.tfevents.1724447288.yuhongw-mlt.86138.1
  │   └── startup
  ├── site-1
  │   ├── local
  │   │   └── log.config
  │   ├── log.txt
  │   ├── simulate_job
  │   │   ├── app_site-1
  │   │   │   └── config
  │   │   │       ├── config_fed_client.conf
  │   │   │       └── config_fed_server.conf
  │   │   ├── meta.json
  │   │   └── models
  │   │       └── local_model.pt
  │   └── startup
  ├── site-2
  │   ├── local
  │   │   └── log.config
  │   ├── log.txt
  │   ├── simulate_job
  │   │   ├── app_site-2
  │   │   │   └── config
  │   │   │       ├── config_fed_client.conf
  │   │   │       └── config_fed_server.conf
  │   │   ├── meta.json
  │   │   └── models
  │   │       └── local_model.pt
  │   └── startup
  └── startup

.. raw:: html

   </details>
   <br />

シミュレータのローカルリソース構成を許可
----------------------------------------------
2.4.0 では、ログのフォーマットを変更するために使用できるのは、シミュレータワークスペースの ``startup`` フォルダ内にある ``log.config`` 設定ファイルのみをサポートしていました。

2.5.0 では、シミュレータワークスペースの下で ``local`` および ``startup`` の全内容を構成できるようにしました。POC の実世界アプリケーションにおける
すべてのローカル設定を ``workspace/local`` フォルダ内に配置し、各サイトへデプロイできます。``log.config`` ファイルも
この ``workspace/local`` フォルダへ移動されました。
