.. _identity_security_page:

##############################
アイデンティティセキュリティ
##############################
この領域は、次の 2 つの信頼に関する課題を扱います。

    - Authentication (認証): 通信を行う当事者が互いのアイデンティティについて十分な確信を持てるようにします。つまり、全員が名乗ったとおりの本人であることを保証します。
    - Authorization (認可): ユーザーが認可された操作のみを実行できるようにします。

認証
==============
NVFLARE の認証モデルは、公開鍵基盤 (PKI) 技術に基づいています。

    - FL プロジェクトにおいて、プロジェクト管理者はプロビジョニングツールを使用して、自己署名ルート証明書を持つルート CA を作成します。このルート CA は、通信を行う当事者が必要とする他のすべての証明書を発行するために使用されます。
    - スタディに関与するアイデンティティ (サーバー、クライアント、ユーザー) は、プロビジョニングツールでプロビジョニングされます。各アイデンティティは一意のコモンネームで定義されます。プロビジョニングツールは、アイデンティティごとに、相互 TLS 認証のためのセキュリティクレデンシャルを含むパスワード保護されたスタートアップキットを個別に生成します。
        - ルート CA の証明書
        - そのアイデンティティの証明書
        - そのアイデンティティの秘密鍵
    - スタートアップキットは、対象となるアイデンティティに配布されます。
        - FL サーバーのキットはプロジェクト管理者に送られます
        - 各 FL クライアントのキットは、そのサイトを担当する組織管理者に送られます
        - FLARE コンソール (旧称 Admin Client) のキットはユーザーに送られます
    - スタートアップキットの完全性を保証するため、キット内の各ファイルはルート CA によって署名されています。
    - 各スタートアップキットには "start.sh" ファイルも含まれており、これを使用して NVFLARE アプリケーションを適切に起動できます。
    - 起動されると、クライアントはスタートアップキット内の PKI クレデンシャルを使用して、サーバーとの相互認証済み TLS 接続の確立を試みます。これは、クライアントとサーバーの双方が正しいスタートアップキットを持っている場合にのみ可能です。
    - 同様に、ユーザーが Admin Client アプリで NVFLARE システムを操作しようとすると、管理クライアントはスタートアップキット内の PKI クレデンシャルを使用して、サーバーとの相互認証済み TLS 接続の確立を試みます。これは、管理クライアントとサーバーの双方が正しいスタートアップキットを持っている場合にのみ可能です。また、管理ユーザーは割り当てられたユーザー名を正しく入力する必要があります。

システムのセキュリティは、スタートアップキット内の PKI クレデンシャルによってもたらされます。ご覧のとおり、このメカニズムはスタートアップキットの配布において手作業と人的なやり取りを伴うため、システムのアイデンティティセキュリティは関係者の信頼に依存します。セキュリティリスクを最小化するため、関係者は以下のベストプラクティスのガイドラインに従うことを推奨します。

    - スタディのプロビジョニングプロセスを担当するプロジェクト管理者は、スタディの設定ファイルを保護し、作成したスタートアップキットを安全に保管してください。
    - スタートアップキットを配布する際、プロジェクト管理者は信頼できる通信手段を用い、スタートアップキットのパスワードを同じ通信手段で送らないでください。キットとパスワードは別々の通信手段で送ることが望ましいです。
    - 組織管理者およびユーザーは、自身のスタートアップキットを保護し、意図された目的にのみ使用してください。

.. note::

    プロビジョニングツールは、PKI クレデンシャルを生成する際に可能な限り強力な暗号スイートを使用しようとします。すべての証明書は X.509 標準に準拠しています。すべての秘密鍵は 2048 ビットのサイズで生成されます。バックエンドは 2020 年 3 月 31 日にリリースされた openssl 1.1.1f で、既知の CVE はありません。すべての証明書は 360 日以内に有効期限を迎えます。

.. note::

    :ref:`NVFlare Dashboard <nvflare_dashboard_ui>` は、ユーザーおよびサイトの登録をサポートする Web サイトです。ユーザーは、この Web サイトからスタートアップキット (およびその他の成果物) をダウンロードできます。


.. _federated_authorization:

認可: フェデレーテッド認可
======================================
連合学習は、異なる組織が所有する計算リソース上で実施されます。当然ながら、これらの組織は自身の計算リソースが誤用または悪用されることを
懸念します。NVFLARE の docker が参加組織から信頼されていたとしても、研究者は依然として独自のカスタムコードをスタディの一部として
持ち込むことができ (BYOC)、これは多くの組織にとって大きな懸念となり得ます。さらに、組織は自組織の研究者が実施するスタディに対して
IP (知的財産) 要件を持つこともあります。

NVFLARE には、これらのセキュリティ上の懸念と IP 要件に対処するのに役立つ認可システムが備わっています。このシステムにより、組織は自身の計算リソースや FL ジョブへのアクセスを制御する厳格なポリシーを定義できます。

組織が実行できることの例をいくつか示します。

    - BYOC をその組織自身の研究者のみに制限する
    - 自組織の研究者から、指定した他の組織から、あるいは指定した信頼できる他の研究者からのジョブのみを許可する
    - 自サイト上でのリモートシェルコマンドを完全に無効にする
    - "ls" シェルコマンドは許可し、それ以外のすべてのリモートシェルコマンドを無効にする

集中型認可とフェデレーテッド認可
---------------------------------------
バージョン 2.2.1 より前の NVFLARE では、認可ポリシーは FL サーバーによって集中的に適用されていました。真のフェデレーテッド環境では、各組織が他者 (別の組織が所有する FL サーバーなど) に依存するのではなく、自身の認可ポリシーを定義し適用できるべきです。

NVFLARE は現在、各組織が自身の認可ポリシーを定義し適用するフェデレーテッド認可を採用しています。

    - 各組織は、自身の authorization.json (ワークスペースの local フォルダー内) にポリシーを定義します
    - このローカルに定義されたポリシーは、その組織が所有する FL クライアントによって読み込まれます
    - ポリシーはこれらの FL クライアントによって適用されます

この分散型の認可には、追加の利点があります。各組織が自身の認可を管理するため、新しい組織やクライアントが追加されても、他の参加者 (FL サーバーやクライアント) のポリシーを更新する必要がありません。

認可のためのフェデレーテッドサイトポリシーの動作する例については、 :github_nvflare_link:`Federated Policies (Github) <examples/advanced/federated-policies/README.rst>` を参照してください。

簡素化された認可ポリシー設定
---------------------------------------------
各組織が自身のポリシーを定義するため、すべての組織とユーザーを集中的に定義する必要はありません。ある組織のポリシー設定は、単なるロール/権限のパーミッションのマトリクスです。パーミッションマトリクスにおける各ロール/権限の組み合わせは、「このロールのどのようなユーザーがこの権限を持てるか」という問いに答えます。

この問いに答えるため、ロール/権限の組み合わせは 1 つ以上の条件を定義し、ユーザーはその権限を得るためにこれらの条件のいずれかを満たす必要があります。この条件の集合はコントロールと呼ばれます。

ロール
^^^^^^^^
ユーザーはロールに分類されます。NVFLARE は 4 つのロールを定義しています。

    - Project Admin - このロールは FL プロジェクト全体に責任を持ちます
    - Org Admin - このロールは、その組織内のすべてのサイトの管理に責任を持ちます。各組織には 1 人の Org Admin が必要です
    - Lead (研究者) - このロールは FL スタディを実施します
    - Member (研究者) - このロールは FL スタディを観察できますが、ジョブを送信することはできません

権限
^^^^^^
NVFLARE は、より柔軟にするために、より詳細な権限定義をサポートしています。

    - サーバー側の各管理コマンドが 1 つの権限です。これにより、組織は各コマンドを明示的に制御できます
    - 管理コマンドはカテゴリにグループ化されています。たとえば、abort_job、delete_job、start_app などのコマンドは manage_job カテゴリに属し、すべてのシェルコマンドは shell_commands カテゴリに入れられます。各カテゴリも 1 つの権限です。
    - BYOC は権限として定義されるようになったため、一部のユーザーには BYOC を伴うジョブの送信を許可し、他のユーザーには許可しないといったことが可能です。

この権限システムにより、コマンドカテゴリのみを使用したシンプルなポリシーを簡単に記述できます。また、個々のコマンドを制御するポリシーを記述することも可能です。カテゴリとコマンドの両方が使用されている場合は、コマンドベースの制御がカテゴリベースの制御よりも優先されます。

コマンドカテゴリについては :ref:`command_categories` を参照してください。

コントロールと条件
^^^^^^^^^^^^^^^^^^^^^^^
*コントロール* とは、パーミッションマトリクスで指定される 1 つ以上の条件の集合です。条件は、対象となるユーザー、サイト、ジョブの送信者の間の関係を指定します。サポートされている関係は次のとおりです。

    - ユーザーがサイトの組織に属している (user org = site org)
    - ユーザーがジョブの送信者である (user name = submitter name)
    - ユーザーとジョブの送信者が同じ組織に属している (user org = submitter org)
    - ユーザーが指定された人物である (user name = specified name)
    - ユーザーが指定された組織に属している (user org = specified org)

この関係は常に対象となるユーザーを基準とした相対的なものであることに注意してください。つまり、ユーザーの名前または組織が、サイトまたはジョブの送信者と適切な関係にあるかどうかを確認します。

条件はポリシー定義ファイル (authorization.json) の中で表現する必要があるため、簡潔で一貫した記法が必要です。これらの条件の記法は次のとおりです。

.. csv-table::
    :header: 記法,条件,例
    :widths: 15, 20, 15

    o:site,ユーザーがサイトの組織に属している
    n:submitter,ユーザーがジョブの送信者である
    o:submitter,ユーザーとジョブの送信者が同じ組織に属している
    n:<person_name>,ユーザーが指定された人物である,n:john@nvidia.com
    o:<org_name>,ユーザーが指定された組織に属している,o:nvidia

"site" と "submitter" という語は予約語です。

さらに、極端な条件のために 2 つの語が使用されます。

    - すべてのユーザーを許可する: any
    - どのユーザーも許可しない: none

ポリシーの例については :ref:`sample_auth_policy` を参照してください。

ポリシー評価
^^^^^^^^^^^^^^^^^
ポリシー評価とは、「このユーザーはこのコマンドを実行することを許可されているか」という問いに答えることです。

評価アルゴリズムは次のとおりです。

    - このコマンドとユーザーロールに対してコントロールが定義されている場合は、そのコントロールが評価されます
    - そうでない場合、コマンドがあるカテゴリに属し、そのカテゴリとユーザーロールに対してコントロールが定義されていれば、そのコントロールが評価されます
    - いずれでもない場合は、False を返します

省略記法として、あるロールのすべての権限に対してコントロールが同じ場合は、権限を 1 つずつ明示的に指定せずに、ロールに対してコントロールを指定できます。たとえば、"project_admin" ロールはすべてを実行できるため、この記法が使用されます。

コマンドの認可プロセス
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
ユーザーは FLARE コンソールを介した管理コマンドで NVFLARE システムを操作します。しかし、ユーザーがコマンドを発行したとき、
システム全体ではどのように認可が行われるのでしょうか。

コマンドがサーバーのみに関わる場合は、サーバーの認可ポリシーが評価され適用されます。コマンドが FL クライアントに関わる場合、
そのコマンドはサーバー側で認可の評価を行わずにそれらのクライアントへ送信されます。
クライアントはコマンドを受信すると、自身の認可ポリシーを評価します。クライアントは認可を通過した場合にのみコマンドを実行します。
そのため、一部のクライアントはコマンドを受け入れ、他のクライアントは受け入れない、ということも起こり得ます。

クライアントがコマンドを拒否した場合は、"authorization denied" エラーをサーバーに返します。

ジョブの送信
""""""""""""""
ジョブの送信は、NVFLARE における特別かつ重要な機能です。研究者は "submit_job" コマンドを使ってジョブを送信します。しかしジョブは、
後でスケジュールされデプロイされるまで実行されません。ジョブがスケジュールされる時点で、ユーザーがオンラインであるとは限らないことに
注意してください。

ジョブの認可は 2 か所で行われます。ジョブが送信されるとき、サーバーのみが "submit_job" 権限を評価します。許可された場合、
ジョブは Job Store に受け入れられます。後にジョブが実行のためにスケジュールされると、そのジョブに関わるすべてのサイト
(FL サーバーおよびクライアント) が、それぞれの認可ポリシーに基づいて再度 "submit_job" を評価します。ジョブがカスタムコードを
伴う場合は、"byoc" 権限も評価されます。いずれかの権限が失敗した場合、ジョブは拒否されます。

したがって、送信時にはジョブが受け入れられたにもかかわらず、FL クライアントからの認可エラーによって実行できない、ということも
十分に起こり得ます。

スタディスコープの認可
""""""""""""""""""""""""""
マルチスタディが有効になっている場合、スタディスコープの認可におけるユーザーのロールは、証明書のロールではなく、アクティブな
スタディセッションによって決定されます。ログイン時に、サーバーはユーザーがそのスタディの ``admins`` 設定にマッピングされていることを
検証し、以降のスタディスコープの認可チェックにはマッピングされたロールを使用します。つまり、同じユーザーがスタディごとに異なる権限を
持つことができます。設定の詳細については :ref:`multi_study_guide` を参照してください。

ジョブの送信時に、関わる各 FL クライアントに対して認可を確認しないのはなぜかと疑問に思うかもしれません。これには 3 つの理由があります。

1) サーバーがクライアントとやり取りする必要が生じ、システムがより複雑になります
2) 送信の時点では、一部またはすべての FL クライアントがオンラインでない可能性があります
3) ジョブのクライアントは、利用可能なすべてのクライアントにデプロイされるという意味で、対象が確定していない場合があります。利用可能なクライアントのリストは、ジョブが実行のためにスケジュールされる時点では異なっている可能性があります。

ジョブ管理コマンド
"""""""""""""""""""""""
"manage_jobs" カテゴリには複数のコマンド (clone_job、delete_job、download_job など) があります。これらのコマンドはサーバー上でのみ実行され、FL クライアントは一切関与しません。したがって、組織がこれらのコマンドに対してコントロールを定義したとしても、そのコントロールは効果を持ちません。

ジョブ管理コマンドの認可では、例に示すように、対象となるユーザーとジョブの送信者との関係を評価することがよくあります。

.. _command_categories:

コマンドカテゴリ
------------------

.. code-block:: python

    class CommandCategory(object):

    MANAGE_JOB = "manage_job"
    OPERATE = "operate"
    VIEW = "view"
    SHELL_COMMANDS = "shell_commands"


    COMMAND_CATEGORIES = {
        AC.ABORT: CommandCategory.MANAGE_JOB,
        AC.ABORT_JOB: CommandCategory.MANAGE_JOB,
        AC.START_APP: CommandCategory.MANAGE_JOB,
        AC.DELETE_JOB: CommandCategory.MANAGE_JOB,
        AC.DELETE_WORKSPACE: CommandCategory.MANAGE_JOB,
        AC.CONFIGURE_JOB_LOG: CommandCategory.MANAGE_JOB,

        AC.CHECK_STATUS: CommandCategory.VIEW,
        AC.SHOW_STATS: CommandCategory.VIEW,
        AC.RESET_ERRORS: CommandCategory.VIEW,
        AC.SHOW_ERRORS: CommandCategory.VIEW,
        AC.LIST_JOBS: CommandCategory.VIEW,

        AC.SYS_INFO: CommandCategory.OPERATE,
        AC.RESTART: CommandCategory.OPERATE,
        AC.SHUTDOWN: CommandCategory.OPERATE,
        AC.REMOVE_CLIENT: CommandCategory.OPERATE,
        AC.DISABLE_CLIENT: CommandCategory.OPERATE,
        AC.ENABLE_CLIENT: CommandCategory.OPERATE,
        AC.SET_TIMEOUT: CommandCategory.OPERATE,
        AC.CALL: CommandCategory.OPERATE,
        AC.CONFIGURE_SITE_LOG: CommandCategory.OPERATE,

        AC.SHELL_CAT: CommandCategory.SHELL_COMMANDS,
        AC.SHELL_GREP: CommandCategory.SHELL_COMMANDS,
        AC.SHELL_HEAD: CommandCategory.SHELL_COMMANDS,
        AC.SHELL_LS: CommandCategory.SHELL_COMMANDS,
        AC.SHELL_PWD: CommandCategory.SHELL_COMMANDS,
        AC.SHELL_TAIL: CommandCategory.SHELL_COMMANDS,
    }


.. _sample_auth_policy:

解説付きのサンプルポリシー
-------------------------------

これは authorization.json (サイトのワークスペースの local フォルダー内) の例です。

.. code-block:: shell

    {
        "format_version": "1.0",
        "permissions": {
            "project_admin":  "any",   # can do everything on my site
            "org_admin": {
                "submit_job": "none",  # cannot submit jobs to my site
                "manage_job": "o:submitter",  # can only manage jobs submitted by people in the user's own org
                "download_job": "o:submitter", # can only download jobs submitted by people in the user's own org
                "view": "any", # can do commands in the "view" category
                "operate": "o:site",  # can do commands in the "operate" category only if the user is in my org
                "shell_commands": "o:site"  # can do shell commands only if the user is in my org
            },
            "lead": {
                "submit_job": "any",  # can submit jobs to my sites
                "byoc": "o:site",  # can submit jobs with BYOC to my sites only if the user is in my org
                "manage_job": "n:submitter", # can only manage the user's own jobs
                "view": "any",  # can do commands in "view" category
                "operate": "o:site", # can do commands in "operate" category only if the user is in my org
                "shell_commands": "none", # cannot do shell commands on my site
                "ls": "o:site",  # can do the "ls" shell command if the user is in my org
                "grep": "o:site"  # can do the "grep" shell command if the user is in my org
            },
            "member": {
                "submit_job": [
                    "o:site",  # can submit jobs to my site if the user is in my org
                    "O:orgA", # can submit jobs to my site if the user is in org "orgA"
                    "N:john" # can submit jobs to my site if the user is "john"
                    ],
                "byoc": "none",  # cannot submit BYOC jobs to my site
                "manage_job": "none",  # cannot manage jobs
                "download_job": "n:submitter",  # can download user's own jobs
                "view": "any",  # can do commands in the "view" category
                "operate": "none"  # cannot do commands in "operate" category
            }
        }
    }

.. _site_specific_auth:

サイト固有の認証とフェデレーテッドなジョブレベル認可
==================================================================
サイト固有の認証と認可により、ユーザーは独自の認証および認可の方式を NVFlare システムに組み込むことができます。これには、
FL サーバー/クライアントの登録、認証、およびジョブのデプロイと実行の認可が含まれます。

NVFlare は、次のような機能拡張を可能にするために、汎用的なイベントベースのプラグイン可能な認証・認可フレームワークを提供します。

    - WAF (Web Application Firewall) や、相互トランスポート層セキュリティ (mTLS) を強制するその他のネットワーク要素を通じてアプリを公開する
    - コンフィデンシャル認証局を使用して、参加する各サイトのアイデンティティを保証し、それらがコンフィデンシャルコンピューティングの計算要件を満たしていることを保証する
    - NVFlare 内でどの種類のジョブを誰が送信して実行できるかを管理する追加のロールを定義し、誰がジョブを送信するか、どのデータセットにアクセスできるかを識別する

ユーザーは独自の :ref:`FLComponents <fl_component>` を記述し、ワークフローのさまざまな時点で NVFlare システムのイベントを購読することで、
必要に応じて認証・認可のロジックを簡単に組み込むことができます。

前提とリスク
---------------------
カスタマイズされたサイト固有の認証と認可を有効にすると、NVFlare は IDENTITY_NAME、PUBLIC_KEY、CERTIFICATE など、
セキュリティに関連するいくつかのデータを外部の FL コンポーネントから利用できるようにします。これらが侵害されるのを防ぐため、
そのデータは読み取り専用にする必要があります。

外部のプラグイン可能な認証・認可プロセスを使用するため、それらのプロセスの結果によってジョブがデプロイまたは実行できなくなる
可能性があります。これらの機能を設定して使用する際、ユーザーはその影響を認識し、どこに認証・認可のチェックを組み込むべきかを
把握しておく必要があります。

イベントベースのプラグイン可能な認証と認可
-------------------------------------------------------
NVFlare のイベントベースのソリューションは、サイト固有の認証とフェデレーテッドなジョブレベルの認可をサポートします。
ユーザーは、適切なイベントを購読してカスタムの認証・認可機能を提供する FLComponent を作成して組み込むことで、
あらゆる種類の追加のセキュリティチェックを提供・実装できます。

.. code-block:: python

    class EventType(object):
        """Built-in system events."""

        SYSTEM_START = "_system_start"
        SYSTEM_END = "_system_end"
        ABOUT_TO_START_RUN = "_about_to_start_run"
        START_RUN = "_start_run"
        ABOUT_TO_END_RUN = "_about_to_end_run"
        END_RUN = "_end_run"
        SWAP_IN = "_swap_in"
        SWAP_OUT = "_swap_out"
        START_WORKFLOW = "_start_workflow"
        END_WORKFLOW = "_end_workflow"
        ABORT_TASK = "_abort_task"
        FATAL_SYSTEM_ERROR = "_fatal_system_error"
        FATAL_TASK_ERROR = "_fatal_task_error"
        JOB_DEPLOYED = "_job_deployed"
        JOB_STARTED = "_job_started"
        JOB_COMPLETED = "_job_completed"
        JOB_ABORTED = "_job_aborted"
        JOB_CANCELLED = "_job_cancelled"

        BEFORE_PULL_TASK = "_before_pull_task"
        AFTER_PULL_TASK = "_after_pull_task"
        BEFORE_PROCESS_SUBMISSION = "_before_process_submission"
        AFTER_PROCESS_SUBMISSION = "_after_process_submission"

        BEFORE_TASK_DATA_FILTER = "_before_task_data_filter"
        AFTER_TASK_DATA_FILTER = "_after_task_data_filter"
        BEFORE_TASK_RESULT_FILTER = "_before_task_result_filter"
        AFTER_TASK_RESULT_FILTER = "_after_task_result_filter"
        BEFORE_TASK_EXECUTION = "_before_task_execution"
        AFTER_TASK_EXECUTION = "_after_task_execution"
        BEFORE_SEND_TASK_RESULT = "_before_send_task_result"
        AFTER_SEND_TASK_RESULT = "_after_send_task_result"

        CRITICAL_LOG_AVAILABLE = "_critical_log_available"
        ERROR_LOG_AVAILABLE = "_error_log_available"
        EXCEPTION_LOG_AVAILABLE = "_exception_log_available"
        WARNING_LOG_AVAILABLE = "_warning_log_available"
        INFO_LOG_AVAILABLE = "_info_log_available"
        DEBUG_LOG_AVAILABLE = "_debug_log_available"

        PRE_RUN_RESULT_AVAILABLE = "_pre_run_result_available"

        # event types for job scheduling - server side
        BEFORE_CHECK_CLIENT_RESOURCES = "_before_check_client_resources"

        # event types for job scheduling - client side
        BEFORE_CHECK_RESOURCE_MANAGER = "_before_check_resource_manager"

追加のシステムイベント
^^^^^^^^^^^^^^^^^^^^^^^^
.. code-block:: python

    AFTER_CHECK_CLIENT_RESOURCES = "_after_check_client_resources"
    DEPLOY_JOB_TO_SERVER = "_deploy_job_to_server"
    DEPLOY_JOB_TO_CLIENT = "_deploy_job_to_client"

    BEFORE_SEND_ADMIN_COMMAND = "_before_send_admin_command"

    BEFORE_CLIENT_REGISTER = "_before_client_register"
    AFTER_CLIENT_REGISTER = "_after_client_register"
    CLIENT_REGISTERED = "_client_registered"
    SYSTEM_BOOTSTRAP = "_system_bootstrap"

    AUTHORIZE_COMMAND_CHECK = "_authorize_command_check"


セキュリティチェックの入力
------------------------------
セキュリティチェックに関連するあらゆるデータを保持する ``SECURITY_ITEMS`` の dict を FLContext から利用できるようにします。

NVFlare の標準データ:

.. code-block:: python

    IDENTITY_NAME
    SITE_NAME
    SITE_ORG
    USER_NAME
    USER_ORG
    USER_ROLE
    JOB_META


セキュリティチェックの出力
------------------------------

.. code-block:: python

    AUTHORIZATION_RESULT
    AUTHORIZATION_REASON

NVFlare は ``AUTHORIZATION_RESULT`` を確認して、操作の実行が認可されているかどうかを判断します。各操作の前に、
NVFLare プラットフォームは FLContext 内のすべての ``AUTHORIZATION_RESULT`` を削除します。認可チェックのプロセスの後、
これらの結果が FLContext に存在するかどうかを調べます。存在する場合は、その TRUE/FALSE の値を使ってアクションを決定します。
存在しない場合は、デフォルトで TRUE として扱われます。

イベントを購読して処理する各 FLComponent は、セキュリティデータを使用して、必要に応じた認可チェックの結果を生成できます。
ワークフローは、すべての FLComponent がセキュリティチェックを通過した場合にのみ継続します。いずれか 1 つの FLComponent が
FALSE の値を持つと、ワークフローは実行を停止します。

FLARE コンソールのイベントサポート
------------------------------------
サイト固有のカスタマイズされた認証のために追加のセキュリティデータをサポートするには、FLARE コンソールにイベントベースの
ソリューションのサポートを追加する必要があります。これらのイベントを使用することで、FLARE コンソールはカスタムの SSL 証明書などの
セキュリティ関連データを追加し、サイト固有の認証チェックのために管理コマンドとともにサーバーへ送信できるようになります。

.. code-block:: python

    BEFORE_ADMIN_REGISTER
    AFTER_ADMIN_REGISTER
    BEFORE_SENDING_COMMAND
    AFTER_SENDING_COMMAND
    BEFORE_RECEIVING_ADMIN_RESULT
    AFTER_RECEIVING_ADMIN_RESULT

.. note::

    サイト固有の認証と認可は、FLARE コンソールと :ref:`flare_api` の両方に適用されます。

クライアント登録時にサーバーへ追加データを送信できるようにする
----------------------------------------------------------------
認証チェックを行うためにクライアントからサーバーへ追加データを送信する必要がある場合、クライアントはそのデータを
パブリックデータとして FL_Context に設定できます。その後、サーバー側は PEER_FL_CONTEXT を通じてそのデータにアクセスできます。
アプリケーションは、EventType.CLIENT_REGISTERED を購読する FLComponent を作成して、必要な認証チェックを実行できます。


サイト固有のセキュリティの例
------------------------------
サイト固有のセキュリティ機能を使用するには、 ``local/custom/security_handler.py`` にカスタムの Security 実装を記述し、
それをサイトの ``resources.json`` でコンポーネントとして設定します。

.. code-block:: python

    from typing import Tuple

    from nvflare.apis.event_type import EventType
    from nvflare.apis.fl_component import FLComponent
    from nvflare.apis.fl_constant import FLContextKey
    from nvflare.apis.fl_context import FLContext
    from nvflare.apis.job_def import JobMetaKey


    class CustomSecurityHandler(FLComponent):

        def handle_event(self, event_type: str, fl_ctx: FLContext):
            if event_type == EventType.AUTHORIZE_COMMAND_CHECK:
                result, reason = self.authorize(fl_ctx=fl_ctx)
                if not result:
                    fl_ctx.set_prop(FLContextKey.AUTHORIZATION_RESULT, False, sticky=False)
                    fl_ctx.set_prop(FLContextKey.AUTHORIZATION_REASON, reason, sticky=False)

        def authorize(self, fl_ctx: FLContext) -> Tuple[bool, str]:
            command = fl_ctx.get_prop(FLContextKey.COMMAND_NAME)
            if command in ["check_resources"]:
                security_items = fl_ctx.get_prop(FLContextKey.SECURITY_ITEMS)
                job_meta = security_items.get(FLContextKey.JOB_META)
                if job_meta.get(JobMetaKey.JOB_NAME) == "FL Demo Job1":
                    return False, f"Not authorized to execute: {command}"
                else:
                    return True, ""
            else:
                return True, ""

``local/resources.json`` では次のようになります。

.. code-block::

    {
        "format_version": 2,
        ...
        "components": [
            {
                "id": "resource_manager",
                "path": "nvflare.app_common.resource_managers.gpu_resource_manager.GPUResourceManager",
                "args": {
                "num_of_gpus": 0,
                "mem_per_gpu_in_GiB": 0
                }
            },
            ...
            {
                "id": "security_handler",
                "path": "security_handler.CustomSecurityHandler"
            }
        ]
    }


上記の例では、"FL Demo Job1" という名前のジョブがサーバーからこのクライアント上で実行されるようスケジュールされると、
クライアントは認可エラーを発生させてジョブの実行を防ぎます。それ以外のジョブは、このクライアント上で実行できます。
