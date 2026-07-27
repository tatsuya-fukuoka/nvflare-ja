.. _deploy_prepare_command:

####################
デプロイコマンド
####################

``nvflare deploy`` は、すでに作成済みのサーバーまたはクライアントのスタートアップキットを、
サイトのデプロイランタイム向けに準備します。最初にサポートされるサブコマンドは
``nvflare deploy prepare`` です。

``deploy prepare`` は、アイデンティティ、証明書、スタートアップキット、Kubernetes クラスタを
作成しません。まず ``nvflare provision`` または分散型の ``nvflare cert`` / ``nvflare package``
ワークフローを使用し、その後、Docker、Kubernetes、Slurm で実行すべきサーバーまたはクライアントの
各キットに対して ``deploy prepare`` を実行してください。

Kubernetes のデプロイワークフローについては :ref:`helm_chart` を参照してください。Slurm の
デプロイワークフローとセキュリティチェックリストについては :ref:`slurm_job_launcher` を
参照してください。ジョブレベルのランタイム設定については :ref:`launcher_spec` を参照してください。

**********
使い方
**********

.. code-block:: none

   nvflare deploy prepare <startup-kit-dir> [--output <prepared-kit-dir>] [--config <runtime-config.yaml>]

引数:

- ``<startup-kit-dir>``: 既存のサーバーまたはクライアントのスタートアップキットディレクトリです。
- ``--kit``: ``<startup-kit-dir>`` の名前付きオプション形式のエイリアスです。
- ``--output``: 準備されたキットのコピーを配置するディレクトリです。デフォルトは
  ``<startup-kit-dir>/prepared/<runtime>`` です。
- ``--config``: YAML 形式のランタイム設定です。デフォルトは
  ``<startup-kit-dir>/config.yaml`` です。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

デフォルトの規約:

ランタイム設定を ``config.yaml`` としてスタートアップキット内に配置し、
``nvflare deploy prepare <startup-kit-dir>`` を実行します。コマンドはその設定から ``runtime`` を
読み取り、準備したコピーを ``<startup-kit-dir>/prepared/<runtime>`` （たとえば ``docker`` 、
``k8s`` 、 ``slurm`` ）に書き込みます。別のパスにある設定ファイルを読み込むには ``--config`` を、
準備したキットを別の場所に書き出すには ``--output`` を使用します。

管理者用スタートアップキットは、親のサーバープロセスやクライアントプロセスを実行しないため、
``deploy prepare`` ではサポートされません。

入力キットは読み取り専用として扱われます。ランタイム固有のファイルは、準備された出力ディレクトリに
書き込まれます。

********************
Docker 設定
********************

``docker.yaml`` の例:

.. code-block:: yaml

   runtime: docker

   parent:
     docker_image: registry.example.com/nvflare-site:2.8
     network: nvflare-network

   job_launcher:
     default_python_path: /usr/local/bin/python
     default_job_env:
       NCCL_P2P_DISABLE: "1"
     default_job_container_kwargs:
       shm_size: 8g
       ipc_mode: host

トップレベルのキー:

- ``runtime``: 必須です。 ``docker`` でなければなりません。
- ``parent``: 親のサーバー/クライアントコンテナ用の必須マッピングです。
- ``job_launcher``: ジョブごとの Docker コンテナのデフォルト値を指定する任意のマッピングです。

``parent`` のキー:

- ``docker_image``: ``startup/start_docker.sh`` が使用する親イメージです。必須です。
- ``network``: 親コンテナとジョブコンテナ用の Docker ネットワークです。デフォルトは
  ``nvflare-network`` です。

``job_launcher`` のキー:

- ``default_python_path``: ジョブコンテナ内で使用される Python 実行ファイルです。ただしジョブが
  ``launcher_spec[site]["docker"]["python_path"]`` で上書きした場合は、そちらが優先されます。
- ``default_job_env``: すべての Docker ジョブコンテナに注入される環境変数です。
- ``default_job_container_kwargs``: すべてのジョブコンテナに適用される Docker SDK のコンテナ
  kwargs です。 ``volumes`` 、 ``mounts`` 、 ``network`` 、 ``environment`` 、 ``command`` 、
  ``name`` 、 ``detach`` 、 ``auto_remove`` 、 ``user`` 、 ``working_dir`` 、 ``image`` など、
  ランチャーが制御するキーは拒否されます。サイトのデフォルトのジョブイメージを設定するには、
  ``local/study_runtime.yaml`` で ``studies.<study>.container.image`` を設定してください。

準備と起動:

.. code-block:: shell

   nvflare deploy prepare ./site-1 --config docker.yaml --output ./site-1-docker
   cd ./site-1-docker
   ./startup/start_docker.sh

このコマンドは次のファイルを書き出します:

- ``startup/start_docker.sh``
- ``DockerJobLauncher`` を反映するようパッチされた ``local/resources.json.default``
- パッチされた ``local/comm_config.json``
- 存在しない場合は ``local/study_runtime.yaml`` のテンプレート（すでに ``study_data.yaml`` を
  持つレガシーキットではスキップされます）

********************
K8s 設定
********************

``k8s.yaml`` の例:

.. code-block:: yaml

   runtime: k8s
   namespace: nvflare

   parent:
     docker_image: registry.example.com/nvflare-site:2.8
     image_pull_secrets:
       - registry-credentials
     parent_port: 8102
     workspace_pvc: nvflws
     workspace_mount_path: /var/tmp/nvflare/workspace
     python_path: /usr/local/bin/python3
     resources:
       requests:
         cpu: "2"
         memory: 8Gi
     pod_security_context: {}

   job_launcher:
     config_file_path: null
     pending_timeout: 300
     default_python_path: /usr/local/bin/python3
     image_pull_secrets:
       - job-registry-credentials
     job_pod_security_context: {}

トップレベルのキー:

- ``runtime``: 必須です。 ``k8s`` でなければなりません。
- ``namespace``: 親 Pod とジョブ Pod 用の Kubernetes 名前空間です。デフォルトは
  ``default`` です。
- ``server_service_name``: FL サーバー用の Kubernetes Service 名です（任意）。
- ``parent``: 生成される親 Helm チャート用の必須マッピングです。
- ``job_launcher``: 動的に起動されるジョブ Pod 用の任意のマッピングです。

``parent`` のキー:

- ``docker_image``: Helm チャートが使用する親イメージです。必須です。
- ``image_pull_secrets``: 親のサーバー/クライアント Pod に ``imagePullSecrets`` として
  レンダリングする、既存の Kubernetes Secret 名の任意のリストです。チャートをインストールする前に、
  対象の名前空間にこれらのレジストリプル用 Secret を作成してください。
  この設定は、生成される親 Pod のチャートに適用されます。動的に起動されるジョブ Pod には
  ``job_launcher.image_pull_secrets`` を使用してください。
- ``parent_port``: ジョブ Pod が親 Pod の FLARE プロセスに到達するために使用するポートです。
  デフォルトは ``8102`` です。
- ``workspace_pvc``: ランタイムワークスペースを含む PVC クレームです。デフォルトは
  ``nvflws`` です。
- ``workspace_mount_path``: 親 Pod のワークスペースのマウントパスです。デフォルトは
  ``/var/tmp/nvflare/workspace`` です。この値は Kubernetes ジョブランチャーの設定にも
  書き込まれるため、ジョブ Pod はコンテナ内で同じワークスペースパスを使用します。
- ``python_path``: 親 Pod のコマンドが使用する Python 実行ファイルです。デフォルトは
  ``/usr/local/bin/python3`` です。
- ``resources``: ``values.yaml`` にレンダリングされる親 Pod のリソースです。
- ``pod_security_context``: ``values.yaml`` にレンダリングされる親 Pod のセキュリティ
  コンテキストです。

``job_launcher`` のキー:

- ``config_file_path``: ``K8sJobLauncher`` が使用する kubeconfig のパスです。クラスタ内設定を
  使用する場合は ``null`` を指定します。この場合、Kubernetes Python クライアントは Pod の
  ServiceAccount トークンを使用します。
- ``pending_timeout``: ジョブ Pod が ``Pending`` を抜けるまで待機する秒数です。
- ``default_python_path``: ジョブ Pod 内で使用される Python 実行ファイルです。ただしジョブが
  ``launcher_spec[site]["k8s"]["python_path"]`` で上書きした場合は、そちらが優先されます。
  デフォルトは ``/usr/local/bin/python3`` です。
- ``image_pull_secrets``: この準備済みサイトで動的に起動されるすべてのジョブ Pod に付与される、
  既存の Kubernetes Secret 名の任意のリストです。これはデプロイ担当者が設定するものであり、
  ジョブ作成者が ``meta.json`` にレジストリ Secret 名を追加する必要はありません。
- ``job_pod_security_context``: 動的に起動されるジョブ Pod に渡されるセキュリティコンテキストです。

スタディ固有の Pod テンプレートはランチャーの引数ではありません。 ``local/study_runtime.yaml``
でスタディごとに設定してください（ ``studies.<study>.pod_template`` に、インラインまたは
``local/`` からの相対パスで指定します）。一致するスタディでは、ランチャーが所有するフィールドを
重ねた上でテンプレートが使用されます。テンプレート内のボリュームやジョブコンテナのマウントのうち
``workspace-job`` または ``startup-kit`` という名前のものは、ランチャーが生成するワークスペース
およびスタートアップのマウントで置き換えられます。

まず親のサーバーまたはクライアントのキットを準備します:

.. code-block:: shell

   nvflare deploy prepare ./site-1 --config k8s.yaml --output ./site-1-k8s

``deploy prepare`` の実行後、親 Pod のステージングまたは起動の前に、デプロイ担当者は生成された
``local/study_runtime.yaml`` を編集して、スタディごとのデータセット、環境変数、シークレット、
Pod テンプレートを、自動検出される 1 つのファイルの中で設定できます。ランチャーの引数は
必要ありません。v1 の ``local/study_data.yaml`` を今も持つレガシーキットの場合、生成される K8s
ランチャー設定では代わりに ``study_data_pvc_file_path`` が
``<workspace_mount_path>/local/study_data.yaml`` に設定され、既存のデータマウントが引き続き
動作します。この 2 つのファイルが同時に存在してはいけません。参照されるファイルは ``local/``
配下にステージングまたはコピーし、親プロセスが Pod 内のパスから読み取れるようにしてください。

その後、Helm で親 Pod を起動する前に、次の 2 つのステージング方法のいずれかを選択してください。

**方法 1: ``startup/`` と ``local/`` をワークスペース PVC にコピーする。**

準備済みキットの ``startup/`` および ``local/`` ディレクトリを、設定したワークスペース PVC の
ルートにコピーします。チャートはその PVC を ``workspace_mount_path`` にマウントします。
OpenShift 用のヘルパー
:github_nvflare_link:`examples/devops/openshift/scripts/k8s_deploy.sh <examples/devops/openshift/scripts/k8s_deploy.sh>`
は、この PVC コピー方式の完全なスクリプト例です。

コピーが完了したら、チャートをインストールまたはアップグレードします:

.. code-block:: shell

   helm upgrade --install site-1 ./site-1-k8s/helm_chart --namespace nvflare

**方法 2: ``local/`` を ConfigMap として、 ``startup/`` を Secret としてステージングする。**

``nvflare deploy k8s stage`` を使用して Kubernetes リソースを作成し、生成された Helm の values に
パッチを適用します。 ``nvflare deploy k8 stage`` もエイリアスとして受け付けられます。

.. code-block:: shell

   nvflare deploy k8s stage ./site-1-k8s --namespace nvflare
   helm upgrade --install site-1 ./site-1-k8s/helm_chart --namespace nvflare

この方法では、書き込み可能なランタイム状態のためにワークスペース PVC は引き続き
``workspace_mount_path`` にマウントされますが、親 Pod は ``local/`` を生成された ConfigMap から、
``startup/`` を生成された Secret から読み取ります。

``nvflare deploy prepare`` は、準備済みキットの内部通信設定にもパッチを適用し、動的に起動される
ジョブ Pod が ``parent_port`` 上の生成された親 Kubernetes Service に接続するようにします。
チャートの Service 名やポートをカスタマイズする場合は、その Service エンドポイントを準備済み
キットと一致させてください。

このコマンドは次のファイルを書き出します:

- 親のサーバーまたはクライアント Pod 用の ``helm_chart/``
- ``K8sJobLauncher`` を反映するようパッチされた ``local/resources.json.default``
- パッチされた ``local/comm_config.json``
- 存在しない場合は ``local/study_runtime.yaml`` のテンプレート（すでに ``study_data.yaml`` を
  持つレガシーキットではスキップされます）

************************
K8s ステージング
************************

``nvflare deploy k8s stage`` は、準備済みの K8s キットから Kubernetes リソースを作成し、それらを
マウントするように、生成された Helm チャートの values にパッチを適用します:

.. code-block:: none

   nvflare deploy k8s stage <prepared-kit-dir> [--namespace <namespace>]

このコマンドは、対象クラスタへの Kubernetes CLI アクセスを必要とします。デフォルトでは
``kubectl`` を使用します。 ``oc`` を使って OpenShift にステージングする場合は ``--kubectl oc``
または ``KUBECTL=oc`` を設定してください。このコマンドは次の処理を行います:

- 準備済みの ``local/`` 配下のすべてのファイルを含む ConfigMap を作成または更新します
- 準備済みの ``startup/`` 配下のすべてのファイルを含む Secret を作成または更新します
- 親 Pod が ConfigMap を ``workspace_mount_path/local`` に、Secret を
  ``workspace_mount_path/startup`` にマウントするよう ``helm_chart/values.yaml`` にパッチを
  適用します
- ``nvflare deploy k8s unstage`` で削除できるように、解決された名前空間とオブジェクト名を
  記録します

リソース名のデフォルトは ``nvflare-local-<site>`` と ``nvflare-startup-<site>`` です。
``--local-configmap`` と ``--startup-secret`` で上書きできます。名前空間のデフォルトは、準備済み
キットの ``K8sJobLauncher`` 設定に書き込まれた名前空間であり、利用できない場合は ``default``
になります。

このステージングコマンドが成功したら、出力された ``helm_command`` 、または準備済みチャートに
対する同等の ``helm upgrade --install`` コマンドを実行して、親のサーバーまたはクライアント Pod を
起動してください。このコマンドは、Helm のアンインストール後に使用する ``cleanup_command`` も
出力します。

生成された Helm チャートは、引き続き設定済みのワークスペース PVC をワークスペースのルートに
マウントします。ConfigMap と Secret は ``local/`` と ``startup/`` のサブディレクトリのみを
置き換えます。

****************************
K8s アンステージング
****************************

``nvflare deploy k8s stage`` が作成した ConfigMap と Secret は、生成された Helm リリースの一部では
ありません。リリースをアンインストールした後は ``nvflare deploy k8s unstage`` を実行し、
ステージングされた参加者アイデンティティの Secret がクラスタに残らないようにしてください:

.. code-block:: shell

   helm uninstall site-1 --namespace nvflare
   nvflare deploy k8s unstage ./site-1-k8s

``unstage`` は、直近の ``stage`` コマンドが記録した正確な名前空間とリソース名を読み取り、Secret と
ConfigMap を削除し、 ``helm_chart/values.yaml`` からそれらの参照を消去します。削除は正確な名前を
使用するため、いずれかのオブジェクトがすでに削除されていても安全です。

同じ準備済み出力を別の ``nvflare deploy prepare`` コマンドで置き換える前に ``unstage`` を実行して
ください。ステージング済みリソースがまだ記録されているチャートを上書きすると、そのクリーンアップ
対象が失われてしまうため、prepare は上書きを拒否します。

名前空間を記録していない古いバージョンの NVFlare によってステージングされたキットの場合は、
元の名前空間を明示的に渡してください:

.. code-block:: shell

   nvflare deploy k8s unstage ./site-1-k8s --namespace nvflare

名前が記録されていないレガシーなリソースや、部分的にステージングされたリソースをクリーンアップ
するために、 ``--local-configmap`` と ``--startup-secret`` を渡すこともできます。OpenShift では
``--kubectl oc`` または ``KUBECTL=oc`` を使用してください。 ``unstage`` は、Helm リリースを
アンインストールした後にのみ実行してください。インストール済みの親 Pod は、これらのボリュームに
依存しています。

********************
Slurm 設定
********************

Slurm バックエンドには、安定した共有ワークスペースとランチャーポリシーが必要です。最小限の
``slurm.yaml`` は次のとおりです:

.. code-block:: yaml

   runtime: slurm
   job_launcher:
     sandbox: apptainer
     image: /lustre/images/nvflare-prod.sif
     python_path: /usr/bin/python3
     parent_host: nvflare-site1.internal

共有ランタイムワークスペースに直接準備し、その後で親を起動します:

.. code-block:: shell

   nvflare deploy prepare ./site-1 --config slurm.yaml --output /lustre/proj123/nvflare/site-1
   /lustre/proj123/nvflare/site-1/startup/start_slurm.sh

この出力は稼働中のワークスペースそのものであり、親ノードと計算ノードの両方で同じ絶対パスから
参照できる必要があります。同じ出力先に再度準備を行うと、ワークスペース全体が置き換えられます。
クライアントキットでは、任意で ``startup/parent.slurm`` を生成できます。prepare は、それを
アロケーション内で実行する ``sbatch`` コマンドをそのまま出力します。完全なガイドについては
:ref:`slurm_job_launcher` を参照してください。

********************
ジョブイメージ
********************

Docker、Kubernetes、Slurm のジョブは、 ``meta.json`` でジョブイメージを選択できます。推奨される
形式は ``launcher_spec`` です:

.. code-block:: json

   {
     "launcher_spec": {
       "default": {
         "docker": {"image": "registry.example.com/nvflare-job:2.8"},
         "k8s": {"image": "registry.example.com/nvflare-job:2.8"},
         "slurm": {"image": "/shared/images/nvflare-job.sif"}
       },
       "site-1": {
         "docker": {"shm_size": "8g"}
       }
     },
     "resource_spec": {
       "site-1": {
         "num_of_gpus": 1
       }
     }
   }

``launcher_spec["default"][mode]`` は、そのモードにおけるすべてのサイトに適用されます。
``launcher_spec[site][mode]`` は、1 つのサイトについてデフォルトを上書きします。
``num_of_gpus`` などのリソース要求は ``resource_spec`` に記述してください。

ジョブが提供するイメージは実行可能なコンテンツであり、サイトの通常の BYOC 認可が必要です。
Slurm は、有効なイメージをジョブ、次にスタディの ``container.image`` 、次にサイトの
``job_launcher.image`` の順に解決します。Docker/Kubernetes で使用されるレジストリのイメージ名とは
異なり、Slurm のイメージは絶対パスで、サイトから参照できる既存のファイルでなければなりません。
:ref:`slurm_job_launcher` を参照してください。

********************
終了ステータス
********************

検証エラーはコード ``4`` で終了し、構造化されたエラーを報告します。よくある原因は次のとおりです:

- ランタイム設定が存在しない、または不正である
- サポートされていない管理者用スタートアップキット
- ``startup/`` または ``local/`` ディレクトリが存在しない
- ``resources.json.default`` が不正である
- 予約済みの Docker ランチャー kwargs が指定されている
- ``--output`` が入力キット自体、または入力キットの内部を指している
- Slurm の ``--output`` パスがランタイムワークスペースとして有効でない
- Slurm の親 CLI が存在しないか実行可能でない、sandbox/image が不正である、または Slurm の
  ``connection_security`` やサーバーの ``parent`` 設定がサポートされていない
