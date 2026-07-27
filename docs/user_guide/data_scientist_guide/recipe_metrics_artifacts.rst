.. _recipe_metrics_artifacts:

レシピのメトリクスアーティファクト
====================================

組み込みの学習集約レシピは、サーバーワークフローがラウンド単位の集約メトリクスを報告する際に、
標準のメトリクスアーティファクトを書き出します。これらのファイルにより、サーバーログをスクレイピング
することなく、ベンチマーク、レポート、エージェント系ツールからレシピの結果を利用しやすくなります。

アーティファクトはサーバーの実行ディレクトリ配下に書き出されます。

.. code-block:: text

   metrics/
     metrics_summary.json
     round_metrics.jsonl

学習集約メトリクスを報告しないレシピやワークフローでは、これらのファイルを作成する必要はありません。
これには PSI、統計のみのジョブ、単独で実行するクロスサイト検証が含まれます。クロスサイト検証は
引き続き既存の ``cross_site_val/cross_val_results.json`` 出力を使用します。

レシピの動作
---------------

サポートされている組み込みの学習集約レシピでは、ユーザーがこのライターを選択する必要はありません。
レシピのセットアップがサーバー設定の一部としてライターをインストールし、ワークフローが集約メトリクスを
報告した場合にのみファイルを書き出します。

本リリースでは、メトリクスアーティファクトを無効化するレシピ引数は提供されていません。これらの
アーティファクトを書き出したくないカスタムジョブでは、サーバー設定からメトリクスアーティファクト
ライターを除外してください。

レコーダーのセマンティクス
----------------------------

メトリクスアーティファクトライターはレコーダーです。ワークフロー、アグリゲータ、モデルセレクタが
すでに生成しているメトリクスとメタデータを永続化します。

* ラウンド集約結果から得られる公式の集約メトリクス
* 各ラウンドでクライアントから受け取ったサイトごとのメトリクス
* モデル選択ロジックが公開する公式のベストメトリクスメタデータ
* 利用可能な場合は、集約の来歴 (provenance)、重み、スキップされた値

このライターはメトリクスの再計算、ベストラウンドの選択、max/min ポリシーの推定、ログの解析、
プールされた予測値からの AUROC のような非線形メトリクスの計算は行いません。

メトリクス名は動的です。 ``auroc`` 、 ``accuracy`` 、 ``loss`` 、 ``dice`` 、 ``rmse`` 、 ``f1``
といった名前は、クライアントまたはワークフローのメトリクスキーであり、ハードコードされたスキーマ
フィールドではありません。

ラウンド番号は ``AppConstants.CURRENT_ROUND`` や ``FLModel.current_round`` などのワークフロー
メタデータから提供されたとおりに記録されます。既定では 0 始まりであり、ライターによって振り直される
ことはありません。

``metrics_summary.json``
------------------------

``metrics_summary.json`` には、最後に完了したメトリクスラウンドの最終集約メトリクスと、利用可能な
場合はモデルセレクタからの公式のベストメトリクスメタデータが含まれます。

例:

.. code-block:: json

   {
     "schema_version": "1",
     "status": "metrics_reported",
     "job_name": "ames_fedavg",
     "metric_source": "client_reported_flmodel_metrics",
     "key_metric": {
       "name": "auroc",
       "mode": "max",
       "mode_source": "IntimeModelSelector.negate_key_metric"
     },
     "final_round": 2,
     "final_aggregated_metrics": [
       {
         "name": "auroc",
         "value": 0.7421
       },
       {
         "name": "train_loss",
         "value": 0.492
       }
     ],
     "best_round": 0,
     "best_metrics": [
       {
         "name": "auroc",
         "value": 0.7500010132169
       }
     ],
     "best_metric_source": "IntimeModelSelector",
     "best_metric_detail_source": "initial_metrics",
     "aggregation": {
       "method": "weighted_average",
       "weight_key": "NUM_STEPS_CURRENT_ROUND",
       "metric_policy": "finite_numeric_metrics_only_per_key_denominator"
     },
     "round_metrics_file": "round_metrics.jsonl",
     "notes": [
       "Aggregated metrics are weighted averages of client-reported metric values.",
       "Nonlinear metrics are not recomputed from pooled predictions."
     ]
   }

ベストメトリクス関連のフィールドは任意です。セレクタまたはワークフローが明示的なベスト選択メタデータを
公開している場合にのみ存在します。ライターがメトリクス値からベストラウンドを推定することはありません。

``round_metrics.jsonl``
-----------------------

``round_metrics.jsonl`` には、完了したメトリクスラウンドごとに 1 つの JSON オブジェクトが含まれます。
各行には、公式の集約メトリクス、サイトごとのクライアントメトリクス、任意の集約メタデータ、および
スキップされたメトリクス値が記録されます。

行の例:

.. code-block:: json

   {
     "round": 0,
     "aggregated_metrics": [
       {
         "name": "auroc",
         "value": 0.7500010132169
       },
       {
         "name": "train_loss",
         "value": 0.4855
       }
     ],
     "sites": [
       {
         "name": "site-1",
         "metrics": [
           {
             "name": "train_loss",
             "value": 0.4707
           },
           {
             "name": "auroc",
             "value": 0.7380791446479046
           }
         ],
         "weight": 2911,
         "weight_key": "NUM_STEPS_CURRENT_ROUND"
       },
       {
         "name": "site-2",
         "metrics": [
           {
             "name": "train_loss",
             "value": 0.5003
           },
           {
             "name": "auroc",
             "value": 0.7619228817858955
           }
         ],
         "weight": 2911,
         "weight_key": "NUM_STEPS_CURRENT_ROUND"
       }
     ],
     "aggregation": {
       "method": "weighted_average",
       "weight_key": "NUM_STEPS_CURRENT_ROUND",
       "metric_policy": "finite_numeric_metrics_only_per_key_denominator"
     },
     "skipped_metrics": [
       {
         "site": "site-1",
         "name": "debug_blob",
         "reason": "unsupported_type"
       }
     ]
   }

動的なメトリクス名は、JSON オブジェクトのキーとしてではなく、配列内の ``name`` の値として格納されます。
これにより、下流のツールでクライアント由来の名前がオブジェクト構造として扱われることを防ぎます。

安全なメトリクス値
--------------------

クライアントは信頼できないメトリクス生成元です。ライターは正規化された JSON セーフなスカラー値のみを
シリアライズし、サーバーの実行ディレクトリ配下の固定ファイル名に書き出します。

公式の集約メトリクスは、有限の数値と真偽値を受け付けます。サイトごとのメトリクスは、有限の数値、
真偽値、および長さが制限された文字列値を受け付けます。サポートされないオブジェクト、テンソル、配列、
入れ子のコンテナ、サイズ超過の値、 ``NaN`` 、 ``Infinity`` はスキップされ、長さが制限された理由の
レコードとともに ``skipped_metrics`` に報告されます。

ダウンロードされたアーティファクト
------------------------------------

メトリクスファイルは、存在する場合は通常のダウンロード対象のジョブ結果に含まれます。自動化では、
ワークスペースのレイアウトからパスを組み立てるのではなく、ジョブダウンロードの JSON 出力を使って
ダウンロード先のローカルパスを見つけてください。

.. code-block:: shell

   nvflare job download <job_id> -o ./downloads --format json

レスポンスの抜粋例:

.. code-block:: json

   {
     "schema_version": "1",
     "status": "ok",
     "data": {
       "download_path": "/abs/path/downloads/abc123",
       "artifact_discovery": "completed",
       "artifacts": {
         "metrics_summary": "/abs/path/downloads/abc123/workspace/metrics/metrics_summary.json",
         "round_metrics": "/abs/path/downloads/abc123/workspace/metrics/round_metrics.jsonl"
       },
       "missing_artifacts": []
     }
   }

``metrics_summary`` と ``round_metrics`` は、それらのファイルがダウンロード結果に存在する場合にのみ
報告されます。古いジョブや集約メトリクスを持たないジョブはラウンドごとのメトリクスファイルを作成しない
ため、 ``round_metrics`` は任意です。
