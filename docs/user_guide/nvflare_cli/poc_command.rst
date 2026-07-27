.. _poc_command:

*****************************************
概念実証 (POC) コマンド
*****************************************

``nvflare poc`` コマンドは、単一マシン上でのローカルな概念実証 (proof-of-concept) デプロイメントを
管理します。サーバー、クライアント、管理者のスタートアップキットはそれぞれ別のプロセスとして表現され、
これにより POC モードは分散デプロイメントの前にジョブのワークフローを検証する便利な手段となります。

***********************
コマンドの使い方
***********************

POC コマンドは、 ``config`` 、 ``prepare`` 、 ``add-user`` 、 ``add-site`` 、 ``start`` 、 ``stop`` 、
``clean`` の各サブコマンドを提供します。

.. code-block:: none

   nvflare poc -h

   usage: nvflare poc [-h] {config,prepare,add-user,add-site,start,stop,clean} ...

*************************
一般的なワークフロー
*************************

1. 必要に応じて ``nvflare poc config --pw <poc_workspace>`` を実行し、ローカルワークスペースのパスを
   選択します。
2. ``nvflare poc prepare`` を実行して、ローカルワークスペースとスタートアップキットを作成します。
3. 必要に応じて ``nvflare poc add-user`` または ``nvflare poc add-site`` を実行し、ローカル参加者の
   スタートアップキットを追加します。
4. ``nvflare poc start`` を実行して、サーバーとクライアントを起動します。
5. ``nvflare job submit -j <path/to/job>`` でジョブを直接投入します。
6. 管理コンソールが必要な場合にのみ、明示的に起動します。
7. ``nvflare poc stop`` を実行してシステムを停止します。
8. システムの停止後に ``nvflare poc clean`` を実行します。

*****************************
ワークスペースの設定
*****************************

ローカル POC ワークスペースのパスを表示または設定するには、 ``nvflare poc config`` を使用します。

.. code-block:: none

   nvflare poc config [-h] [-pw [POC_WORKSPACE_DIR]] [--schema]

オプション:

- ``-pw, --pw, --poc_workspace_dir, --poc-workspace-dir``: POC ワークスペースの場所です。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare poc config
   nvflare poc config --pw /tmp/nvflare/poc

*********************************
ワークスペースの準備
*********************************

ローカルプロジェクトをプロビジョニングするには、 ``nvflare poc prepare`` を使用します。

.. code-block:: none

   nvflare poc prepare [-h] [-n [NUMBER_OF_CLIENTS]] [-c [CLIENTS ...]]
                       [-he] [-i [PROJECT_INPUT]] [-d [DOCKER_IMAGE]]
                       [-debug] [--force] [--schema]

オプション:

- ``-n, --number_of_clients``: サイトまたはクライアントの数です。既定値: ``2`` 。
- ``-c, --clients``: 空白区切りのクライアント名です。指定した場合、 ``number_of_clients`` は
  無視されます。
- ``-he, --he``: 生成されるローカルプロジェクトで準同型暗号を有効にします。
- ``-i, --project_input``: ``project.yaml`` ファイルへのパスです。指定した場合、クライアント数、
  クライアント名、Docker イメージのオプションは無視されます。
- ``-d, --docker_image``: POC を Docker ランタイムモードでプロビジョニングし、 ``nvflare deploy prepare``
  と同じ Docker 準備処理を用いてサーバー／クライアントのスタートアップキットを準備します。値は SP/CP の
  Docker イメージです。値を指定せずに与えた場合は、既定のイメージが使用されます。Docker モードのサイトに
  投入されるジョブは、ジョブの ``launcher_spec`` で SJ/CJ の Docker イメージを指定する必要があります。
- ``-debug, --debug``: デバッグモードです。
- ``--force``: 確認プロンプトを表示せずに既存のワークスペースを上書きします。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

動作に関する注意:

- ワークスペースが既に存在し、標準入力が対話的でない場合は、 ``--force`` が必須です。
- ``nvflare poc prepare`` は ``~/.nvflare/config.conf`` を POC ワークスペースで更新し、生成された
  管理者／ユーザーのスタートアップキットを登録して、既定のプロジェクト管理者キットを有効化します。
  サイトのスタートアップキットは POC ワークスペース内に留まり、CLI のアイデンティティとしては登録されません。
- Docker POC モードでは、サーバーおよびクライアントの親プロセスを ``start_docker.sh`` で起動し、
  ジョブコンテナの起動には ``DockerJobLauncher`` を使用します。既定のスタディに属するジョブの場合、
  POC ワークスペースの ``data`` ディレクトリが Docker ジョブコンテナ内の ``/data/default/poc`` に
  マウントされるため、サンプルはそのパスからローカルデータセットをダウンロードしたり読み込んだりできます。
- 成功すると、コマンドはワークスペースのパスと検出されたクライアントの一覧を含む JSON の結果を出力します。

例:

.. code-block:: shell

   nvflare poc prepare -n 2

***********************
参加者の追加
***********************

準備済みのローカル POC ワークスペースにユーザーやサイトを追加するには、 ``nvflare poc add-user`` または
``nvflare poc add-site`` を使用します。

.. code-block:: none

   nvflare poc add-user [-h] [--org ORG] [--force] [--schema]
                        {org_admin,lead,member} email

   nvflare poc add-site [-h] [--org ORG] [--force] [--schema] name

動作に関する注意:

- ``poc add-user`` と ``poc add-site`` は、ローカル POC ワークスペースに対する操作です。これらは
  ``poc prepare`` が作成したローカル POC プロジェクトのメタデータとローカル POC CA を使用し、
  現在有効なスタートアップキットによる制約を受けません。
- ``poc add-user`` は、永続化された POC の ``project.yml`` にセカンダリの管理者参加者を追加し、
  既存の POC CA を用いてその新しいユーザーのみを動的にプロビジョニングして、生成されたユーザーの
  スタートアップキットを共有スタートアップキットレジストリに登録します。別の ``project_admin`` を
  追加することはできません。POC のプロジェクト管理者は ``poc prepare`` によって作成されます。
- ``poc add-site`` は、永続化された POC の ``project.yml`` にクライアント参加者を追加し、既存の POC CA を
  用いてその新しいサイトのみを動的にプロビジョニングします。生成されたサイトキットは現在の POC 出力
  ディレクトリ（通常は ``prod_00`` ）に配置され、CLI のアイデンティティとなるのは管理者／ユーザーの
  キットのみであるため、 ``~/.nvflare/config.conf`` には登録されません。
- POC の追加処理は既存のプロビジョニング状態およびルート CA を使用し、既存の参加者のスタートアップキットを
  再生成することはありません。
- ``--force`` は、ローカル POC プロジェクトのメタデータ内にある既存の参加者エントリを置き換える場合にのみ
  使用してください。

例:

.. code-block:: shell

   nvflare poc add-user lead bob@nvidia.com --org nvidia
   nvflare config use bob@nvidia.com

   nvflare poc add-site site-3 --org nvidia
   nvflare config list
   nvflare poc start -p site-3

***********************
サービスの起動
***********************

準備済みの POC ワークスペースでサービスを起動するには、 ``nvflare poc start`` を使用します。

.. code-block:: none

   nvflare poc start [-h] [-p [SERVICE]] [-ex [EXCLUDE]] [-gpu [GPU ...]]
                     [--study STUDY] [--no-wait] [--timeout SECONDS]
                     [-debug] [--schema]

オプション:

- ``-p, --service``: 起動する参加者です。既定ではサーバーとクライアントを起動し、明示的に要求しない限り
  管理コンソールは除外されます。
- ``-ex, --exclude``: 起動対象から除外する参加者です。
- ``-gpu, --gpu``: ``CUDA_VISIBLE_DEVICES`` として使用する GPU デバイス ID です。
- ``--study``: 管理コンソールの起動時にのみ使用するスタディです。サーバーおよびクライアントのサービスでは
  無視されます。
- ``--no-wait``: 管理サーバーおよび選択されたクライアントが利用可能になるのを待たずに、プロセスを起動した
  時点で戻ります。
- ``--timeout``: 管理サーバーおよび選択されたクライアントが利用可能になるまで待機する秒数です。既定では
  組み込みの POC レディネスタイムアウトが使用されます。
- ``-debug, --debug``: デバッグモードです。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

動作の変更点:

- 管理コンソールの参加者は **既定では起動されません** 。
- サービスを明示せずに ``nvflare poc start`` を実行した場合、起動されるのはサーバーとクライアントのみです。
- 既定では、管理サーバーが接続を受け付け、選択されたクライアントが登録されるまで待機してから
  ``status: running`` を返します。
- このレディネス待機は ``--timeout`` で制御できます。
- ``--no-wait`` を指定すると、コマンドは直ちに ``status: starting`` を返します。
- コマンドは ``status`` 、 ``server_url`` 、 ``server_address`` 、 ``admin_address`` 、 ``clients`` 、
  ``ready_timeout`` 、 ``port_conflict`` 、 ``port_preflight`` 、 ``warnings`` を含む JSON を返し、
  レディネスが確認されたか明示的にスキップされた場合は ``ready`` も返します。
- 以降の自動化では、機械可読なエンドポイントアドレスとして ``data.server_address`` と
  ``data.admin_address`` を使用してください。 ``data.server_url`` は既存のクライアントとの互換性のために
  維持されています。
- ``data.port_conflict`` は、ローカルのポートチェックに基づくベストエフォートの起動前警告です。true の
  場合は、別の稼働中の POC システムに接続してしまうことを避けるため、ジョブを投入する前に
  ``data.port_preflight.conflicts`` と ``data.warnings`` を確認してください。

例:

.. code-block:: shell

   nvflare poc start
   nvflare poc start --timeout 60
   nvflare poc start -p server
   nvflare poc start -p admin@nvidia.com
   nvflare poc start -p admin@nvidia.com --study cancer_research
   nvflare poc start -ex admin@nvidia.com

管理コンソールを起動するには、 ``-p`` で明示的に指定してください。

スタディに関する注意:

- ``--study`` は管理コンソールを起動するときにのみ使用してください。
- 名前付きスタディを使用するには、POC ワークスペースが ``api_version: 4`` と ``studies:`` を含む
  カスタムの ``project.yml`` から準備されている必要があります。ワークスペースが既定の生成プロジェクトから
  準備された場合、有効なスタディは ``default`` のみです。

***********************
サービスの停止
***********************

稼働中の POC サービスを停止するには、 ``nvflare poc stop`` を使用します。

.. code-block:: none

   nvflare poc stop [-h] [-p [SERVICE]] [-ex [EXCLUDE]] [--no-wait]
                    [-debug] [--schema]

オプション:

- ``-p, --service``: 停止する参加者です。既定では、管理コンソールを含むすべての稼働中サービスを停止します。
- ``-ex, --exclude``: 停止処理から除外する参加者です。
- ``--no-wait``: シャットダウンを要求した後、完了を待たずに戻ります。
- ``-debug, --debug``: デバッグモードです。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare poc stop
   nvflare poc stop -p server
   nvflare poc stop -p site-1
   nvflare poc stop --no-wait

サーバーを停止する経路では、協調的なシステムシャットダウンのロジックが使用されます。一部のサービスのみを
停止する場合は、ローカルの停止スクリプトのフローが使用されます。既定では、サーバーの経路はシャットダウンの
完了を待ってから ``status: stopped`` を返します。 ``--no-wait`` を指定した場合は、直ちに
``status: shutdown_initiated`` を返します。

**********************************
ワークスペースのクリーンアップ
**********************************

POC ワークスペースを削除するには、 ``nvflare poc clean`` を使用します。

.. code-block:: none

   nvflare poc clean [-h] [-debug] [--force] [--schema]

オプション:

- ``-debug, --debug``: デバッグモードです。
- ``--force``: ワークスペースを削除する前に、稼働中のローカル POC システムを停止します。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

動作に関する注意:

- ワークスペースは、有効な POC ディレクトリである場合にのみ削除されます。
- POC システムがまだ稼働している場合、 ``nvflare poc clean`` は先に停止するよう促すヒントとともに
  失敗します。 ``nvflare poc clean --force`` を使用すると、ローカル POC システムの停止とワークスペースの
  削除を 1 つのコマンドで実行できます。

*******************************
ワークスペースの構成
*******************************

既定の POC ワークスペースは ``/tmp/nvflare/poc`` です。

ワークスペースは次の方法でも制御できます。

- ``NVFLARE_POC_WORKSPACE``
- ``nvflare poc config --pw <poc_workspace>`` を介した ``~/.nvflare/config.conf``

ローカル POC ワークスペースを表示または設定するには、 ``nvflare poc config`` を使用します。

.. code-block:: shell

   nvflare poc config
   nvflare poc config --pw /tmp/nvflare/poc

以前のルートコマンドである ``nvflare config -pw <poc_workspace>`` も互換性のために引き続き受け付けられますが、
非推奨であり、 ``nvflare poc config --pw`` を案内する警告を出力します。

``nvflare poc prepare`` は、POC ワークスペースをローカルの NVFlare 設定に書き込み、生成された管理者／
ユーザーのスタートアップキットを共有スタートアップキットレジストリに自動的に登録します。生成された POC の
アイデンティティが、POC ワークスペース外の既存のスタートアップキット登録と衝突する場合、その登録先のパスが
まだ存在していれば、prepare は既存の登録を保持します。既存の登録が既に存在しないパスを指している場合は、
prepare はそれを古いローカル POC の状態とみなして置き換えます。サイトのスタートアップキットは、ローカルな
サービス管理のために POC ワークスペース内に残ります。

既定のプロジェクト管理者スタートアップキットが有効になるため、 ``nvflare job list`` や
``nvflare system status`` のようなサーバー接続を伴うコマンドを、追加のスタートアップキットフラグなしで
実行できます。

JSON モードでは、 ``nvflare poc prepare`` は有効なキットの遷移を報告します。
``data.startup_kit.prior_active`` は prepare の前に有効だったスタートアップキット ID、
``data.startup_kit.active`` は prepare の後に有効になった ID、 ``data.startup_kit.changed`` は prepare が
既定のアイデンティティを変更したかどうかを示します。エージェントはこの情報を利用して、POC のワークフローの
後にユーザーの以前のアイデンティティを復元できます。

``nvflare poc prepare`` は、 ``data.port_preflight`` の下にベストエフォートのローカルサーバーポート
プリフライトも報告します。プロジェクト設定を読み取れる場合、このコマンドは生成された POC サーバーポートを
ループバックアドレス上でチェックし、利用できないポートを ``data.port_preflight.conflicts`` に列挙します。
これらの衝突は後続の ``nvflare poc start`` ステップに対する警告であり、 ``poc prepare`` を失敗させることは
ありません。このプリフライトはワイルドカードインターフェースにバインドしないため、
``data.port_preflight.note`` はこのチェックがベストエフォートであることを説明します。

JSON モードでは、 ``nvflare poc start`` はバインドされた POC のエンドポイントを ``data.server_address``
および ``data.admin_address`` の下に報告します。 ``--no-wait`` が使用されない限り、コマンドは既定で
レディネスを待機します。また、起動前にベストエフォートのローカルサーバーポートプリフライトを再度実行し、
利用できない設定済みポートを ``data.port_preflight.conflicts`` の下に報告するとともに
``data.port_conflict`` を ``true`` に設定します。このプリフライトはループバックのみを対象とした早期警告を
意図したものであり、他のローカルバインドアドレスが競合している場合には起動が失敗することもあります。

生成された POC スタートアップキットの登録内容を確認したり、POC で生成されたユーザースタートアップキットを
切り替えたりするには、 :ref:`config_command` を使用してください。

*************************
JSON 出力とヘルプ
*************************

機械可読な出力を得るには、サブコマンドの後に ``--format json`` を追加します。

.. code-block:: shell

   nvflare poc prepare -n 2 --format json
   nvflare poc start --format json

機械可読なコマンド探索には ``--schema`` を使用します。 ``--schema`` は常に JSON を返すため、
``--format json`` を併用する必要はありません。

.. code-block:: shell

   nvflare poc prepare --schema
   nvflare poc start --schema

人間向けの引数エラーでは、まずヘルプが表示され、その後に具体的なエラーが表示されます。JSON モードでは
JSON のエラーエンベロープのみが出力されます。
