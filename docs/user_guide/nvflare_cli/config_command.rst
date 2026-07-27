.. _config_command:

#########################
Config コマンド
#########################

``nvflare config`` を使用すると、スタートアップキットの登録や有効化を含む、ローカルの CLI 設定を
管理できます。通常のユーザーは、内部的な ``~/.nvflare/config.conf`` のストレージレイアウトを
編集したり、その内容を意識したりする必要はありません。

***********************
コマンドの使い方
***********************

.. code-block:: none

   usage: nvflare config [-h] [--schema] [-d [STARTUP_KIT_DIR]]
                         [-pw [POC_WORKSPACE_DIR]] [-jt [JOB_TEMPLATES_DIR]]
                         {add,use,inspect,list,remove} ...

*****************
よく使う例
*****************

スタートアップキットを登録して有効化します。

.. code-block:: shell

   nvflare config add project_admin /tmp/nvflare/poc/example_project/prod_00/admin@nvidia.com
   nvflare config use project_admin

設定に関する注意事項:

- 保存される設定フォーマットは v2 に正規化され、先頭行が ``version = 2`` になります。
- ``startup_kits.active`` と ``startup_kits.entries`` は ``nvflare config`` によって管理されます。
- ``nvflare config inspect --format json`` と ``nvflare config list --format json`` は、
  自動化のために、ベストエフォートでスタートアップキットの識別情報、証明書の有効期限、および
  ローカルの古いパスの検出結果を出力に含めます。
- ``nvflare config use`` はグローバルな CLI の状態を変更します。サーバーに接続するコマンドを
  実行する自動化処理では、コマンドごとに指定できるオプションのセレクター ``--kit-id`` または
  ``--startup-kit`` を使用することが推奨されます。これらのセレクターは、そのコマンド 1 回に
  限りアクティブなスタートアップキットを上書きし、``startup_kits.active`` を変更しません。
- ``nvflare config -d/--startup_kit_dir`` は 2.7.x のスクリプトとの互換性のために引き続き
  受け付けられますが、非推奨です。新しいワークフローでは ``nvflare config add`` および
  ``nvflare config use`` を使用してください。
- ``nvflare config -pw/--poc_workspace_dir`` は互換性のために引き続き受け付けられますが、
  非推奨です。新しいワークフローでは ``nvflare poc config --pw <poc-workspace-dir>`` を
  使用してください。
- ``nvflare config -jt/--job_templates_dir`` は互換性のために引き続き受け付けられますが、
  ジョブテンプレートの設定は非推奨です。カスタムテンプレートの場所は、それを必要とする
  ジョブコマンドに直接渡すようにしてください。
- ``--poc.workspace`` 、 ``--poc.startup_kit`` 、 ``--prod.startup_kit`` といった開発専用の
  表記は、互換性フラグとしてはサポートされません。
