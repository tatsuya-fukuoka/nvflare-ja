##########################
システムアーキテクチャ
##########################

NVIDIA FLARE は「less is more (少ないほど豊かである)」という考えのもとに設計されており、スペックベースの設計原則を用いることで、本質的なこと (他の人が解決したがらない難しく面倒な問題への解決策) に注力し、他の人が実世界のアプリケーションでやりたいことをできるようにしています。FL はオープンエンドな領域であるため、スペックベースの設計により、他の人がさまざまなコンポーネントに対して独自の実装や解決策を持ち込めるようになっています。

.. _concepts_and_system_components:

************************************
概念とシステムコンポーネント
************************************

システムサービスオブジェクトのスペックベースプログラミング
============================================================
NVIDIA FLARE は、ストレージやジョブ定義管理などのための追加サービスを必要とします。こうしたサービスを実装する方法は数多くあります。例えば、ストレージはファイルシステム、AWS S3、あるいは何らかのデータベース技術で実装できます。同様に、ジョブ定義管理は単純なファイル読み取りでも、データベースや検索エンジンを用いた高度なソリューションでも実現できます。

NVIDIA FLARE でこれらいずれのソリューションも使えるようにするため、そうしたオブジェクトについてはスペックベースのアプローチを採用しています。こうした各サービスはインターフェース定義 (スペック) を提供し、すべての実装はそのスペックに従わなければなりません。スペックは実装に求められる振る舞いを定義します。同時に、これらのスペックに従ういくつかの実装も提供しています。

各システムコンポーネントについてのスペック定義と提供される実装は以下のとおりです。

.. _system_components:

システムコンポーネント
========================
これらのコンポーネントが StaticFileBuilder でどのように設定されるかについては、:ref:`project_yml` の例を参照してください。

ジョブ定義マネージャー
------------------------
ジョブ定義マネージャーの設定では、ジョブストレージに保存されたジョブ定義オブジェクトへのアクセスと操作を管理する Python オブジェクトを指定します。

システム予約のコンポーネント ID である job_manager は、project.yml ファイル内でジョブ定義マネージャーを表すために使用されます。

このコンポーネントは components.server セクションの 1 項目として指定されます。

この設定はサーバーのスタートアップキットの fed_server.json に含まれます。

:class:`Job Definition Manager Spec<nvflare.apis.job_def_manager_spec.JobDefManagerSpec>`

NVIDIA FLARE は、ジョブ定義オブジェクトのスキャンに基づく単純な実装を提供しています。

    - :class:`Simple Job Def Manager<nvflare.apis.impl.job_def_manager.SimpleJobDefManager>`

ジョブストレージ
^^^^^^^^^^^^^^^^^^
ジョブ定義は永続ストア (Simple Job Def Manager が使用) に保存されます。ジョブストレージの設定では、そのストアへのアクセスを管理する Python オブジェクトを指定します。

このコンポーネントは components.server セクションの 1 項目として指定されます。

この設定はサーバーのスタートアップキットの fed_server.json に含まれます。

.. note::

   デフォルトのストレージは `FilesystemStorage<nvflare.app_common.storages.filesystem_storage.FilesystemStorage>` であり、
   データを永続化するためにファイルシステム上で利用可能なパスを使うように設定されています。代わりに他の実装を使うこともでき、
   その場合は別の引数や設定が必要になることがあります。

ジョブスケジューラー
----------------------
ジョブスケジューラーは、次に実行するジョブを決定する役割を担います。ジョブスケジューラーの設定では、ジョブスケジューラーの Python オブジェクトを指定します。

システム予約のコンポーネント ID である job_scheduler は、project.yml ファイル内でジョブスケジューラーを表すために使用されます。

このコンポーネントは components.server セクションの 1 項目として指定されます。

この設定はサーバーのスタートアップキットの fed_server.json に含まれます。

:class:`Job Scheduler Spec<nvflare.apis.job_scheduler_spec.JobSchedulerSpec>`

NVIDIA FLARE は、冒頭で説明したリソースベースのスケジューリングを行うジョブスケジューラーのデフォルト実装を提供しています。

    - :class:`Default Job Scheduler<nvflare.app_common.job_schedulers.job_scheduler.DefaultJobScheduler>`

ストレージ
------------
ストレージはジョブストレージおよびジョブ実行状態ストレージで使用されます。詳細は該当するセクションを参照してください。

:class:`Storage Spec<nvflare.apis.storage.StorageSpec>`

NVIDIA FLARE は 2 つの単純なストレージ実装を提供しています。

    - :class:`File System Storage<nvflare.app_common.storages.filesystem_storage.FilesystemStorage>`
    - :class:`AWS S3 Storage<nvflare.app_common.storages.s3_storage.S3Storage>`

リソースマネージャー
----------------------
リソースマネージャーは、FL クライアント上のジョブリソースを管理する役割を担います。リソースマネージャーの設定では、リソースマネージャーの Python オブジェクトを指定します。

システム予約のコンポーネント ID である resource_manager は、project.yml ファイル内でリソースマネージャーを表すために使用されます。

このコンポーネントは components.client セクションの 1 項目として指定されます。

この設定は FL クライアントのスタートアップキットの fed_client.json に含まれます。

:class:`Resource Manager Spec<nvflare.apis.resource_manager_spec.ResourceManagerSpec>`

NVIDIA FLARE は、リソースを項目のリストとして管理する単純なリソースマネージャーを提供しています。

    - :class:`List Resource Manager<nvflare.app_common.resource_managers.list_resource_manager.ListResourceManager>`

リソースコンシューマー
------------------------
リソースコンシューマーは、FL クライアント上でジョブリソースを消費および/または初期化する役割を担います。リソースコンシューマーの設定では、リソースコンシューマーの Python オブジェクトを指定します。

この設定は FL クライアントのスタートアップキットの fed_client.json に含まれます。

システム予約のコンポーネント ID である resource_consumer は、project.yml ファイル内でリソースコンシューマーを表すために使用されます。

このコンポーネントは components.client セクションの 1 項目として指定されます。

:class:`Resource Consumer Spec<nvflare.apis.resource_manager_spec.ResourceConsumerSpec>`

NVIDIA FLARE は GPU リソースコンシューマーを提供しています。

    - :class:`GPU Resource Consumer<nvflare.app_common.resource_consumers.gpu_resource_consumer.GPUResourceConsumer>`

スナップショットの永続化
--------------------------
ジョブ実行状態は、ジョブ実行状態ストレージによってスナップショットとして永続化されます。

ジョブ実行状態ストレージ
^^^^^^^^^^^^^^^^^^^^^^^^^^
ジョブ実行状態は永続ストアに保存されます。ジョブ実行状態ストレージの設定では、そのストアへのアクセスを管理する Python オブジェクトを指定します。

この設定はサーバーのスタートアップキットの fed_server.json に含まれます。
