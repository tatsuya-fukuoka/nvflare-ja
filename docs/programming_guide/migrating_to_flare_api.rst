:orphan:

.. _migrating_to_flare_api:

.. deprecated:: 2.7
   この移行ガイドはすでに積極的にメンテナンスされていません。新しいプロジェクトでは :ref:`Client API <client_api>` を使用してください。

FLAdminAPI から FLARE API への移行
====================================================

:mod:`FLARE API<nvflare.fuel.flare_api.flare_api>` は、バージョン 2.3 でより良いユーザー体験を実現するために再設計された :ref:`fladmin_api` です。
FLAdminAPI と同様に、FLARE API は FL サーバーに発行できる管理コマンドのラッパーであり、プロビジョニングされた管理
クライアントの証明書と鍵を使って :class:`Session<nvflare.fuel.flare_api.flare_api.Session>` を初期化し、API のコマンドを利用できます。

ここで説明しているレガシーの FLAdminAPI モジュールは NVFlare から削除されています。このページは、古いコードを
FLARE API へ移行するための歴史的な対応表として残されています。

.. _migrating_to_flare_api_initialization:

API 初期化の移行
----------------------------
FLAdminAPI の初期化は、証明書へのパスを含む多数の引数が必要だったため煩雑でした。そのため、管理ユーザーの
ユーザー名と管理スタートアップキットのディレクトリへのパスを指定して FLAdminAPI を初期化するために
``FLAdminAPIRunner``
が使われていました。

FLAdminAPI の初期化:

.. code-block:: python

    api_instance = FLAdminAPI(
        ca_cert="/workspace/example_project/prod_00/super@nvidia.com/startup/rootCA.pem",
        client_cert="/workspace/example_project/prod_00/super@nvidia.com/startup/client.crt",
        client_key="/workspace/example_project/prod_00/super@nvidia.com/startup/client.key",
        upload_dir="/workspace/example_project/prod_00/super@nvidia.com/transfer",
        download_dir="/workspace/example_project/prod_00/super@nvidia.com/transfer",
        user_name="super@nvidia.com"
    )

FLAdminAPIRunner の初期化。指定された admin_dir 内のスタートアップキットの fed_admin.json の値を使って FLAdminAPI を初期化します:

.. code-block:: python

    runner = FLAdminAPIRunner(
        username="super@nvidia.com",
        admin_dir="/workspace/example_project/prod_00/super@nvidia.com"
    )

:ref:`flare_api_initialization` は ``FLAdminAPIRunner`` と似ており、
:func:`new_secure_session<nvflare.fuel.flare_api.flare_api.new_secure_session>` は、ユーザー名と、管理クライアントの
証明書および鍵を含む startup フォルダを持つルート管理ディレクトリへのパスという 2 つの必須引数を
取ります:

.. code-block:: python

    from nvflare.fuel.flare_api.flare_api import new_secure_session

    sess = new_secure_session(
        username="super@nvidia.com",
        startup_kit_location="/workspace/example_project/prod_00/super@nvidia.com"
    )


ログインは自動的に処理され、返されたセッションオブジェクト(前のコードブロックの ``sess``)でコマンドを実行できます。
これは、API オブジェクト自体を通じてコマンドを発行していた :ref:`fladmin_api` や、``FLAdminAPIRunner`` の場合の
``self.api`` とは対照的です(以下のコードブロックでは FLAdminAPI に対して ``runner.api`` を使用しています)。


FLARE API への移行に関する一般的な注意事項
---------------------------------------------------

戻り値の構造
^^^^^^^^^^^^^^^^
FLAdminAPI のコマンドの戻り値の構造は、ステータス、詳細、サーバーからの生のレスポンスを含む ``FLAdminAPIResponse`` オブジェクトでした。
そのため、ステータスやその他の情報を利用・出力するにはレスポンスをパースする必要がありました。FLARE API では、ステータスと
詳細のディクショナリを持つオブジェクトを返すことはなくなり、レスポンスはコマンドに応じて決まり、大幅に簡略化されています。各コマンドが何を返すかの詳細は以下、
または :mod:`FLARE API<nvflare.fuel.flare_api.flare_api>` の docstring を参照してください。

FLARE API は例外を送出するようになりました
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FLAdminAPI のようにパースが必要なエラー付きのステータスを返すのではなく、FLARE API はエラーや予期しないことが
発生した場合に例外を送出するようになりました。これらの例外の処理は FLARE API を使用するコード側の責任となります。つまり一般に、
FLAdminAPI のレスポンスをパースしていた ``api_command_wrapper()`` のようなものはもはや不要です。

セッションのクローズ
^^^^^^^^^^^^^^^^^^^^^^^^
FLARE API では、セッションを終了するために ``close()`` を使用します。``finally`` ブロックに ``close()`` を置き、try ブロックの中でセッションを使ってコマンドを実行するのが理想的です。
詳細は :ref:`flare_api_implementation_notes` を参照してください。


.. _migrating_fladminapi_commands_to_flare_api:

FLAdminAPI のコマンドを FLARE API へ移行する
--------------------------------------------------------
このセクションではまずコマンドの概要を示し、続いて各コマンドについて、これまでの FLAdminAPI での使い方と出力、
および FLARE API での新しい方法の例を説明します。

.. csv-table::
    :header: FLAdminAPI,FLARE API,追加バージョン,備考
    :widths: 15, 15, 30, 30

    check_status,get_system_info,2.3.0,出力を簡略化し再フォーマット(詳細は後述)
    submit_job,submit_job,2.3.0,出力を簡略化(詳細は後述)
    list_job,list_job,2.3.0,出力を簡略化(詳細は後述)
    wait_until_server_status,monitor_job,2.3.0,引数名と機能を変更(詳細は後述)
    download_job,download_job_result,2.3.0,出力を簡略化(詳細は後述)
    clone_job,clone_job,2.3.0,出力を簡略化(詳細は後述)
    abort_job,abort_job,2.3.0,出力を簡略化(詳細は後述)
    delete_job,delete_job,2.3.0,出力を簡略化(詳細は後述)
    check_status,get_client_job_status,2.4.0,クライアント専用
    restart,restart,2.4.0,
    shutdown,shutdown,2.4.0,
    set_timeout,set_timeout,2.4.0,セッションベースに変更
    get_available_apps_to_upload,get_available_apps_to_upload,2.4.0,
    shutdown_system,shutdown_system,2.4.0,
    ls_target,ls_target,2.4.0,
    cat_target,cat_target,2.4.0,
    ,tail_target,2.4.0,一貫性のために追加
    tail_target_log,tail_target_log,2.4.0,
    ,head_target,2.4.0,新規
    ,head_target_log,2.4.0,新規
    grep_target,grep_target,2.4.0,
    get_working_directory,get_working_directory,2.4.0,
    show_stats,show_stats,2.4.0,戻り値の構造を変更
    show_errors,show_errors,2.4.0,戻り値の構造を変更
    reset_errors,reset_errors,2.4.0,
    get_connected_client_list,get_connected_client_list,2.4.0,
    abort,,2.4.0,廃止
    remove_client,remove_client,2.4.0,アクティブなクライアントトークンを解放するのみです。再接続を防ぐには disable_client を使用してください

Check Status から Get System Info へ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
これまで FLAdminAPI でシステム情報を取得するには、主に ``check_status()`` コマンドを使用していました:

.. code-block:: python

    from nvflare.fuel.hci.client.fl_admin_api_spec import TargetType

    api_command_wrapper(runner.api.check_status(TargetType.SERVER))

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {<FLDetailKey.SERVER_ENGINE_STATUS: 'server_engine_status'>: 'stopped',
    <FLDetailKey.STATUS_TABLE: 'status_table'>: [['CLIENT',
        'TOKEN',
        'LAST CONNECT TIME'],
    ['site_a',
        '32ebdf1c-b51b-4eb3-ae49-4ac488a2aaa1',
        'Thu Jan 26 15:13:12 2023'],
    ['site_b',
        '4bbfb243-9ae1-4339-9e6d-750092ebc240',
        'Thu Jan 26 15:13:12 2023']],
    <FLDetailKey.REGISTERED_CLIENTS: 'registered_clients'>: 2},
    'raw': {'time': '2023-01-26 15:13:25.652993',
    'data': [{'type': 'string', 'data': 'Engine status: stopped'},
    {'type': 'table', 'rows': [['JOB_ID', 'APP NAME']]},
    {'type': 'string', 'data': 'Registered clients: 2 '},
    {'type': 'table',
        'rows': [['CLIENT', 'TOKEN', 'LAST CONNECT TIME'],
        ['site_a',
        '32ebdf1c-b51b-4eb3-ae49-4ac488a2aaa1',
        'Thu Jan 26 15:13:12 2023'],
        ['site_b',
        '4bbfb243-9ae1-4339-9e6d-750092ebc240',
        'Thu Jan 26 15:13:12 2023']]}],
    'meta': {'status': 'ok',
    'info': '',
    'server_status': 'stopped',
    'server_start_time': 1674763921.3592467,
    'jobs': [],
    'clients': [{'client_name': 'site_a',
        'client_last_conn_time': 1674763992.4529057},
        {'client_name': 'site_b', 'client_last_conn_time': 1674763992.4763987}]},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}


FLARE API では、新しいコマンド ``get_system_info()`` が、server_info(サーバーのステータスと起動時刻)、
client_info(接続中の各クライアントとそのクライアントの最終接続時刻)、job_info(job_id と app_name を含む現在のジョブ一覧)
から成る SystemInfo オブジェクトを返します。

.. code-block:: python

    sess.get_system_info()

:class:`SystemInfo<nvflare.fuel.flare_api.api_spec.SystemInfo>` オブジェクトに対して print を呼び出すと以下のような結果が得られます。
また、server_info、client_info、job_info の各変数にアクセスして内部のデータを取得することもできます。

.. code-block:: bash

    SystemInfo
    server_info: status: stopped, start_time: Thu Jan 26 15:12:01 2023
    client_info:
    site_a(last_connect_time: Thu Jan 26 15:12:42 2023)
    site_b(last_connect_time: Thu Jan 26 15:12:42 2023)
    job_info:
    job_id: 44d32a5f-9766-44b6-aef5-7ed9fd168335
    app_name: hello-numpy


ジョブの投入 (Submit Job)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FLAdminAPI と FLARE API の ``submit_job()`` コマンドは非常によく似ています。必須の引数は両者で同じで、
投入するジョブへのパスを文字列で指定します。FLAdminAPI での ``submit_job()``:

.. code-block:: python

    path_to_example_job = "/workspace/NVFlare/examples/hello-world/hello-numpy"
    runner.api.submit_job(path_to_example_job)

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {'message': 'Submitted job: 5d0eaa30-6936-4044-918e-cd9c3f5edf9b',
    'job_id': '5d0eaa30-6936-4044-918e-cd9c3f5edf9b'},
    'raw': {'time': '2023-01-26 15:30:35.260527',
    'data': [{'type': 'string',
        'data': 'Submitted job: 5d0eaa30-6936-4044-918e-cd9c3f5edf9b'},
    {'type': 'success', 'data': ''}],
    'meta': {'status': 'ok',
    'info': '',
    'job_id': '5d0eaa30-6936-4044-918e-cd9c3f5edf9b'},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}


FLARE API では、``submit_job()`` はジョブの投入に成功した場合にそのジョブの job_id を返すため、その値を
保存して後で使用できます。

.. code-block:: python

    path_to_example_job = "/workspace/NVFlare/examples/hello-world/hello-numpy"
    job_id = sess.submit_job(path_to_example_job)
    print(job_id + " was submitted")

.. code-block:: bash

    5d0eaa30-6936-4044-918e-cd9c3f5edf9b was submitted


ジョブの一覧表示 (List Jobs)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FLAdminAPI の ``list_jobs()`` コマンドはオプションの引数としてオプション指定の文字列を取っていましたが、FLARE API では
オプションはブール値として設定します。FLAdminAPI での ``list_jobs()``:

.. code-block:: python

    runner.api.list_jobs()
    # runner.api.list_jobs("-a -d")

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': [['JOB ID', 'NAME', 'STATUS', 'SUBMIT TIME', 'RUN DURATION'],
    ['5d0eaa30-6936-4044-918e-cd9c3f5edf9b',
    'hello-numpy',
    'FINISHED:COMPLETED',
    '2023-01-26T15:30:35.262048-05:00',
    '0:00:48.170128']],
    'raw': {'time': '2023-01-26 15:47:23.091621',
    'data': [{'type': 'table',
        'rows': [['JOB ID', 'NAME', 'STATUS', 'SUBMIT TIME', 'RUN DURATION'],
        ['5d0eaa30-6936-4044-918e-cd9c3f5edf9b',
        'hello-numpy',
        'FINISHED:COMPLETED',
        '2023-01-26T15:30:35.262048-05:00',
        '0:00:48.170128']]},
    {'type': 'success', 'data': ''}],
    'meta': {'jobs': [{'job_id': '5d0eaa30-6936-4044-918e-cd9c3f5edf9b',
        'job_name': 'hello-numpy',
        'status': 'FINISHED:COMPLETED',
        'submit_time': '2023-01-26T15:30:35.262048-05:00',
        'duration': '0:00:48.170128'}],
    'status': 'ok',
    'info': ''},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}


FLARE API での ``list_job()``:

.. code-block:: python

    list_jobs_output = sess.list_jobs()
    print(list_jobs_output)
    # list_jobs_output_detailed_all = sess.list_jobs(detailed=True, all=True)
    # print(list_jobs_output_detailed_all)

.. code-block:: bash

    [{'job_id': '9382ff9e-eb7e-4e0d-9a8e-78c82747b5ac', 'job_name': 'hello-numpy', 'status': 'RUNNING', 'submit_time': '2023-01-26T15:56:30.188836-05:00', 'duration': '0:00:32.686275'}]


Wait Until から Monitor Job へ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FLAdminAPI には、トレーニングのステータスを監視するために使用できる ``wait_until_server_status()`` と
``wait_until_client_status()`` がありました:

.. code-block:: python

    runner.api.wait_until_server_status()

デフォルトでは、FLAdminAPI の ``wait_until`` 系の関数は、サーバーエンジンのステータスが stopped になるか、クライアントにアクティブな
ジョブがなくなるまで待機してから "SUCCESS" のステータスを返していました。

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>}

FLARE API では、``monitor_job()`` が同様の機能を提供しますが、必須の引数として job_id を取り、そのジョブが完了するまで
ジョブのメタ情報を継続的に取得します。

.. code-block:: python

    sess.monitor_job(job_id)

.. code-block:: bash

    <MonitorReturnCode.JOB_FINISHED: 0>

追加のオプション引数は少し変更されており、``interval`` は ``poll_interval`` になって型が int から float に変わり、
``timeout`` は名前は同じですが型が int から float に変わり、``callback`` は ``cb`` になりました。

FLARE API の ``monitor_job()`` コマンドはコールバックによってカスタマイズできるように設計されています。詳細は :ref:`flare_api_monitor_job` を参照してください。


Download Job から Download Job Result へ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
FLAdminAPI の ``download_job()`` コマンドは ``download_job_result()`` に名前が変更されました。必須の引数として job_id を文字列で取る点は
FLARE API でも同じです。コマンドの動作も同じままですが、出力はダウンロードされたジョブへのパスのみに
簡略化されています。FLAdminAPI の場合:

.. code-block:: python

    runner.api.download_job(job_id)

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {'message': 'Download to dir /workspace/workspace/hello-example/prod_00/admin@nvidia.com/transfer'},
    'raw': {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': 'Download to dir /workspace/workspace/hello-example/prod_00/admin@nvidia.com/transfer',
    'meta': {'status': 'ok',
    'info': '',
    'job_id': '5d0eaa30-6936-4044-918e-cd9c3f5edf9b'}}}

FLARE API での ``download_job_result()``:

.. code-block:: python

    sess.download_job_result(job_id)

.. code-block:: bash

    '/workspace/workspace/hello-example/prod_00/admin@nvidia.com/transfer/5d0eaa30-6936-4044-918e-cd9c3f5edf9b'


ジョブのクローン (Clone Job)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``clone_job()`` コマンドの使い方は FLAdminAPI と FLARE API で同じで、必須の引数は job_id の文字列のみです。
コマンドの動作も同じままですが、出力は新しくクローンされたジョブの job_id のみに簡略化されています。FLAdminAPI の場合:

.. code-block:: python

    runner.api.clone_job(job_id)

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {'message': 'Cloned job 5d0eaa30-6936-4044-918e-cd9c3f5edf9b as: 4a2cf195-314d-4476-9ea5-c69bed397e3a',
    'job_id': '4a2cf195-314d-4476-9ea5-c69bed397e3a'},
    'raw': {'time': '2023-01-25 15:08:40.235304',
    'data': [{'type': 'string',
        'data': 'Cloned job 5d0eaa30-6936-4044-918e-cd9c3f5edf9b as: 4a2cf195-314d-4476-9ea5-c69bed397e3a'},
    {'type': 'success', 'data': ''}],
    'meta': {'status': 'ok',
    'info': '',
    'job_id': '4a2cf195-314d-4476-9ea5-c69bed397e3a'},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}

FLARE API での ``clone_job()``:

.. code-block:: python

    sess.clone_job(job_id)

.. code-block:: bash

    '4a2cf195-314d-4476-9ea5-c69bed397e3a'


ジョブの中止 (Abort Job)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``abort_job()`` コマンドは FLAdminAPI と FLARE API で同じで、必須の引数は job_id の文字列のみです。
コマンドの動作も同じままですが、出力は None に簡略化されています。FLAdminAPI の場合:

.. code-block:: python

    runner.api.abort_job(job_id)

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {'message': 'Abort signal has been sent to the server app.'},
    'raw': {'time': '2023-01-26 16:59:32.980711',
    'data': [{'type': 'string',
        'data': 'Abort signal has been sent to the server app.'},
    {'type': 'success', 'data': ''}],
    'meta': {'status': 'ok', 'info': ''},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}

FLARE API での ``abort_job()``:

.. code-block:: python

    sess.abort_job(job_id)

.. code-block:: bash

    None


ジョブの削除 (Delete Job)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
``delete_job()`` コマンドは FLAdminAPI と FLARE API で同じで、必須の引数は job_id の文字列のみです。
コマンドの動作も同じままですが、出力は何も返さないように簡略化されています。FLAdminAPI の場合:

.. code-block:: python

    runner.api.delete_job(job_id)

.. code-block:: bash

    {'status': <APIStatus.SUCCESS: 'SUCCESS'>,
    'details': {'message': 'Job 4a2cf195-314d-4476-9ea5-c69bed397e3a deleted.'},
    'raw': {'time': '2023-01-26 17:02:12.812807',
    'data': [{'type': 'string',
        'data': 'Job 4a2cf195-314d-4476-9ea5-c69bed397e3a deleted.'},
    {'type': 'success', 'data': ''}],
    'meta': {'status': 'ok', 'info': ''},
    'status': <APIStatus.SUCCESS: 'SUCCESS'>}}

FLARE API での ``delete_job()``:

.. code-block:: python

    sess.delete_job(job_id)

その他すべての FLAdminAPI コマンドの FLARE API への移行
--------------------------------------------------------------------
残りの FLAdminAPI コマンドは 2.4.0 で FLARE API に追加されました。
詳細については、上の表の備考および :mod:`FLARE API<nvflare.fuel.flare_api.flare_api>` の定義を参照してください。
