.. _site_policy_management:

****************************************
サイトポリシー管理
****************************************
各サイトは、以下の領域について独自のポリシーを定義できます。

    - リソース管理: ローカルの IT 部門のみが判断すべきシステムリソースの設定
    - 認可ポリシー: ユーザーがローカルサイト上で何をできる / できないかを決定するローカルの認可ポリシー
    - プライバシーポリシー: どの種類の研究 (study) を許可するか、およびローカルサイトの FL クライアントが生成した学習結果にどのようにプライバシー保護を追加するかを指定するローカルポリシー
    - ロギング設定: 各サイトは、システムが生成するログメッセージに対して独自のロギング設定を定義できます


ワークスペースの構造
====================
NVFLARE のポリシーファイルはワークスペースに保存されます。ローカルサイトのポリシーをサポートするため、ワークスペースに新しく "local" フォルダが追加されました。
"local" フォルダを含む、ワークスペースの完全な構造は次のとおりです。

.. code-block::
    :emphasize-lines: 2-11

    {WSROOT}
        startup
            fed_server|client.json
            Site cert, site private key, root certificate, and site
            start.sh
            …
        local
            resources.json.default
            authorization.json.default
            privacy.json.sample
            log_config.json.default
            resources.json
            authorization.json
            privacy.json
            log_config.json
            custom/
                local_code.xyz
        audit.txt
        log.txt
        1234567 (run)
                log.txt
                job_meta.json
                app_xxx
                    fl_app.txt
                    config
                        config_fed_client.json
                        …
                    custom
                            xyz.py
        234562 (run)
                log.txt
                job_meta.json
                app_xxx
                    fl_app.txt
                    config
                    custom

黄色でハイライトされている内容は、Provision プロセスによって生成されます。Provision が生成する ZIP パッケージには、startup と local という 2 つの
フォルダが含まれるようになりました。"startup" フォルダには、FL サーバーとの通信に必要なセキュリティ資格情報のほか、
一般的なシステム設定情報が含まれます。"local" フォルダには、ローカルポリシーのデフォルトおよび / またはサンプルが含まれます。組織管理者 (Org Admin) が
独自のポリシーを定義したい場合は、デフォルトを上書きする別ファイルを作成することで定義できます。これらのファイルが、
"local" フォルダ内でハイライトされていないものです。

組織管理者は、"local/custom" フォルダに追加のカスタムコードをインストールすることもできます。これにより、サイトはプライバシー制御のための
独自のカスタムフィルターを開発できます。

リソース管理ポリシー
--------------------
fed_server|client.json に含まれていた設定項目のうち、ローカルで判断すべきものは、ローカルの resources.json.default に移されました。

FL サーバー向けの resources.json.default の例を次に示します。

.. code-block::

    {
        "format_version": 2,
        "servers": [
            {
                "admin_storage": "transfer",
                "max_num_clients": 100,
                "heart_beat_timeout": 600,
                "num_server_workers": 4,
                "download_job_url": "http://download.server.com/",
                "compression": "Gzip"
            }
        ],
        "snapshot_persistor": {
            "path": "nvflare.app_common.state_persistors.storage_state_persistor.StorageStatePersistor",
            "args": {
                "uri_root": "/",
                "storage": {
                    "path": "nvflare.app_common.storages.filesystem_storage.FilesystemStorage",
                    "args": {
                        "root_dir": "/tmp/nvflare/snapshot-storage",
                        "uri_root": "/"
                    }
                }
            }
        },
        "components": [
            {
                "id": "job_scheduler",
                "path": "nvflare.app_common.job_schedulers.job_scheduler.DefaultJobScheduler",
                "args": {
                    "max_jobs": 1
                }
            },
            {
                "id": "job_manager",
                "path": "nvflare.apis.impl.job_def_manager.SimpleJobDefManager",
                "args": {
                    "uri_root": "/tmp/nvflare/jobs-storage",
                    "job_store_id": "job_store"
                }
            },
            {
                "id": "job_store",
                "path": "nvflare.app_common.storages.filesystem_storage.FilesystemStorage"
            }
        ]
    }


ご覧のとおり、組織管理者は再度 Provision プロセスを実行することなく、パラメータを変更したり、ストレージに別の Python オブジェクトを使用したりすることを決定できます。

FL クライアント向けの resources.json.default の例を次に示します。

.. code-block::

    {
        "format_version": 2,
        "client": {
            "retry_timeout": 30,
            "compression": "Gzip"
        },
        "components": [
            {
                "id": "resource_manager",
                "path": "nvflare.app_common.resource_managers.list_resource_manager.ListResourceManager",
                "args": {
                    "resources": {
                        "gpu": [
                            0,
                            1,
                            2,
                            3
                        ]
                    }
                }
            },
            {
                "id": "resource_consumer",
                "path": "nvflare.app_common.resource_consumers.gpu_resource_consumer.GPUResourceConsumer",
                "args": {
                    "gpu_resource_key": "gpu"
                }
            }
        ]
    }

ご覧のとおり、FL クライアントサイトの組織管理者は、再度 Provision プロセスを実行することなく、GPU の数やその他のパラメータを変更できます。

認可ポリシーの管理
------------------
組織管理者は、authorization.json でローカルの認可ポリシーを定義できます。

プライバシー管理
----------------
NVFLARE には、各サイトがクライアントの生成した学習結果に適用する独自のプライバシー保護ポリシーを定義できるようにするセキュリティ強化機能が備わっています。

なお、ここでの説明におけるデータプライバシー保護とは、特に次の脅威を指します。すなわち、送信者 (クライアント) が生成した学習結果の受信者 (サーバー) が、その学習結果をリバースエンジニアリングすることで学習データを発見・再構築できてしまう、という脅威です。

NVFLARE の以前のバージョンと同様に、主要なプライバシー保護技術はフィルタリングの仕組みです。フィルターには 2 種類あります。

    - タスクデータフィルター - タスクを実行するためにエグゼキューターを呼び出す前に、タスクデータに適用されます。フィルター処理されたタスクデータのみがタスクエグゼキューターに渡されます。
    - タスク結果フィルター - タスクエグゼキューターが生成したタスク結果をサーバーに返送する前に適用されます。フィルター処理された結果のみがサーバーに送信されます。

NVFLARE の以前のバージョンでは、ジョブ設定でフィルターを指定できるのは研究者のみでした。しかし、FL クライアントのデータプライバシーを保護することは、必ずしも研究者の関心事ではないかもしれません。データプライバシーの保護は組織管理者の関心事です。

NVFLARE では、組織管理者がデータプライバシー保護のためのフィルターを指定できます。ジョブにのみ適用される研究者指定のフィルターとは異なり、サイトのプライバシーポリシーで指定されたフィルターは、すべてのジョブに適用されます。これは Scope (スコープ) という概念によって実現されています。

スコープは、その中でジョブが実行される空間と考えることができます。例えば、FL プロジェクトの目的に応じて、プロジェクト管理者 (Project Admin) は研究を 2 つのフェーズに分けて実施すると決めるかもしれません。まず、公開されているデータセットを使用し、緩やかなデータプライバシー保護を適用した "public" スコープでジョブを実行します。アルゴリズムが確定した後、各サイト独自のデータセットを使用し、より厳格なデータプライバシー保護を適用した "private" スコープでジョブを実行します。

各スコープは次の属性を持ちます。

    - 名前 (Name) - スコープには一意の名前が必要です。プロジェクトの開始時に、すべてのサイトと協力してスコープとその名前を決めるのはプロジェクト管理者の役割です。
    - プロパティ (Properties) - そのスコープでエグゼキューターがタスクを実行する際に役立つ可能性のある追加プロパティを定義する任意のキー / 値。
    - タスクデータフィルター (Task Data Filters) - そのスコープ内のジョブのタスクデータに適用されるフィルター。
    - タスク結果フィルター (Task Result Filters) - そのスコープ内のジョブのタスク結果に適用されるフィルター。

以下はポリシーのサンプルです。

.. code-block:: json

    {
        "scopes": [
            {
                "name": "public",
                "properties": {
                "train_dataset": "/data/public/train",
                "val_dataset": "/data/public/val"
                },
                "task_result_filters": [
                {
                    "path": "nvflare.app_common.statistics.min_max_cleanser.AddNoiseToMinMax",
                    "args": {
                    "min_noise_level": 0.2,
                    "max_noise_level": 0.2
                    }
                },
                {
                    "path": "nvflare.app_common.filters.percentile_privacy.PercentilePrivacy",
                    "args": {
                    "percentile": 10,
                    "gamma": 0.02
                    }
                }
                ],
                "task_data_filters": [
                {
                    "path": "custom_filters.BadModelDetector"
                }
                ]
            },
            {
                "name": "private",
                "properties": {
                "train_dataset": "/data/private/train",
                "val_dataset": "/data/private/val"
                },
                "task_result_filters": [
                {
                    "path": "nvflare.app_common.statistics.min_max_cleanser.AddNoiseToMinMax",
                    "args": {
                    "min_noise_level": 0.1,
                    "max_noise_level": 0.1
                    }
                },
                {
                    "path": "nvflare.app_common.filters.svt_privacy.SVTPrivacy",
                    "args": {
                    "fraction": 0.1,
                    "epsilon": 0.2
                    }
                }
                ]
            }
        ],
        "default_scope": "public"
    }


ジョブのスコープは、meta のキー "scope" で指定します。ジョブがスコープを指定しない場合は、デフォルトのスコープが使用されます。

プライバシー処理ルール
----------------------
NVFLARE に組み込まれているプライバシー処理ルールは次のとおりです。

サイトが privacy.json を定義していない場合、プライバシー制御は適用されません。

ジョブがスコープ名を明示的に指定していない場合、サイトが指定した "default_scope" がそのジョブのスコープとして使用されます。サイトがデフォルトスコープを指定していない場合、そのジョブは拒否されます。このルールはジョブのデプロイ時に適用されます。

ジョブが指定したスコープがサイトのスコープ一覧に見つからない場合、そのジョブは拒否されます。このルールはジョブのデプロイ時に適用されます。

ジョブのスコープが見つかった場合 (デフォルトスコープとして、またはサイトのスコープ一覧で明示的に定義されている場合)、そのスコープのフィルター (存在する場合) が、ジョブが指定したフィルター (存在する場合) よりも先に適用されます。このルールはタスクの実行時に適用されます。

サイトポリシーの作成
--------------------
システムの整合性を確保し、エラーの可能性を最小限に抑えるため、以下の簡単な手順に従ってください。

1) 上書きしたいファイルのコピーを作成し、新しいファイルに一時的な名前を付けます。例:  cp resources.json.default my_resources.json
2) 新しいファイルを独自のポリシー定義で編集し、保存します
3) ファイルを正しい名前に変更します:  mv my_resources.json resources.json
