.. _application:

##################################
NVIDIA FLARE アプリケーション
##################################

NVIDIA FLARE アプリケーションは、サーバーとクライアントがどのように実行されるべきかを定義します。
1つのジョブの範囲においては、各サイトは1つのアプリケーションのみを実行することに注意してください。

アプリケーションフォルダの構造は次のようにする必要があります::

    app_folder/
        config/
            config_fed_client.json [required if this app needs to be deployed to clients]
            config_fed_server.json [required if this app needs to be deployed to server]
        custom/
            [any of your custom code].py
            [another file with custom code].py
            ...
        resources/
            log_config.json

.. note::

    アプリケーションは、ジョブの deploy_map 設定によって特定のサイト上で実行するように構成できることに注意してください。
    アプリケーションはジョブなしで実行することもできます。
    その場合は、アプリケーションを単にジョブとして送信するだけで、全サイトを対象とするデフォルトのデプロイマップが使用されます。

.. note::

    同一のアプリケーションをサーバーとクライアントの両方にデプロイする場合、そのアプリケーションは
    ``config_fed_server.json`` と ``config_fed_client.json`` の両方を含めることができます。

.. note::

    設定用のJSONファイル config_fed_server.json および config_fed_client.json は、FLアプリケーションの
    ルートフォルダに置くことも、FLアプリケーションのサブフォルダ(例: config)に置くこともできます。

********************************
FLサーバーの設定
********************************

``config_fed_server.json`` はFLサーバーの設定ファイルです。

例:

.. literalinclude:: ../../resources/config_fed_server.json
    :language: json

.. csv-table::
    :header: キー, 説明

    format_version, この設定に対応する NVIDIA FLARE のバージョン
    server, ハートビートがタイムアウトするまでの秒数を指定する heart_beat_timeout など、サーバー固有の属性を指定します
    task_data_filters, "サーバーから送出されるデータに適用するフィルタです。 :ref:`filters` を参照してください"
    task_result_filters, "サーバーに到着するデータに適用するフィルタです。 :ref:`filters` を参照してください"
    components, 使用するすべてのコンポーネント
    workflows, "使用するワークフローです。 :ref:`controllers` を参照してください"

********************************
FLクライアントの設定
********************************

``config_fed_client.json`` はFLクライアントの設定ファイルです。

例:

.. literalinclude:: ../../resources/config_fed_client.json
    :language: json

.. csv-table::
    :header: キー, 説明

    format_version, この設定に対応する NVIDIA FLARE のバージョン
    executors, タスクとエグゼキュータの設定です。現在はトレーナーもここに含まれます
    task_data_filters, "クライアントに到着するデータに適用するフィルタです。 :ref:`filters` を参照してください"
    task_result_filters, "クライアントから送出されるデータに適用するフィルタです。 :ref:`filters`"
    components, 使用するすべてのコンポーネント


.. _custom_code:

************************
カスタムコード
************************

:ref:`programming_guide` に従って、独自のコンポーネントを記述し、独自のコードを持ち込む(BYOC)ことができます。

アプリケーションでそれを使用するには、そのコードをアプリケーションフォルダの "custom" フォルダ内に配置し、
BYOCが有効かつ許可されていることを確認してください。

サーバーまたはクライアントの設定では、path を使ってそのコンポーネントを参照します。

カスタムコードの設定例
==============================
例えば、custom フォルダ内の ``my_trainer.py`` というファイルに ``SimpleTrainer`` クラスが格納されている場合、
それをエグゼキュータとして設定するには、クライアント設定に次のような記述が必要です::

    ...
    "executor": {
      "path": "my_trainer.SimpleTrainer",
      "args": {}
    },
    ...

.. note::

    ここではエグゼキュータのタスク設定は省略しています。

詳しくは :ref:`getting_started` を参照してください。

.. _troubleshooting_byoc:

BYOCのトラブルシューティング
========================================
2.2.1では認可の仕組みが再設計され、BYOCはプロビジョニング時の設定では制御されなくなり、代わりに各サイトの
authorization.json (ワークスペースの local フォルダ内)によって制御されるようになりました。BYOCは権利(right)であり、
特定のロール、さらには組織やユーザーに対して制限することができます。詳細は :ref:`federated_authorization` を参照してください。

************
リソース
************

resources フォルダ内には ``log_config.json`` が必要です。
このファイルはPythonのロガーが使用します。
ログの挙動をカスタマイズする必要がない場合は、サンプルアプリケーションフォルダのいずれかにある
``log_config.json`` をそのまま使用できます。

.. literalinclude:: ../../resources/log_config.json
