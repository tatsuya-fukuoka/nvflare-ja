.. _nvflare_cli:

###########################
NVFlare CLI
###########################

``nvflare`` は、ローカル開発、プロビジョニング、分散プロビジョニング、
ジョブの提出、およびランタイム操作のためのメインのコマンドラインエントリポイントです。

``nvflare -h`` を実行すると、現在のコマンド一覧が表示されます:

.. code-block:: none

   usage: nvflare [-h] [--version] [--format {txt,json}]
                  [--connect-timeout CONNECT_TIMEOUT] ...

グローバルオプション
==========================================

- ``--version`` / ``-V``: NVFlare のバージョンを表示します
- ``--format {txt,json}``: 人間が読める出力か JSON エンベロープかを選択します
- ``--connect-timeout``: リモートコマンドのサーバー接続タイムアウトを制御します

人間が読める形式の引数エラーは、まずコマンドのヘルプを表示し、その後に具体的な
メッセージとヒントを表示します。``--format json`` は JSON レスポンスまたは JSON
エラーエンベロープのみを表示します。これは自動化やエージェント形式の呼び出し元を
想定したものです。

コマンドグループ
================================

- ``poc``: ローカルの概念実証(POC)デプロイメントを管理します
- ``provision``: 集中型プロビジョニングワークフロー
- ``cert`` / ``package``: 分散型(手動)プロビジョニングワークフロー
- ``deploy``: 既存のサーバーまたはクライアントのスタートアップキットを、
  Docker や Kubernetes などのデプロイメントランタイム向けに準備します
- ``job``: ジョブの提出と管理を行います
- ``study``: マルチスタディのライフサイクルを管理します(登録、サイトの登録、ユーザーの管理)
- ``system``: 稼働中の FL システムを検査・操作します
- ``config``: スタートアップキットの登録やアクティブキットの選択など、
  ローカルの CLI 設定を管理します
- ``recipe``: エクスポートされたジョブ向けの組み込みレシピファミリーを一覧表示します
- ``preflight_check`` / ``preflight``: デプロイ前にプロビジョニング済み
  スタートアップキットを検証します(``preflight`` が推奨エイリアスです)
- ``dashboard``: ダッシュボードサービスを起動します

``simulator`` や ``authz_preview`` のように、ヘルプに引き続き表示される非推奨コマンドは
簡潔にのみ文書化されており、古いセットアップを維持している場合を除き、新しい
ワークフローでは使用しないでください。

.. toctree::
   :maxdepth: 1

   fl_simulator
   poc_command
   config_command
   provision_command
   distributed_provisioning
   deploy_command
   job_cli
   study_command
   system_command
   cert_command
   package_command
   recipe_command
   preflight_check
   dashboard_command
