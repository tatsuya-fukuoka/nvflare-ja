.. _recipe_command:

#########################
Recipe コマンド
#########################

``nvflare recipe`` は、組み込みの Job Recipe API レシピを一覧表示し、エージェントやスクリプトが
レシピを選択する際に利用できる構造化されたメタデータを表示します。

***********************
コマンドの使い方
***********************

.. code-block:: none

   nvflare recipe -h

   usage: nvflare recipe [-h]  ...

   recipe subcommands:
     list      list available recipes (default)
     show      show structured metadata for a recipe

********************
レシピの一覧表示
********************

利用可能な組み込みレシピを表示するには ``nvflare recipe list`` を使用します。

.. code-block:: shell

   nvflare recipe list

テキストモードでは、インストール済みのレシピメタデータをインポートして検査する前に、
コマンドが ``Loading installed recipe catalog...`` と出力します。これはローカルの Python 環境上の
操作であり、FLARE サーバーには接続しません。

フレームワークで絞り込む:

.. code-block:: shell

   nvflare recipe list --framework pytorch

``--framework`` は ``--filter framework=<framework>`` の短縮形です。

サポートされているフレームワークのフィルタ値:

- ``core``
- ``numpy``
- ``pytorch``
- ``tensorflow``
- ``sklearn``
- ``xgboost``

レシピのメタデータで絞り込む:

.. code-block:: shell

   nvflare recipe list --filter framework=pytorch --filter algorithm=fedavg
   nvflare recipe list --filter privacy=homomorphic_encryption

サポートされている ``--filter`` のキー:

- ``framework``
- ``privacy``
- ``algorithm``
- ``aggregation``
- ``state_exchange``

``--filter`` は繰り返し指定できます。異なるキーに対するフィルタは組み合わせて適用され、
同じキーを繰り返した場合は指定されたいずれかの値に一致します。フィルタ値ではハイフンと
アンダースコアが正規化されるため、``homomorphic-encryption`` と ``homomorphic_encryption`` は
同等に扱われます。

メタデータのフィルタ値
======================

フィルタ値はレシピのメタデータの値です。ある値が有効となるのは、インストール済みのカタログ内の
少なくとも 1 つのレシピがその値を宣言している場合のみです。実際には、ほとんどのメタデータ値は
特定のフレームワークに対応しているため、正確な結果を得たい場合はメタデータのフィルタを
``framework`` と組み合わせてください。

アルゴリズムの値:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 値
     - フレームワーク
     - 例
   * - ``cyclic``
     - ``core``, ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter algorithm=cyclic``
   * - ``fedavg``
     - ``core``, ``numpy``, ``pytorch``, ``sklearn``, ``tensorflow``
     - ``nvflare recipe list --filter algorithm=fedavg``
   * - ``fedavg_logistic_regression``
     - ``numpy``
     - ``nvflare recipe list --filter algorithm=fedavg_logistic_regression``
   * - ``fedeval``
     - ``pytorch``
     - ``nvflare recipe list --filter algorithm=fedeval``
   * - ``fedopt``
     - ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter algorithm=fedopt``
   * - ``fedprox``
     - ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter algorithm=fedprox``
   * - ``fedstats``
     - ``core``
     - ``nvflare recipe list --filter algorithm=fedstats``
   * - ``kmeans``
     - ``sklearn``
     - ``nvflare recipe list --filter algorithm=kmeans``
   * - ``scaffold``
     - ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter algorithm=scaffold``
   * - ``svm``
     - ``sklearn``
     - ``nvflare recipe list --filter algorithm=svm``
   * - ``swarm``
     - ``pytorch``
     - ``nvflare recipe list --filter algorithm=swarm``
   * - ``xgboost_bagging``
     - ``xgboost``
     - ``nvflare recipe list --filter algorithm=xgboost_bagging``
   * - ``xgboost_horizontal``
     - ``xgboost``
     - ``nvflare recipe list --filter algorithm=xgboost_horizontal``
   * - ``xgboost_vertical``
     - ``xgboost``
     - ``nvflare recipe list --filter algorithm=xgboost_vertical``

集約 (aggregation) の値:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 値
     - フレームワーク
     - 例
   * - ``cluster_centers``
     - ``sklearn``
     - ``nvflare recipe list --filter framework=sklearn --filter aggregation=cluster_centers``
   * - ``server_optimizer``
     - ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter aggregation=server_optimizer``
   * - ``support_vectors``
     - ``sklearn``
     - ``nvflare recipe list --filter framework=sklearn --filter aggregation=support_vectors``
   * - ``tree_ensemble``
     - ``xgboost``
     - ``nvflare recipe list --filter framework=xgboost --filter aggregation=tree_ensemble``
   * - ``weighted_average``
     - ``core``, ``numpy``, ``pytorch``, ``sklearn``, ``tensorflow``
     - ``nvflare recipe list --filter aggregation=weighted_average``

状態交換 (state exchange) の値:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 値
     - フレームワーク
     - 例
   * - ``cluster_centers``
     - ``sklearn``
     - ``nvflare recipe list --filter state_exchange=cluster_centers``
   * - ``full_model``
     - ``core``, ``numpy``, ``pytorch``, ``sklearn``, ``tensorflow``
     - ``nvflare recipe list --filter state_exchange=full_model``
   * - ``model_weights``
     - ``numpy``
     - ``nvflare recipe list --filter state_exchange=model_weights``
   * - ``support_vectors``
     - ``sklearn``
     - ``nvflare recipe list --filter state_exchange=support_vectors``
   * - ``trees``
     - ``xgboost``
     - ``nvflare recipe list --filter state_exchange=trees``
   * - ``weight_diff``
     - ``pytorch``, ``tensorflow``
     - ``nvflare recipe list --filter state_exchange=weight_diff``

プライバシー (privacy) の値:

.. list-table::
   :header-rows: 1
   :widths: 30 30 40

   * - 値
     - フレームワーク
     - 例
   * - ``homomorphic_encryption``
     - ``pytorch``
     - ``nvflare recipe list --filter privacy=homomorphic_encryption``

``privacy`` フィルタは、レシピのエントリが宣言しているプライバシー機能に一致します。
レシピがプライバシーの値を宣言していないからといって、そのアルゴリズム自体が
プライバシー強化技術 (PET) と互換性がないという意味ではありません。たとえば FedAvg は、
追加の構成やコンポーネントによって複数の PET と組み合わせることができますが、
汎用の FedAvg レシピはデフォルトではいずれも有効にしません。

その他の例:

.. code-block:: shell

   nvflare recipe list --filter framework=pytorch --filter algorithm=fedopt
   nvflare recipe list --filter framework=tensorflow --filter aggregation=server_optimizer
   nvflare recipe list --filter framework=xgboost --filter state_exchange=trees
   nvflare recipe list --filter framework=sklearn --filter aggregation=cluster_centers
   nvflare recipe list --filter privacy=homomorphic_encryption

その他のオプション:

- ``--framework`` を省略すると、利用可能なすべてのレシピが返されます。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

動作に関する注意:

- このコマンドは、オプションのフレームワーク依存関係がローカルにインストールされていない
  レシピバリアントも含め、ドキュメント化されているすべての組み込みレシピバリアントを一覧表示します。
- ``optional_dependencies`` は、現在インストールされていないフレームワークのレシピを実行するために
  必要なパッケージを報告します。
- 有効なメタデータフィルタでも、該当するレシピがない場合は空のリストが返されます。
- このコマンドは、ドキュメント化されたレシピマニフェストと、動的なレシピモジュール検出を
  組み合わせて使用します。オプションの依存関係が不足しているためにレシピモジュールを
  インポートできない場合でも、CLI はドキュメント化されたメタデータを返し、可能な範囲で
  ソースから静的にコンストラクタのパラメータを導出します。

CLI はテキストモードでは人間が読みやすいテーブルを出力し、併せて機械可読な結果エンベロープも
出力します。

``nvflare recipe list --format json`` が出力する JSON は、スクリプトやツールが依拠できる安定した
ドキュメント化済みの構造を持っています。この構造は
:download:`レシピカタログ JSON スキーマ <../../schemas/recipe_catalog.schema.json>` に記述されています。

JSON レスポンスの例:

.. code-block:: json

   {
     "schema_version": "1",
     "status": "ok",
     "exit_code": 0,
     "data": [
       {
         "name": "fedavg-pt",
         "description": "FedAvg for PyTorch nn.Module models",
         "framework": "pytorch",
         "module": "nvflare.app_opt.pt.recipes.fedavg",
         "class": "FedAvgRecipe",
         "algorithm": "fedavg",
         "aggregation": "weighted_average",
         "state_exchange": "full_model",
         "privacy": []
       }
     ]
   }

********************
レシピの詳細表示
********************

1 つのレシピについてクエリ可能なメタデータを取得するには、``nvflare recipe list`` が返した名前を
指定して ``nvflare recipe show`` を使用します。

.. code-block:: shell

   nvflare recipe show fedavg-pt --format json

テキストモードでは、選択したレシピのメタデータをインポートして検査している間、コマンドが
``Loading installed recipe metadata for '<name>'...`` と出力します。人間向けの出力は主要なフィールドを
要約し、コンストラクタのパラメータの詳細をすべて確認するための正確な JSON コマンドを示します。

JSON レスポンスには、一覧表示時のメタデータに加えて、フレームワークのサポート状況、プライバシーの
互換性、クライアント要件、コンストラクタのパラメータ、オプションの依存関係、テンプレート参照が
含まれます。パラメータのメタデータは、レシピのコンストラクタのシグネチャまたは静的なソース解析から
導出されます。コマンドがレシピをインスタンス化することはありません。

パラメータの転送タイプを構成できるレシピについては、テキスト出力にデフォルトの状態交換と転送設定が
報告されます。たとえば FedAvg では、デフォルトの転送がフルモデルであり、かつ差分を送信するように
構成することもできるため、``state_exchange: full_model (default; params_transfer_type=FULL, supports FULL
or DIFF)`` と報告されます。

JSON レスポンスの例:

.. code-block:: json

   {
     "schema_version": "1",
     "status": "ok",
     "exit_code": 0,
     "data": {
       "name": "fedavg-pt",
       "description": "FedAvg for PyTorch nn.Module models",
       "framework": "pytorch",
       "module": "nvflare.app_opt.pt.recipes.fedavg",
       "class": "FedAvgRecipe",
       "algorithm": "fedavg",
       "aggregation": "weighted_average",
       "state_exchange": "full_model",
       "privacy": [],
       "client_requirements": {
         "state_exchange": "full_model",
         "requires_training_script": true,
         "requires_per_site_config": true,
         "requires_site_list": false,
         "min_clients": {"required": true, "default": null}
       },
       "framework_support": ["pytorch"],
       "heterogeneity_support": ["horizontal"],
       "privacy_compatible": [],
       "parameters": [
         {
           "name": "min_clients",
           "type": "int",
           "required": true,
           "default": null,
           "kind": "keyword_only"
         }
       ],
       "optional_dependencies": ["pip install nvflare[PT]", "pip install torch"],
       "template_references": []
     }
   }

*********************
典型的なワークフロー
*********************

``nvflare recipe list`` は探索用のツールです。一般的なワークフローは次のとおりです。

.. code-block:: shell

   nvflare recipe list --filter framework=pytorch --filter algorithm=fedavg
   nvflare recipe show fedavg-pt --format json
   python job.py --export --export-dir /tmp/nvflare/hello-pt
   nvflare job submit -j /tmp/nvflare/hello-pt

新しいサンプルやレシピでは、``nvflare recipe list`` が非推奨の ``nvflare job list_templates`` による
探索フローを置き換えます。

*************************
JSON 出力とヘルプ
*************************

機械可読なコマンド探索には ``--schema`` を使用します。

.. code-block:: shell

   nvflare recipe list --schema
   nvflare recipe show --schema

``--schema`` はコマンドの引数と動作を記述します。``nvflare recipe list --format json`` のカタログ出力を
検証する際には、レシピカタログ JSON スキーマを使用してください。

トップレベルの CLI も JSON 出力モードをサポートしています。

.. code-block:: shell

   nvflare recipe list --format json
   nvflare recipe list --framework sklearn --format json
   nvflare recipe list --filter framework=pytorch --filter state_exchange=full_model --format json
   nvflare recipe show fedavg-pt --format json

人間向けの引数エラーでは、まずヘルプが出力され、続いて具体的なエラーが表示されます。
JSON モードでは JSON エンベロープのみが出力されます。
