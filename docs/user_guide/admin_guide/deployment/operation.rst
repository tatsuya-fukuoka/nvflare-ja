.. _operating_nvflare:

######################################################################
NVFLARE の運用 - Admin クライアント、コマンド、FLARE API
######################################################################

FL システムは、プロビジョニング時に設定された admin タイプのパッケージによって運用されます。admin パッケージには、サーバーへの接続と認証に使用する鍵ファイルと証明書ファイルが含まれており、``fl_admin.sh`` を実行して Admin Console のコマンドプロンプトから、またはプログラムから :ref:`flare_api` を通じて管理を行うことができます。

Admin コマンドプロンプト
==========================================
``fl_admin.sh`` を実行した後、プロンプトに従って、admin パッケージがプロビジョニングされた参加者の名前を入力してログインします(POCモードの場合は、名前とパスワードに "admin" を使用します)。

ターミナルを特定のスタディにスコープするには、``fl_admin.sh --study cancer-research`` のように起動します。``--study`` を省略した場合、admin ターミナルはそのターミナルセッションで ``default`` スタディを使用します。

スタディセッション内で発行されたスタディ対応コマンドは、そのスタディにスコープされます。``list_jobs`` はアクティブなスタディのジョブのみを表示し、``check_status client`` は登録済みのサイトのみを表示し、``submit_job`` はアクティブなスタディでジョブにタグ付けします。詳細は :ref:`multi_study_guide` を参照してください。

"help" または "?" と入力すると、コマンドの一覧と各コマンドの簡単な説明が表示されます。"? check_status" や "?ls" のようにコマンドの前に "? " を入力すると、そのコマンドの使用方法に関する詳細が表示されます。以下に、コマンドの実行例と説明の一覧を示します。

.. csv-table::
    :header: コマンド,例,説明
    :widths: 15, 20, 30

    bye,``bye``,クライアントを終了します
    help,``help``,コマンドのヘルプ情報を取得します
    lpwd,``lpwd``,admin クライアントのローカルワークスペースのルートディレクトリを表示します
    info,``info``,フォルダ設定情報(アップロード・ダウンロードの送信元と送信先)を表示します
    check_status,``check_status server``,"FL ジョブ ID、FL サーバーのステータス、および登録されたクライアントの名前とトークンが表示されます。トレーニングが実行中の場合は、ラウンド情報も表示されます。"
    ,``check_status client``,"接続中の各クライアントの名前、トークン、ステータスが表示されます。"
    ,``check_status client clientname``,"*clientname* で指定したクライアントの名前、トークン、ステータスが表示されます。"
    submit_job,``submit_job job_folder_name``,ジョブをサーバーに送信します。
    list_jobs,``list_jobs``,サーバー上のジョブを一覧表示します。(オプション: [-n name_prefix] [-d] [job_id_prefix])
    configure_job_log,``configure_job_log job_id server config``,"サーバー上のジョブログを設定します。(*config* には json 設定ファイルへのパス、levelname/levelnumber、または 'reload' を指定できます)"
    ,``configure_job_log job_id client <client-name>... config``,対象のクライアント上のジョブログを設定します。
    abort_job,``abort_job job_id``,指定した job_id のジョブが実行中またはディスパッチ済みの場合に中止します
    clone_job,``clone_job job_id``,指定したジョブのコピーを新しい job_id で作成します
    abort,``abort job_id client``,指定した job_id のジョブを全クライアントで中止します。*clientname* を指定すると個々のクライアントのジョブを中止できます。
    ,``abort job_id server``,指定した job_id のサーバージョブを中止します。
    download_job,``download_job job_id``,ジョブとワークスペースを含むフォルダをジョブストアからダウンロードします。大きなジョブの場合、ジョブストア内でのワークスペース作成に追加の遅延が生じることがあります(その前にジョブをダウンロードしようとすると、ワークスペースのデータを取得できない場合があります)
    delete_job,``delete_job job_id``,ジョブストアからジョブを削除します
    cat,``cat server startup/fed_server.json -ns``,ファイルの内容を表示します(-n: すべての出力行に行番号を付ける; -s: 連続する空の出力行をまとめる)
    ,``cat clientname startup/start_docker.sh -bT``,ファイルの内容を表示します(-b: 空でない出力行に行番号を付ける; -T: TAB 文字を ^I として表示する)
    grep,``grep server "info" -i log.txt``,ファイル内のパターンを検索します(-n: 行番号を表示する; -i: 大文字小文字を区別しない)
    head,``head clientname log.txt``,ファイルの先頭 10 行を表示します
    ,``head server log.txt -n 15``,ファイルの先頭 15 行を表示します(-n: 先頭 10 行の代わりに先頭 N 行を表示する)
    tail,``tail clientname log.txt``,ファイルの末尾 10 行を表示します
    ,``tail server log.txt -n 15``,ファイルの末尾 15 行を表示します(-n: 末尾 10 行の代わりに末尾 N 行を出力する)
    ls,``ls server -alt``,ワークスペースのルートディレクトリのファイルを一覧表示します(-a: すべて; -l: 長い一覧形式を使用する; -t: 更新時刻でソートする)
    ,``ls clientname -SR``,ワークスペースのルートディレクトリのファイルを一覧表示します(-S: ファイルサイズでソートする; -R: サブディレクトリを再帰的に一覧表示する)
    pwd,``pwd server``,ワークスペースのルートディレクトリ名を表示します
    ,``pwd clientname``,ワークスペースのルートディレクトリ名を表示します
    configure_site_log,``configure_job_log server config``,"サーバー上のサイトログを設定します。(*config* には json 設定ファイルへのパス、levelname/levelnumber、または 'reload' を指定できます)"
    ,``configure_site_log client <client-name>... config``,対象のクライアント上のサイトログを設定します。
    sys_info,``sys_info server``,システム情報を取得します
    ,``sys_info client *clientname*``,システム情報を取得します。*clientname* を指定すると個々のクライアントを対象にできます。
    restart,``restart client``,すべてのクライアントを再起動します。*clientname* を指定すると個々のクライアントを再起動できます。
    ,``restart server``,サーバーを再起動します。クライアントも再起動されます。サーバーの再起動後、admin クライアントは再度ログインする必要があることに注意してください。
    shutdown,``shutdown client``,すべてのクライアントをシャットダウンします。*clientname* を指定すると個々のクライアントをシャットダウンできます。即時に反映されるとは限らず、コマンドが効果を発揮するまで時間がかかる場合があることに注意してください。
    ,``shutdown server``,サーバーをシャットダウンします。サーバーをシャットダウンする前に、まずクライアントをシャットダウンする必要があります。
    cells,``cells``,"システム内のすべてのアクティブなセルを FQCN (Fully Qualified Cell Name) とともに一覧表示します。他の診断コマンドで利用可能なターゲットを見つけるために使用します。"
    list_pools,``list_pools target``,"対象セル上のすべての統計プールを一覧表示します。各プールのプール名、タイプ(hist または counter)、説明を表示します。"
    show_pool,``show_pool target pool_name [mode]``,"対象セル上の特定のプールの詳細な統計を表示します。オプションの *mode* パラメーターには count、percent、avg、min、max のいずれかを指定できます(ヒストグラムプールの場合のデフォルトは count)。"
    msg_stats,``msg_stats target [mode]``,"対象セルのメッセージリクエスト統計を表示します。オプションの *mode* パラメーターには count、percent、avg、min、max のいずれかを指定できます(デフォルトは count)。メッセージのサイズとタイミングに関する統計を表示します。"

.. note::

   注記: ``cells``、``list_pools``、``show_pool``、``msg_stats`` コマンドは診断コマンドであり、システムで diagnose モードが有効に設定されている場合にのみ利用できます。これらのコマンドの使用例、統計モード、トラブルシューティングを含む詳細情報については、:ref:`diagnostic_commands` を参照してください。

.. tip::

   任意のコマンドの出力は、大なり記号 ">" を使用してファイルにリダイレクトできます。ただし、ファイル名の前に空白を入れてはいけません。例えば、``sys_info server >serverinfo.txt`` のように実行できます。出力を表示せずにファイルへの保存のみを行うには、代わりに大なり記号を2つ ">>" 使用します: ``sys_info server >>serverinfo.txt``。

FLARE API は、FL サーバーに対してプログラムから admin コマンドを発行するためにサポートされている Python インターフェースです。

Python によるジョブ管理については、:ref:`FLARE API <flare_api>` を参照してください。
