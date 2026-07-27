.. _study_command:

############################
NVIDIA FLARE Study CLI
############################

``nvflare study`` コマンド群は、稼働中の NVFlare サーバー上でマルチスタディのライフサイクル操作
(スタディの登録と削除、サイトの登録と削除、スタディのユーザーメンバーシップの管理) を行います。

これらのコマンドは、サーバーが ``api_version: 4`` でプロビジョニングされ、マルチスタディのサポートが
有効になっている場合にのみ意味を持ちます。プロビジョニングの設定については :ref:`multi_study_guide`
を参照してください。

.. code-block:: none

   nvflare study -h

   usage: nvflare study [-h] {register,show,list,remove,add-site,remove-site,add-user,remove-user} ...

   study subcommands:
     register       register a new study with initial site enrollment
     show           show the current definition of a study
     list           list all studies visible to the caller
     remove         remove a study and all its configuration
     add-site       enroll additional sites in a study
     remove-site    remove sites from a study
     add-user       add a user to a study's admin list
     remove-user    remove a user from a study's admin list

**********************************
スタートアップキットの解決
**********************************

すべての ``nvflare study`` コマンドは、admin スタートアップキットを介してサーバーに接続します。
解決の方法は、サーバーに接続する他のすべての ``nvflare`` コマンド ( ``job`` 、``system`` など) と
同一です。

1. 任意の ``--kit-id <id>`` : 登録済みのスタートアップキット ID を使用して、このコマンドに限り
   アクティブなスタートアップキットを上書きします。
2. 任意の ``--startup-kit <path>`` : admin スタートアップキットのディレクトリを明示的に指定して、
   このコマンドに限りアクティブなスタートアップキットを上書きします。
3. ``NVFLARE_STARTUP_KIT_DIR`` 環境変数。
4. ``~/.nvflare/config.conf`` の ``startup_kits.active`` 。
5. いずれのソースからも有効な admin スタートアップキットが解決できない場合、コマンドは接続前に
   失敗します。

コマンドラインのセレクタは必須ではありません。指定した場合は、現在のコマンドに限りアクティブな
スタートアップキットよりも優先され、``~/.nvflare/config.conf`` の ``startup_kits.active`` は
変更されません。

ユーザーは :ref:`config_command` を使って、スタートアップキットを一度登録してアクティブ化できます。

.. code-block:: shell

   nvflare config add project_admin /path/to/admin@nvidia.com
   nvflare config use project_admin
   nvflare study list --kit-id project_admin

いずれのソースからも解決できない場合、コマンドはエラーコード 4 と ``"error_code": "STARTUP_KIT_MISSING"``
を返して終了します。

**********************************
ロールに基づく入力要件
**********************************

スタディへのサイト登録は、2 層のロールチェックに従います。まず CLI 側で (スタートアップキット内の
呼び出し元の証明書に基づいて) チェックされ、次にサーバー側で (認証された接続プロパティに基づいて)
最終的にチェックされます。

- **project_admin** — ``--site-org <org>:<site>`` のペアを指定してサイト登録を管理します。
  ``--sites`` の使用は拒否されます。
- **org_admin** — ``--sites`` を指定して、自分の組織のサイトのみを管理します。
  ``--site-org`` の使用は拒否されます。
- 同じコマンドで ``--sites`` と ``--site-org`` の両方を指定することは常に拒否されます。

*************************
スタディの登録
*************************

新しいスタディを登録し、その初期のサイト群を登録します。

.. code-block:: shell

   # project_admin: register with per-org site groupings
   nvflare study register cancer-research \
       --site-org org_a:hospital-1 \
       --site-org org_a:hospital-2 \
       --site-org org_b:clinic-1

   # org_admin: register and enroll own org's sites
   nvflare study register cancer-research --sites hospital-1 hospital-2

オプション:

- ``<name>`` (必須の位置引数): 作成するスタディの名前。
- ``--site-org <org>:<site>`` (project_admin): 1 つ以上の ``org:site`` ペア。複数指定する場合は
  フラグを繰り返します。
- ``--sites <site> [<site> ...]`` (org_admin): 呼び出し元の組織に属する 1 つ以上のサイト。
  ``--sites hospital-1,hospital-2`` のようなカンマ区切りの入力も受け付けられます。

*************************
スタディの表示
*************************

登録済みのサイトや管理ユーザーを含む、スタディの現在の定義を表示します。

.. code-block:: shell

   nvflare study show cancer-research

サイトと組織のマッピング、およびそのスタディの管理者一覧を返します。

*************************
スタディの一覧表示
*************************

呼び出し元がアクセスできるすべてのスタディを一覧表示します。

.. code-block:: shell

   nvflare study list
   nvflare study list --format json

- ``project_admin`` はすべてのスタディを参照できます。
- ``org_admin`` は、自分の組織がサイトを登録しているスタディを参照できます。
- ``lead`` および ``member`` のユーザーは、自分が明示的にマッピングされているスタディを参照できます。

JSON モードでは、CLI が選択したスタートアップキット、サーバーが認証したアイデンティティ、および
スタディごとのサブミット事前チェックのフィールドが含まれます。

.. code-block:: json

   {
     "startup_kit": {
       "source": "active",
       "id": "lead@nvidia.com",
       "path": "/path/to/lead@nvidia.com"
     },
     "identity": {
       "name": "lead@nvidia.com",
       "org": "nvidia",
       "role": "lead"
     },
     "studies": ["cancer-research"],
     "study_details": [
       {
         "name": "cancer-research",
         "role": "lead",
         "capabilities": {"submit_job": true},
         "can_submit_job": true
       }
     ]
   }

``can_submit_job`` は、``submit_job`` 権限に対するアクティブなサーバー認可ポリシーに基づいて評価されます。
あるアイデンティティがスタディを参照できても、ジョブのサブミットは拒否される場合があります。
その場合、該当する行には認可からの拒否理由 ``reason`` が含まれます。これはサブミットの事前チェックに
すぎず、後で実際にサブミットした際に、サーバー側の他の検証やポリシー上の理由で失敗する可能性は
依然として残ります。

*************************
スタディの削除
*************************

スタディとそのすべての構成を削除します。そのスタディの配下で実行中のジョブがある場合、この操作は
拒否されます。

.. code-block:: shell

   nvflare study remove cancer-research

*******************************
スタディへのサイトの追加
*******************************

既存のスタディに追加のサイトを登録します。

.. code-block:: shell

   # project_admin
   nvflare study add-site cancer-research \
       --site-org org_b:clinic-2

   # org_admin
   nvflare study add-site cancer-research --sites clinic-2

``--site-org`` / ``--sites`` のオプションは ``register`` と同じです。

*********************************
スタディからのサイトの削除
*********************************

スタディからサイトを削除します。スタディ自体は削除されません。

.. code-block:: shell

   # project_admin
   nvflare study remove-site cancer-research \
       --site-org org_b:clinic-2

   # org_admin
   nvflare study remove-site cancer-research --sites clinic-2

***********************************
スタディへのユーザーの追加
***********************************

既存の管理ユーザーをスタディの管理者一覧に追加します。

.. code-block:: shell

   nvflare study add-user cancer-research trainer@org_a.com

- ``<study>`` (必須の位置引数): 更新対象のスタディ。
- ``<user>`` (必須の位置引数): 追加する管理ユーザー。

***********************************
スタディからのユーザーの削除
***********************************

スタディの管理者一覧からユーザーを削除します。そのユーザーがデプロイメントから削除されるわけでは
ありません。

.. code-block:: shell

   nvflare study remove-user cancer-research trainer@org_a.com

*************************
出力フォーマット
*************************

すべての ``nvflare study`` コマンドは、グローバルな ``--format {txt,json}`` フラグに従います。
``--format json`` (自動化向けのデフォルト) では、すべてのレスポンスが JSON エンベロープになります。

.. code-block:: json

   {"status": "ok", "data": { ... }}

エラーは次の形式で返されます。

.. code-block:: json

   {"status": "error", "error_code": "STUDY_NOT_FOUND", "message": "...", "hint": "...", "exit_code": 1}

主なエラーコード:

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - エラーコード
     - 意味
   * - ``STARTUP_KIT_MISSING``
     - ``--kit-id`` 、``--startup-kit`` 、``NVFLARE_STARTUP_KIT_DIR`` 、アクティブな設定エントリのいずれからもスタートアップキットを解決できませんでした (終了コード 4)。
   * - ``STARTUP_KIT_NOT_CONFIGURED``
     - アクティブなスタートアップキットが構成されておらず、コマンドごとのセレクタや環境変数による上書きも指定されていません (終了コード 4)。
   * - ``CONNECTION_FAILED``
     - サーバーに接続できない、または認証できません (終了コード 2)。
   * - ``INVALID_ARGS``
     - 引数の形式がロールの契約に違反しています (終了コード 4)。
   * - ``STUDY_NOT_FOUND``
     - 指定されたスタディが存在しない、または呼び出し元から参照できません (終了コード 1)。
   * - ``STUDY_ALREADY_EXISTS``
     - そのスタディ名はすでに登録されています (終了コード 1)。
   * - ``INVALID_SITE``
     - サイトが登録されていない、または呼び出し元の組織に属していません (終了コード 4)。
   * - ``INVALID_STUDY_NAME``
     - スタディ名が命名規則に適合しません (終了コード 4)。
   * - ``STUDY_HAS_JOBS``
     - 関連するジョブがあるスタディは削除できません (終了コード 1)。
   * - ``USER_ALREADY_IN_STUDY``
     - ユーザーがすでにこのスタディのメンバーシップ一覧に含まれているため、``add-user`` が拒否されました (終了コード 1)。
   * - ``USER_NOT_IN_STUDY``
     - ユーザーがこのスタディのメンバーシップ一覧に含まれていないため、``remove-user`` が拒否されました (終了コード 1)。
   * - ``NOT_AUTHORIZED``
     - 呼び出し元の証明書のロールでは、この操作を行う権限が不足しています (終了コード 1)。
   * - ``LOCK_TIMEOUT``
     - レジストリがビジー状態です。別の変更処理が進行中です (終了コード 3)。

*************************
スキーマ出力
*************************

どのサブコマンドでも ``--schema`` を指定すると、その引数スキーマを JSON として出力できます。

.. code-block:: shell

   nvflare study register --schema
