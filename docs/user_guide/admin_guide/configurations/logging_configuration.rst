.. _logging_configuration:

####################
ロギング設定
####################

FLARE は `dictConfig API <https://docs.python.org/3/library/logging.config.html#logging.config.dictConfig>`_ を用いた Python の logging を、`設定ディクショナリスキーマ <https://docs.python.org/3/library/logging.config.html#configuration-dictionary-schema>`_ に従って使用します。
FLARE のロガーは、異なるレベルできめ細かく制御できるようにするため、ドット区切りのロガー名を用いてパッケージレベルの階層に従うよう設計されています。

すべての NVFLARE サブシステム向けに、コンソールレベルの色付け、ログ、エラーログ、構造化された json ログ、FL 学習ログ用のハンドラをあらかじめ設定した :ref:`デフォルトのロギング設定 <default_logging_configuration>` ファイル **log_config.json.default** を提供しています。

デフォルト設定を上書きするには :ref:`ロギング設定の変更 <modifying_logging_configurations>` でファイルを変更するか、
:ref:`動的ロギング設定コマンド <dynamic_logging_configuration_commands>` の ``configure_site_log`` および ``configure_job_log`` を使って実行時にロギング設定を変更します。

************************
ロギング設定と機能
************************

.. _default_logging_configuration:

デフォルトのロギング設定
==============================

デフォルトのロギング設定 json ファイル (**log_config.json.default**、``LogMode.FULL``) は、formatters、handlers、loggers という 3 つの主要セクションに分かれています。
このファイルは :github_nvflare_link:`log_config.json <nvflare/fuel/utils/log_config.json>` にあります。
詳細については `設定ディクショナリスキーマ <https://docs.python.org/3/library/logging.config.html#configuration-dictionary-schema>`_ を参照してください。

.. code-block:: json

    {
        "version": 1,
        "disable_existing_loggers": false,
        "formatters": {
            "baseFormatter": {
                "()": "nvflare.fuel.utils.log_utils.BaseFormatter",
                "fmt": "%(asctime)s - %(name)s - %(levelname)s - %(fl_ctx)s - %(message)s"
            },
            "consoleFormatter": {
                "()": "nvflare.fuel.utils.log_utils.ColorFormatter",
                "fmt": "%(asctime)s - %(name)s - %(levelname)s - %(fl_ctx)s - %(message)s"
            },
            "jsonFormatter": {
                "()": "nvflare.fuel.utils.log_utils.JsonFormatter",
                "fmt": "%(asctime)s - %(name)s - %(fullName)s - %(levelname)s - %(fl_ctx)s - %(message)s"
            }
        },
        "filters": {
            "FLFilter": {
                "()": "nvflare.fuel.utils.log_utils.LoggerNameFilter",
                "logger_names": ["custom", "nvflare.app_common", "nvflare.app_opt"]
            }
        },
        "handlers": {
            "consoleHandler": {
                "class": "logging.StreamHandler",
                "level": "DEBUG",
                "formatter": "consoleFormatter",
                "filters": [],
                "stream": "ext://sys.stdout"
            },
            "logFileHandler": {
                "class": "logging.handlers.RotatingFileHandler",
                "level": "DEBUG",
                "formatter": "baseFormatter",
                "filename": "log.txt",
                "mode": "a",
                "maxBytes": 20971520,
                "backupCount": 10
            },
            "errorFileHandler": {
                "class": "logging.handlers.RotatingFileHandler",
                "level": "ERROR",
                "formatter": "baseFormatter",
                "filename": "error_log.txt",
                "mode": "a",
                "maxBytes": 20971520,
                "backupCount": 10
            },
            "jsonFileHandler": {
                "class": "logging.handlers.RotatingFileHandler",
                "level": "DEBUG",
                "formatter": "jsonFormatter",
                "filename": "log.json",
                "mode": "a",
                "maxBytes": 20971520,
                "backupCount": 10
            },
            "FLFileHandler": {
                "class": "logging.handlers.RotatingFileHandler",
                "level": "DEBUG",
                "formatter": "baseFormatter",
                "filters": ["FLFilter"],
                "filename": "log_fl.txt",
                "mode": "a",
                "maxBytes": 20971520,
                "backupCount": 10,
                "delay": true
            }
        },
        "loggers": {
            "root": {
                "level": "INFO",
                "handlers": ["consoleHandler", "logFileHandler", "errorFileHandler", "jsonFileHandler", "FLFileHandler"]
            }
        }
    }

ログレコードをコンソールや各種ログファイルに出力するために、さまざまなフォーマッタ、フィルタ、ハンドラを使用します。これらについては以下で詳しく説明します。

フォーマッタ
==================

`フォーマッタ <https://docs.python.org/3/library/logging.html#formatter-objects>`_ は、ログレコードの形式を指定するために使用します。
デフォルトで便利なフォーマッタをいくつか提供しています。

BaseFormatter
-------------
:class:`BaseFormatter<nvflare.fuel.utils.log_utils.BaseFormatter>` はデフォルトのフォーマッタであり、他の FLARE フォーマッタの基底クラスとして機能します。

- `ログレコード属性 <https://docs.python.org/3/library/logging.html#logrecord-attributes>`_ を伴う **fmt** や、**datefmt** の `日付フォーマット文字列 <https://docs.python.org/3/library/logging.html#logging.Formatter.formatTime>`_ など、標準の `Formatter <https://docs.python.org/3/library/logging.html#logging.Formatter>`_ の引数をすべて指定できます。
- **record.name** はロガーのベース名に短縮され、**record.fullName** にはロガーのフルネームが設定されます。

設定例と出力例:

.. code-block:: json

    "baseFormatter": {
        "()": "nvflare.fuel.utils.log_utils.BaseFormatter",
        "fmt": "%(asctime)s - %(name)s - %(fullName)s - %(levelname)s - %(fl_ctx)s - %(message)s",
        "datefmt": "%m-%d-%Y- %H:%M:%S"
    }

.. code-block:: shell

    01-14-2025 14:44:46 - PTInProcessClientAPIExecutor - nvflare.app_opt.pt.in_process_client_api_executor.PTInProcessClientAPIExecutor - INFO - [identity=site-1, run=fc711945-a7cf-4834-9fc4-aa9cb60e327b, peer=example_project, peer_run=fc711945-a7cf-4834-9fc4-aa9cb60e327b, task_name=train, task_id=a16b7a02-b2ea-4eb5-895a-b40d507b2c5c] - execute for task (train)


ColorFormatter
--------------
:class:`ColorFormatter<nvflare.fuel.utils.log_utils.ColorFormatter>` は ANSI カラーコードを使用して、ログレベルやロガー名に基づいてログレコードを整形します。

よく使われる色とログレベルのデフォルトマッピングのために、:class:`ANSIColor<nvflare.fuel.utils.log_utils.ANSIColor>` クラスを提供しています。
色をカスタマイズするには、ANSIColor.COLORS で指定されている色名の文字列、または ANSI カラーコード (追加の ANSI 引数にはセミコロンを使用できます) を使用します。

- **level_colors**: levelname と ANSI カラーの dict。デフォルトは ANSIColor.DEFAULT_LEVEL_COLORS です。
- **logger_colors**: loggername と ANSI カラーの dict。デフォルトは {} です。

設定例:

.. code-block:: json

    "consoleFormatter": {
        "()": "nvflare.fuel.utils.log_utils.ColorFormatter",
        "fmt": "%(asctime)s - %(name)s - %(levelname)s - %(fl_ctx)s - %(message)s",
        "level_colors": {
            "NOTSET": "grey",
            "DEBUG": "grey",
            "INFO": "grey",
            "WARNING": "yellow",
            "ERROR": "red",
            "CRITICAL": "bold_red"
        },
        "logger_colors": {
            "nvflare.app_common": "blue",
            "nvflare.app_opt": "38;5;212"
        }
    }


JsonFormatter
-------------
:class:`JsonFormatter<nvflare.fuel.utils.log_utils.JsonFormatter>` はログレコードを json 文字列に変換します。

設定例と出力例:

.. code-block:: json

    "jsonFormatter": {
        "()": "nvflare.fuel.utils.log_utils.JsonFormatter",
        "fmt": "%(asctime)s - %(name)s - %(levelname)s - %(fl_ctx)s - %(message)s"
    }

.. code-block:: json

    {"asctime": "2025-01-14 14:44:46,559", "name": "PTInProcessClientAPIExecutor", "fullName": "nvflare.app_opt.pt.in_process_client_api_executor.PTInProcessClientAPIExecutor", "levelname": "INFO", "fl_ctx": "[identity=site-1, run=fc711945-a7cf-4834-9fc4-aa9cb60e327b, peer=example_project, peer_run=fc711945-a7cf-4834-9fc4-aa9cb60e327b, task_name=train, task_id=a16b7a02-b2ea-4eb5-895a-b40d507b2c5c]", "message": "execute for task (train)"}


フィルタ
============

`フィルタ <https://docs.python.org/3/library/logging.html#filter-objects>`_ は、指定した条件に基づいて特定のログレコードのみを通過させるために使用します。

LoggerNameFilter
----------------
:class:`LoggerNameFilter<nvflare.fuel.utils.log_utils.LoggerNameFilter>` は logger_names のリストに基づいてロガーをフィルタリングします。
フィルタはロガーの階層を利用するため、指定した名前の子孫にあたるロガーもフィルタを通過します。
デフォルトでは、LoggerNameFilter は allow_all_error_logs が設定されており、logger_names に含まれるロガー以外からのログであっても、INFO より高いレベルのログはすべて通過させます。

- **logger_names**: フィルタを通過させるロガー名のリスト
- **exclude_logger_names**: フィルタを通過させないロガー名のリスト (logger_names による許可よりも優先されます)
- **allow_all_error_logs**: logger_names に含まれるロガー以外からのログであっても、levelno > logging.INFO のログレコードをすべてフィルタを通過させます。デフォルトは True です。

これを FLFilter で活用しており、FL 学習やカスタムコードに関連するロガーをフィルタリングします。

.. code-block:: json

    "FLFilter": {
        "()": "nvflare.fuel.utils.log_utils.LoggerNameFilter",
        "logger_names": ["custom", "nvflare.app_common", "nvflare.app_opt"]
    }

ハンドラ
============
`ハンドラ <https://docs.python.org/3/library/logging.html#handler-objects>`_ は、指定された Formatter や Filter を (順次) 適用しながら、ログレコードを送信先に送る役割を担います。

consoleHandler
--------------

consoleHandler は `StreamHandler <https://docs.python.org/3/library/logging.handlers.html#streamhandler>`_ を使用して、sys.stdout などのストリームにログ出力を送ります。

設定例:

.. code-block:: json

    "consoleHandler": {
        "class": "logging.StreamHandler",
        "level": "DEBUG",
        "formatter": "consoleFormatter",
        "filters": ["FLFilter"],
        "stream": "ext://sys.stdout"
    }


FileHandlers
------------
`FileHandlers <https://docs.python.org/3/library/logging.handlers.html#filehandler>`_ を使用して、異なる形式でフォーマットされフィルタされたログレコードを異なるファイルに送ります。

あらかじめ設定されたハンドラでは、より具体的には `RotatingFileHandler <https://docs.python.org/3/library/logging.handlers.html#rotatingfilehandler>`_ を利用して、一定のファイルサイズに達した後にバックアップファイルへローテーションします。
FLARE は ``filename`` を、ワークスペースのルートディレクトリからの相対パス (サイトのログファイルの場合)、または実行ディレクトリからの相対パス (ジョブのログファイルの場合) として動的に解釈します。

設定例:

.. code-block:: json

    "logFileHandler": {
        "class": "logging.handlers.RotatingFileHandler",
        "level": "DEBUG",
        "formatter": "baseFormatter",
        "filename": "log.txt",
        "mode": "a",
        "maxBytes": 20971520,
        "backupCount": 10
    }

以下のログファイルハンドラがあらかじめ設定されています。

- logFileHandler: baseFormatter を使用してすべてのログを ``log.txt`` に書き込みます
- errorFileHandler: baseFormatter とレベル "ERROR" を使用して、エラーレベルのログを ``error_log.txt`` に書き込みます
- jsonFileHandler: jsonFormatter を使用して json 形式のログを ``log.json`` に書き込みます
- FLFileHandler: baseFormatter と FLFilter を使用して、FL 学習ログとカスタムログを ``log_fl.txt`` に書き込みます

.. _loggers:

ロガー
============

ロガーは logger セクションでレベルとハンドラを設定できます。

root ロガーは INFO レベルで定義し、必要なハンドラを追加します。

.. code-block:: json

    "root": {
        "level": "INFO",
        "handlers": ["consoleHandler", "logFileHandler", "errorFileHandler", "jsonFileHandler", "FLFileHandler"]
    }

ロガーは階層構造を持つため、ドット区切りの名前を使って個別のロガーを設定できます。
さらに、中間にあたる親ロガーもすでに作成されており、設定可能です。

カスタムコード用のロガーを作成する際には、ユーザー向けのカスタムロガー関数を提供しています。

:func:`custom_logger<nvflare.fuel.utils.log_utils.custom_logger>`: あるロガーから、ロガー名の先頭に "custom" を付加した新しいロガーを返します。
これにより、カスタムロガーからのログがデフォルトの FLFilter を通過できるようになり、"concise" モードでもログが表示されます。

FLARE のコード用のロガーを作成する際には、パッケージのロガー階層に従うのを助ける開発者向け関数をいくつか提供しています。

- クラス用の :func:`get_obj_logger<nvflare.fuel.utils.log_utils.get_obj_logger>`
- スクリプト用の :func:`get_script_logger<nvflare.fuel.utils.log_utils.get_script_logger>`
- モジュール用の :func:`get_module_logger<nvflare.fuel.utils.log_utils.get_module_logger>`


.. _modifying_logging_configurations:

************************
ロギング設定の変更
************************

.. _log_config_argument:

ログ設定引数
==================
ログ設定引数を提供しています (シミュレータモードでは ``-l`` または ``log_config``、POC モードおよび本番モードの動的ロギング管理コマンドでは ``config``)。
この引数には次のいずれかを指定できます。

- ログ設定 json ファイル (``/path/to/my_log_config.json``、``my_log_config.json``)
- 定義済みのコンソール :class:`LogMode<nvflare.fuel.utils.log_utils.LogMode>` (``concise``、``full``、``verbose``)

    - ``concise`` (シミュレータモードのデフォルト): 簡略化されたログ属性で FL 学習ログ向けの FLFilter を適用します
    - ``full`` (POC モードおよび本番モードのワークスペースにおけるデフォルト): 完全な info レベルのログ
    - ``verbose``: 詳細なログ属性を伴う debug レベルのログ

- ログレベル名または番号 (``debug``、``info``、``warning``、``error``、``critical``、``30``)
- 管理コマンドのみ: ワークスペースにある現在のログ設定ファイル log_config.json を読み込みます (``reload``)

.. _fl_log_level_env_var:

FL_LOG_LEVEL 環境変数
==============================

``FL_LOG_LEVEL`` 環境変数を使用すると、コマンドライン引数や API パラメータを渡さずにログ設定を行えます。
上記の :ref:`ログ設定引数 <log_config_argument>` と同じ値 (``concise``、``full``、``verbose``、ファイルパス、またはログレベル) を受け付けます。

この環境変数は、シミュレータ、POC、本番のすべてのモードで適用されます。

**優先順位** (高い順):

1. 明示的なパラメータ (``-l`` CLI フラグまたは ``log_config`` API 引数)
2. ``FL_LOG_LEVEL`` 環境変数
3. デフォルト (シミュレータ CLI では ``concise``、POC / 本番ではワークスペースの ``log_config.json``)

使用例:

.. code-block:: shell

    # Only show error level logs
    FL_LOG_LEVEL=error python job.py

    # Use verbose logging
    FL_LOG_LEVEL=verbose nvflare simulator -w /tmp/nvflare/workspace -n 2 -t 2 job_folder

    # Set for all NVFLARE processes in the session
    export FL_LOG_LEVEL=error


シミュレータのログ設定
==============================

ユーザーは、シミュレータコマンドの ``-l`` シミュレータ :ref:`ログ設定引数 <log_config_argument>` でログ設定を指定できます。

.. code-block:: shell

    nvflare simulator -w /tmp/nvflare/hello-numpy -n 2 -t 2 hello-world/hello-numpy -l log_config.json

または、Job API のシミュレータ実行の ``log_config`` 引数を使用します。

.. code-block:: python

    job.simulator_run("/tmp/nvflare/hello-numpy", log_config="log_config.json")

POC のログ設定
========================
POC ワークスペースを検索すると、次のようなファイルが見つかります。

.. code-block:: shell

    find /tmp/nvflare/poc  -name "log_config.json*"

    /tmp/nvflare/poc/server/local/log_config.json.default
    /tmp/nvflare/poc/site-1/local/log_config.json.default
    /tmp/nvflare/poc/site-2/local/log_config.json.default

変更を加えるには ``log_config.json`` を追加できます。

また、:ref:`動的ロギング設定コマンド <dynamic_logging_configuration_commands>` の使用も推奨します。

スタートアップキットのログ設定
========================================

ログ設定ファイルは、スタートアップキットの local ディレクトリ配下にあります。

スタートアップキットのワークスペースで ``log_config.json.*`` ファイルを検索すると、次のファイルが見つかります。

.. code-block:: shell

    find . -name "log_config.json.*"

    ./site-1/local/log_config.json.default
    ./site-2/local/log_config.json.default
    ./server1/local/log_config.json.default

サーバーの ``log_config.json.default`` は、FL サーバーおよびクライアントが使用するデフォルトのロギング設定です。デフォルトを上書きするには、
``log_config.json.default`` を ``log_config.json`` に変更し、設定を修正します。

また、:ref:`動的ロギング設定コマンド <dynamic_logging_configuration_commands>` の使用も推奨します。

.. _dynamic_logging_configuration_commands:

******************************
動的ロギング設定コマンド
******************************

FLARE システムを実行しているとき (POC モードまたは本番モード)、サイトログとジョブログという 2 種類のログがあります。
現在のサイトログ設定は、サイトログに加えて、そのサイトで新たに開始されるジョブのログ設定にも使用されます。
ワークスペース内に生成されたログにアクセスする方法については、:ref:`access_server_workspace` および :ref:`client_workspace` を参照してください。

FLARE システムの実行中に、サイトレベルまたはジョブレベルのロギングを動的に設定できるように、2 つの管理コマンドを提供しています。
これらのコマンドの効果は、再設定されるまで、あるいは対応するサイトやジョブが実行されている間、持続します。
ただし、これらのコマンドはワークスペース内のログ設定ファイルを上書きしません。ログ設定ファイルは "reload" を使って再読み込みできます。

- **target**: ``server``、``client <clients>...``、または ``all``
- **config**: ログ設定引数には次のいずれかを指定できます (詳細は上記の :ref:`ログ設定引数 <log_config_argument>` を参照してください)。

    - json ログ設定ファイルへのパス (``/path/to/my_log_config.json``)
    - 定義済みのログモード (``concise``、``full``、``verbose``)
    - ログレベル名または番号 (``debug``、``info``、``warning``、``error``、``critical``、``30``)
    - ワークスペースにある現在のログ設定ファイル log_config.json を読み込む (``reload``)

対象サイトのロギングを設定するには (実行中のジョブには影響しません):

.. code-block:: shell

    configure_site_log target config

対象ジョブのロギングを設定するには (ジョブが実行中である必要があります):

.. code-block:: shell

    configure_job_log job_id target config

コマンドの使い方については :ref:`operating_nvflare` を、デフォルトの認可ポリシーについては :ref:`command_categories` を参照してください。
