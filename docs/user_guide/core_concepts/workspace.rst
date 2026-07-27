##################################
NVIDIA FLARE ワークスペース
##################################

NVIDIA FLARE は、FL アプリとさまざまなジョブの実行結果を ``job_id`` という名前のフォルダ配下に
保持するためのワークスペースを管理します。

以下は、サーバーとクライアントで NVIDIA FLARE を実行したときのワークスペースのフォルダ構造です。

.. _server_workspace:

************
サーバー
************

.. code-block:: shell

    /some_path_on_fl_server/fl_server_workspace_root/
        admin_audit.log
        log.txt
        startup/
            authorization.json
            fed_server.json
            log_config.json
            readme.txt
            rootCA.pem
            server_context.tenseal
            server.crt
            server.key
            signature.pkl
            start.sh
            stop_fl.sh
            sub_start.sh
        transfer/
        aefdb0a3-6fbb-4c53-a677-b6951d6845a6/
            app_server/
                ...
                config_fed_server.json
            fl_app.txt
            log.txt
        baaf8789-e83f-4863-b085-3ca95303e6bc/
            app_server/
                ...
                config/
                    config_fed_server.json
            fl_app.txt
            log.txt

各 ``job_id`` フォルダの中には ``app_server`` フォルダがあり、その ``job_id`` に対してサーバー上で
実行されている :ref:`application` が格納されています。

各 ``job_id`` フォルダ内の ``log.txt`` ファイルには、そのジョブのログエントリが含まれます。

一方、サーバーフォルダ直下の ``log.txt`` ファイルは、サーバーの制御プロセスのログを記録します。

``startup`` フォルダには、FL サーバープログラムを起動するための設定とスクリプトが含まれます。

.. _access_server_workspace:

サーバー側ワークスペースへのアクセス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

ジョブの実行中は、各ジョブが ``server`` フォルダ配下に対応するワークスペースを持ちます。

ジョブが終了すると、サーバー側のワークスペースは削除されます。
ワークスペースは JobStorage に保存されます。

管理クライアントで ``download_job [JOB_ID]`` を実行することで、サーバー側のワークスペースを
ダウンロードできます。

ダウンロードされたワークスペースは ``[DOWNLOAD_DIR]/[JOB_ID]/workspace/`` に配置されます。

.. note::

    ジョブが終了する前に ``download_job`` を実行すると、ワークスペースフォルダは空になります。


.. _client_workspace:

****************
クライアント
****************

.. code-block:: shell

    /some_path_on_fl_client/fl_client_workspace_root/
        log.txt
        startup/
            client_context.tenseal
            client.crt
            client.key
            fed_client.json
            log_config.json
            readme.txt
            rootCA.pem
            signature.pkl
            start.sh
            stop_fl.sh
            sub_start.sh
        transfer/
        aefdb0a3-6fbb-4c53-a677-b6951d6845a6/
            app_clientA/
                ...
                config_fed_client.json
            fl_app.txt
            log.txt
        baaf8789-e83f-4863-b085-3ca95303e6bc/
            app_clientA/
                ...
                config/
                    config_fed_client.json
            fl_app.txt
            log.txt

各 ``job_id`` フォルダの中には ``app_clientname`` フォルダがあり、その ``job_id`` に対して
クライアント上で実行されている :ref:`application` が格納されています。

各 ``job_id`` フォルダ内の ``log.txt`` ファイルには、そのジョブのログエントリが含まれます。

一方、クライアントフォルダ直下の ``log.txt`` は、クライアントの制御プロセスのログです。

``startup`` フォルダには、FL クライアントプログラムを起動するための設定とスクリプトが含まれます。

:class:`Workspace<nvflare.apis.workspace.Workspace>` オブジェクトは FLContext を通じて利用できます。
Workspace からは、各フォルダの場所に応じてアクセスできます。

.. code-block:: python

    workspace = fl_ctx.get_prop(FLContextKey.WORKSPACE_OBJECT)


.. literalinclude:: ../../../nvflare/apis/workspace.py
    :language: python
    :lines: 36-
