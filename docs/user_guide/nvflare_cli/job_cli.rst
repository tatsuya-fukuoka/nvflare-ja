.. _job_cli:

#############################
NVIDIA FLARE ジョブ CLI
#############################

``nvflare job`` コマンドファミリは、管理者用スタートアップキットから連合学習ジョブを投入し、確認し、
監視し、管理するために使用します。

サーバー接続を伴うジョブコマンドを使用する前に、 ``nvflare poc prepare`` を実行するか、
:ref:`config_command` で登録済みのスタートアップキットを有効化してください。

.. code-block:: shell

   nvflare config add project_admin /path/to/admin@nvidia.com
   nvflare config use project_admin

***********************
コマンドの使い方
***********************

.. code-block:: none

   nvflare job -h

   usage: nvflare job [-h]  ...

   job subcommands:
     submit          submit job
     wait            wait for a job and return one final JSON envelope
     monitor         wait for a job and stream progress to stderr
     list            list jobs on the server
     abort           abort a running job
     meta            get metadata for a job
     logs            retrieve job logs from the server-side log store
     log-config      change logging configuration for a running job
     stats           show running job statistics
     download        download job result
     clone           clone an existing job
     delete          delete a job
     list_templates  [DEPRECATED] use 'nvflare recipe list'
     create          [DEPRECATED] use 'python job.py --export --export-dir <job_folder>' + 'nvflare job submit -j <job_folder>'
     show_variables  [DEPRECATED] use 'nvflare recipe list' or the Job Recipe API

*************************
一般的なワークフロー
*************************

1. ジョブフォルダをエクスポートまたは準備します。
2. ``nvflare job submit -j <job_folder>`` でジョブを投入します。
3. 自動化では ``nvflare job wait <job_id>`` で完了を待ちます。
   対話的に進捗を表示したい場合は ``nvflare job monitor <job_id>`` を使用します。
4. 必要に応じて、メタデータ、統計情報、ログを確認します。
5. 適切なタイミングで、ジョブのダウンロード、クローン、中断、削除を行います。

*********************************
スタートアップキットの選択
*********************************

サーバー接続を伴うジョブコマンドは、次の順序でスタートアップキットを解決します。

1. 任意の ``--kit-id <id>``: 登録済みのスタートアップキット ID を使用して、このコマンドに限り有効な
   スタートアップキットを上書きします。
2. 任意の ``--startup-kit <path>``: 明示的な管理者用スタートアップキットディレクトリを使用して、この
   コマンドに限り有効なスタートアップキットを上書きします。
3. 設定されている場合は ``NVFLARE_STARTUP_KIT_DIR`` 。
4. ``~/.nvflare/config.conf`` の ``startup_kits.active`` 。
5. いずれの情報源からも有効な管理者用スタートアップキットが解決できない場合、コマンドは接続前に失敗します。

``--kit-id`` と ``--startup-kit`` は必須ではありません。指定された場合は、現在のコマンドに限り有効な
スタートアップキットより優先され、グローバルに有効なスタートアップキットは変更されません。これらは、
``~/.nvflare/config.conf`` を変更してはならないスクリプト、ノートブック、並行ワークフローで有用です。

********************
ジョブの投入
********************

ビルド済みの NVFlare ジョブフォルダを投入するには、 ``nvflare job submit`` を使用します。

.. code-block:: shell

   nvflare job submit -j /tmp/nvflare/hello-pt

submit のオプション:

- ``-j, --job_folder``: ジョブフォルダのパスです。既定値は ``./current_job`` です。
- ``--study``: サーバーがマルチスタディアクセス用に構成されている場合に、名前付きスタディへ投入します。
  省略した場合は、リテラルのスタディ名 ``default`` が投入されます。
- ``--submit-token``: リトライ安全な投入と、後から ``nvflare job list --submit-token`` で復旧するための、
  呼び出し側が生成するトークンです。
- ``-debug, --debug``: 確認のために、一時的にコピーされたジョブフォルダを保持します。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

submit は ``job_id`` を返して直ちに戻ります。ジョブが終端ステータスに達するまで待機することはありません。

ジョブの設定値を変更するには、投入前にエクスポートされたジョブファイルを編集してください。投入時の
``-f/--config_file`` による上書きはサポートされていません。

例:

.. code-block:: shell

   nvflare config use project_admin
   nvflare job submit -j /tmp/nvflare/hello-pt
   nvflare job list --kit-id project_admin
   nvflare job submit -j /tmp/nvflare/hello-pt --startup-kit /path/to/admin@nvidia.com

登録するスタートアップキットのパスは、より広い ``prod_00`` のルートではなく、管理者用スタートアップキット
ディレクトリそのものを指す必要があります。

JSON の成功レスポンスの例:

.. code-block:: json

   {"schema_version": "1", "status": "ok", "exit_code": 0, "data": {"job_id": "abc123"}}

サーバーがスタディ用に構成されている場合は、次のように明示的に対象を指定できます。

.. code-block:: shell

   nvflare job submit -j /tmp/nvflare/my_job --study cancer_research

リトライ安全な投入トークン
==========================

自動化された呼び出し側が、タイムアウトやクライアント接続の切断の後に投入をリトライする可能性がある場合は、
``--submit-token`` を使用します。

.. code-block:: shell

   TOKEN=$(uuidgen)
   nvflare job submit -j /tmp/nvflare/my_job \
       --study cancer_research \
       --submit-token "$TOKEN" \
       --format json

``--submit-token`` は任意です。指定する場合は呼び出し側が生成する必要があり、1 回の意図した投入に対する
冪等性および復旧用の値として使用されます。このフラグを省略した場合、NVFlare が投入トークンを自動生成する
ことはありません。このトークンは認証トークン、セッショントークン、スタートアップキットの資格情報、API キー、
証明書の秘密情報のいずれでもありません。通常のスタートアップキットによる認証と認可は引き続き適用されます。

トークンは空でなく、128 文字以内で、英字、数字、 ``.`` 、 ``_`` 、 ``:`` 、 ``-`` のみを使用する必要が
あります。

投入トークンのスコープは、選択されたサーバー／プロジェクトのコンテキスト、スタディ、投入者のアイデンティティ、
およびトークンの値です。同じスコープ内で、同じジョブ内容に対して同じトークンを再利用すると、既存の
``job_id`` が返されます。異なるジョブ内容で再利用した場合は ``SUBMIT_TOKEN_CONFLICT`` で失敗します。
スタディはジョブの名前空間として分離されているため、同じトークンを別のスタディで使用することはできます。

``--submit-token`` を使って作成されたジョブが後から削除された場合、サーバーは投入レコードを
``job_deleted`` として保持します。同じトークンで後から投入や list の検索を行うと、削除されたジョブが
黙って再作成されることはなく、 ``SUBMIT_TOKEN_JOB_DELETED`` が返されます。そのジョブを再度投入するには、
新しい投入トークンを使用してください。

投入するジョブのパスは、ジョブ内容のルートを指す必要があります。投入する成果物が、ジョブ内容を 1 つの
ラッパーディレクトリで包んだ zip ファイルである場合、投入トークンの内容ハッシュの計算ではそのラッパーが
無視されます。そのため、通常の ``zip -r my_job.zip my_job/`` によるアーカイブは、 ``my_job/`` を直接投入した
場合と一致します。 ``my_job/`` を含む親ディレクトリを投入した場合は異なる内容とみなされ、同じトークンで
リトライすると衝突する可能性があります。

トークンは、サーバーが所有する投入メタデータとしてのみ保存されます。ジョブの ``meta.json`` には書き込まれず、
このファイルは ``deploy_map`` 、 ``resource_spec`` 、 ``min_clients`` 、ランチャー設定といった、ジョブが
所有する実行メタデータのままです。 ``--submit-token`` を省略した場合、投入の動作は変わらず、各投入は
従来どおり新しいジョブを作成します。サーバーは通常のジョブストアおよびジョブ履歴を通じて投入されたジョブを
記録しますが、リトライ安全な投入トークンのレコードは作成されません。元の投入で呼び出し側が用意したトークンが
使われていない限り、そのジョブを後から ``job list --submit-token`` で復旧することはできません。

クライアント側のタイムアウトやセッションの喪失が発生した後は、 ``job list --submit-token`` で受理済みの
ジョブを復旧します。

.. code-block:: shell

   nvflare job list --study cancer_research --submit-token "$TOKEN" --format json

復旧対象のジョブが削除されていた場合、JSON 出力は通常のエラーエンベロープを使用します。

.. code-block:: json

   {
     "schema_version": "1",
     "status": "error",
     "exit_code": 4,
     "error_code": "SUBMIT_TOKEN_JOB_DELETED",
     "data": {
       "job_id": "abc123",
       "state": "job_deleted",
       "deleted_time": "2026-04-30T10:00:00-07:00"
     }
   }

``--submit-token`` は ``job submit`` と ``job list`` のみで使用できます。復旧したジョブを監視、
ダウンロード、中断、削除、クローンするには、まず ``job list --submit-token`` で ``job_id`` を解決し、
その後に通常のジョブコマンドを使用してください。

***********************************
ジョブの待機または監視
***********************************

スクリプトやエージェントが、ジョブが終端状態に達した後に 1 つの最終的なコマンド結果を必要とする場合は、
``nvflare job wait`` を使用します。

.. code-block:: shell

   nvflare job wait <job_id>
   nvflare job wait <job_id> --study cancer_research
   nvflare job wait <job_id> --timeout 3600 --interval 5 --format json

``job wait`` は次の引数を受け付けます。

- ``job_id``: 待機対象のジョブ ID です。
- ``--timeout``: 待機する最大秒数です。 ``0`` 以上である必要があります。
  既定値: ``0`` （タイムアウトなし）。
- ``--interval``: ポーリング間隔（秒）です。 ``0`` より大きい必要があります。
  既定値: ``2`` 。
- ``--study``: 名前付きスタディ内のジョブを待機します。投入時に使用したものと同じスタディ名を指定して
  ください。省略した場合は、リテラルのスタディ名 ``default`` が使用されます。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

``job monitor`` とは異なり、 ``job wait`` は単一エンベロープの自動化向けコマンドです。進捗行を
ストリーミングすることはありません。JSON モードでは、標準出力には終端のジョブステータスとメタデータを
含む最終的な JSON エンベロープがちょうど 1 つだけ出力され、人間向けの診断情報は引き続き標準エラー出力に
送られます。

終了時の動作:

- 終了コード ``0``: ジョブが正常に完了しました。
- 終了コード ``1``: ジョブが ``FAILED`` 、 ``FINISHED_EXCEPTION`` 、 ``ABORTED`` 、 ``ABANDONED`` などの
  終端の失敗状態に達しました。
- 終了コード ``2``: 接続、認証、または認可の失敗により待機できませんでした。
- 終了コード ``3``: 待機がタイムアウトしました。

これにより、進捗出力を解析することなく CI/CD 形式の連結が可能になります。

.. code-block:: shell

   JOB=$(nvflare job submit -j ./my_job --format json | jq -r .data.job_id)
   nvflare job wait $JOB --format json && nvflare job download $JOB

待機中に人間が進捗の更新を確認したい場合は、 ``nvflare job monitor`` を使用します。ステータス行を標準
エラー出力にストリーミングし、ジョブが終端状態に達したときに最終結果を返します。

.. code-block:: shell

   nvflare job monitor <job_id>
   nvflare job monitor <job_id> --study cancer_research
   nvflare job monitor <job_id> --timeout 3600 --format jsonl

monitor のオプション:

- ``job_id``: 監視対象のジョブ ID です。
- ``--timeout``: 待機する最大秒数です。 ``0`` 以上である必要があります。
  既定値: ``0`` （タイムアウトなし）。
- ``--interval``: ポーリング間隔（秒）です。 ``0`` より大きい必要があります。
  既定値: ``2`` 。
- ``--study``: 名前付きスタディ内のジョブを監視します。投入時に使用したものと同じスタディ名を指定して
  ください。省略した場合は、リテラルのスタディ名 ``default`` が使用されます。
- ``--stats-target``: 統計情報の取得元です。選択肢: ``server`` 、 ``client`` 、 ``all`` 。既定値: ``server`` 。
- ``--metric``: 統計情報から表示する追加のメトリックキーです。繰り返し指定できます。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

``job monitor`` の終了時の動作は ``job wait`` と同じです。

- 終了コード ``0``: ジョブが正常に完了しました
- 終了コード ``1``: ジョブが終端の失敗状態（ ``FAILED`` 、 ``FINISHED_EXCEPTION`` 、 ``ABORTED`` 、
  ``ABANDONED`` ）に達しました
- 終了コード ``2``: 接続、認証、または認可の失敗により監視できませんでした
- 終了コード ``3``: 監視がタイムアウトしました

進捗イベントを必要とする自動化では ``--format jsonl`` を使用します。標準出力の各行が 1 つの完全な JSON
オブジェクトになります。進捗イベントには ``terminal: false`` が含まれ、最終イベントには常に
``terminal: true`` が含まれます。タイムアウトの場合は ``status: "TIMEOUT"`` を持つ最終イベントを出力し、
終了コード ``3`` で終了します。 ``FINISHED_OK`` のような成功時の終端ジョブステータスは
``status: "COMPLETED"`` に正規化され、サーバーの生のステータスは ``job_status`` に保持されます。接続、
認証、認可の失敗では、 ``status: "error"`` と具体的なコードを ``error_code`` に含む終端のエラーイベントが
出力されます。

JSONL の終端イベントの例:

.. code-block:: json

   {"schema_version":"1","event":"terminal","job_id":"abc123","status":"COMPLETED","job_status":"FINISHED_OK","terminal":true}

*******************************
ジョブの一覧表示と確認
*******************************

現在サーバーが把握しているジョブを一覧表示します。

.. code-block:: shell

   nvflare job list

よく使う list のフィルタ:

- ``-n, --name``: ジョブ名のプレフィックスで絞り込みます。
- ``-i, --id``: ジョブ ID のプレフィックスで絞り込みます。
- ``-r, --reverse``: ソート順を逆にします。
- ``-m, --max``: 返す結果の最大件数です。
- ``--study``: 名前付きスタディのジョブを一覧表示します。省略した場合は、リテラルのスタディ名
  ``default`` が使用されます。 ``all`` のような値はそのままサーバーに渡されます。
- ``--submit-token``: 選択されたスタディ内で、リトライ安全な投入トークンに紐づくジョブを検索します。
  これは ``--submit-token`` を使って投入した後の復旧経路です。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

単一のジョブのメタデータを取得します。

.. code-block:: shell

   nvflare job meta <job_id>
   nvflare job meta <job_id> --study cancer_research

メタデータは、投入後にジョブの識別情報、ライフサイクルのフィールド、サーバーが報告するステータス情報を
確認するために使用します。人間向けの出力は簡潔なサマリーにまとめられます。生のメタデータエンベロープ全体を
取得するには ``--format json`` を使用してください。

ジョブ ID による検索および制御を行うすべてのコマンドは ``--study`` を受け付けます。投入時に使用したものと
同じスタディ名を指定してください。省略した場合、コマンドはリテラルの ``default`` スタディを検索します。
ジョブが見つからない場合、エラーはどのスタディを検索したかを報告し、 ``--study`` を付けて再試行するよう
提案します。

``nvflare job meta`` も ``--schema`` をサポートします。

********************************************
ダウンロード・クローン・中断・削除
********************************************

ジョブの結果をダウンロードします。

.. code-block:: shell

   nvflare job download <job_id> -o ./downloads
   nvflare job download <job_id> --study cancer_research -o ./downloads
   nvflare job download <job_id> --study cancer_research --force

自動化では JSON 出力を使用します。

.. code-block:: shell

   nvflare job download <job_id> -o ./downloads --format json

ダウンロードの前に、ジョブは終端状態になっている必要があります。実行中のジョブの場合は、まず待機します。

.. code-block:: shell

   nvflare job wait <job_id> --study cancer_research
   nvflare job download <job_id> --study cancer_research

ローカルの保存先の既定値は ``./<job_id>`` です。そのディレクトリが既に存在する場合、 ``--force`` を
指定しない限りコマンドは失敗します。 ``--force`` は、既存のローカルダウンロードを置き換えることが意図
されている場合にのみ使用してください。

人間向けの出力は簡潔なままで、最終的なダウンロード先のみを表示します。エージェントやスクリプトが
ダウンロードされた成果物のパスを必要とする場合は ``--format json`` を使用してください。JSON の成功
レスポンスは、CLI を実行しているマシン上のローカルパスを報告します。

.. code-block:: json

   {
     "schema_version": "1",
     "status": "ok",
     "exit_code": 0,
     "data": {
       "job_id": "abc123",
       "download_path": "/abs/path/downloads/abc123",
       "path": "/abs/path/downloads/abc123",
       "artifact_discovery": "completed",
       "artifacts": {
         "global_model": "/abs/path/downloads/abc123/workspace/FL_global_model.pt",
         "metrics_summary": "/abs/path/downloads/abc123/workspace/metrics/metrics_summary.json",
         "round_metrics": "/abs/path/downloads/abc123/workspace/metrics/round_metrics.jsonl",
         "client_logs": {
           "site-1": "/abs/path/downloads/abc123/workspace/site-1/log.txt"
         }
       },
       "missing_artifacts": []
     }
   }

``download_path`` は、ダウンロード API が返す最終的なローカルディレクトリです。
``path`` は、存在する場合の ``download_path`` の後方互換エイリアスです。

``artifacts`` には、 ``download_path`` の下で見つかったファイルのローカルパスが含まれます。エージェントや
スクリプトは、サーバーのワークスペースレイアウトを前提としたり ``download_path`` からパスを組み立てたり
するのではなく、利用可能なファイルの正となる情報源として ``data.artifacts.*`` を使用してください。
``missing_artifacts`` には、モデル、メトリクス、クライアントログなど、ローカルに見つからなかった想定
カテゴリが列挙されます。ダウンロード自体が成功していれば、成果物が欠けていてもコマンドが失敗することは
ありません。 ``round_metrics`` は、ラウンドごとの JSONL 成果物が存在する場合に報告されます。古いジョブや
集約メトリクスを持たないジョブでは作成されないため、これは任意です。

グローバルモデルの検出には、ダウンロードツリー内のファイルの順序ではなく、サーバーのプロベナンスが
使用されます。一般的なモデルのファイル名は、 ``workspace`` 、 ``server`` 、 ``app_server`` といった
サーバーが所有する正規の場所においてのみ考慮され、クライアントのチェックポイントがグローバルモデルとして
ラベル付けされることはありません。グローバルモデルをカスタムのファイル名で保存するジョブは、サーバーが
所有する正規の場所に ``artifact_manifest.json`` を配置できます。マニフェストにはスキーマバージョン ``1``
と、マニフェストからの相対パスが必要です。

.. code-block:: json

   {
     "schema_version": "1",
     "artifacts": {
       "global_model": "app_server/production-checkpoint.bin"
     }
   }

マニフェストのパスは、ダウンロードされたジョブツリーの内部に留まる必要があり、シンボリックリンクを
たどることはできません。マニフェストが存在する場合はそれが正となり、マニフェストが不正であったり対象が
存在しなかったりしても、CLI はファイル名の推測にフォールバックしません。

``artifact_discovery`` が ``skipped`` の場合、CLI には検査対象のローカルディレクトリがなかったため、
想定される成果物が存在しないことを検証済みであると主張する代わりに、 ``artifacts`` と
``missing_artifacts`` は ``null`` になります。

サーバーのダウンロードプロトコルは変更されていません。これらの成果物のパスは、結果がダウンロードされた後に
CLI がローカルで計算したものです。

既存のジョブをクローンします。

.. code-block:: shell

   nvflare job clone <job_id>
   nvflare job clone <job_id> --study cancer_research

``nvflare job clone`` は、再利用のためにサーバー側のジョブ全体をクローンします。現在の CLI のインター
フェースは、クローン元の ``job_id`` 、任意の ``--study`` 、 ``--schema`` を受け取ります。戻り値として
``source_job_id`` と ``new_job_id`` を返します。クローンされたジョブを監視・管理するには、返された
``new_job_id`` を使用してください。

実行中のジョブを中断します。

.. code-block:: shell

   nvflare job abort <job_id>
   nvflare job abort <job_id> --study cancer_research
   nvflare job abort <job_id> --force

ジョブを削除します。

.. code-block:: shell

   nvflare job delete <job_id>
   nvflare job delete <job_id> --study cancer_research
   nvflare job delete <job_id> --force

注意事項:

- ``abort`` と ``delete`` は、確認プロンプトをスキップする ``--force`` をサポートします。
- ``abort`` と ``delete`` は、選択されたスタディを検索します。省略した場合は ``default`` が使用されます。
- ``delete --format json`` は ``job_id`` と ``submit_records_marked_deleted`` を返します。この件数が
  0 でない場合、同じ投入トークンを今後使用すると ``SUBMIT_TOKEN_JOB_DELETED`` が返されます。
- ``download`` は、保存先ディレクトリを選択する ``-o, --output-dir`` をサポートします。既定値は、
  カレントワーキングディレクトリ配下のジョブ固有のディレクトリ（ ``./<job_id>`` ）です。
- ``clone`` 、 ``download`` 、 ``abort`` 、 ``delete`` はいずれも ``--schema`` をサポートします。

**********************
オブザーバビリティ
**********************

サーバー側のログストアからジョブのログを取得します。

.. code-block:: shell

   nvflare job logs <job_id>
   nvflare job logs <job_id> --site site-1
   nvflare job logs <job_id> --site all
   nvflare job logs <job_id> --site all --tail 200
   nvflare job logs <job_id> --site site-1 --since 2026-04-28T10:00:00
   nvflare job logs <job_id> --site all --max-bytes 200000
   nvflare job logs <job_id> --study cancer_research

``job logs`` は次の引数を受け付けます。

- ``--study``: 名前付きスタディ内のジョブのログを取得します。省略した場合、 ``job logs`` は既定の
  スタディを検索します。 ``job submit`` や ``job list`` で使用したものと同じスタディ名を指定してください。
- ``--site server``: サーバーのジョブログを返します。これが既定です。
- ``--site <client_name>``: そのクライアントのジョブログを、サーバーにストリーミングされて保存された後に
  返します。
- ``--site all``: サーバーのログと、サーバー側のログストアに現在利用可能なすべてのクライアントログを
  返します。既知のジョブサイトに保存されたログの内容がない場合、JSON レスポンスではそのサイトが
  ``unavailable`` の下に含まれます。
- ``--sites`` は ``--site`` のエイリアスとして受け付けられますが、選択できる対象は 1 つの値のみです。
- ``--tail N``: サイトごとに、最後の N 行までのログを返します。
- ``--since timestamp``: 行のタイムスタンプが解析可能な場合に、そのタイムスタンプ以降のタイムスタンプ付き
  ログ行を返します。含まれるタイムスタンプ付きの行に続く継続行も含まれます。
- ``--max-bytes N``: サイトごとに、最大 N UTF-8 バイトまでを返します。
- ``job logs`` も ``--schema`` をサポートします。

明示的な上限が指定されていない場合、 ``job logs`` はサイトごとに最後の 500 行までを返します。JSON 出力には
``logs_truncated`` 、 ``sites`` の下のサイトごとの利用可否と行数／バイト数、および適用された ``filters``
が含まれます。 ``--tail`` 、 ``--since`` 、 ``--max-bytes`` のいずれかが指定された場合、既定の 500 行の
tail は無効化され、 ``filters.default_tail_applied`` は ``false`` になります。明示的な上限は
``--since`` 、 ``--tail`` 、 ``--max-bytes`` の順に適用されます。

これらの上限オプションは、サーバーが保存されたログの内容を返した後に CLI 側で適用されます。これらは
``nvflare job logs`` が表示または JSON 出力する内容を制限するものであり、サーバーに要求するログの量を
減らすものではありません。大きなログが CLI に届く前にサーバー側の最大レスポンスサイズで既に制限されている
場合、これらのクライアント側の上限は返された内容に対して適用されます。

通常の人間向け出力モードでは、 ``job logs`` はログのテキストをそのまま表示します。 ``--site all`` を
指定した場合、各サイトは短いヘッダーで区切られます。自動化のために構造化された ``logs`` の辞書が必要な
場合は ``--format json`` を使用してください。

``job logs`` には組み込みの ``grep`` オプションはありません。テキストのマッチングが必要な場合は、返された
内容をパイプするか後処理してください。

クライアントのログは、コマンドの実行時にクライアントのマシンから取得されるわけではありません。このコマンドは、
ジョブの実行中に既にサーバーへストリーミングされていたログをサーバーに要求します。ストリーミングされた
クライアントのログは、サーバーのジョブワークスペースから読み込まれます。そこでは、構成されたログストリーマーに
応じて ``<client_name>/log.txt`` または ``<client_name>/log.json`` として保存されます。ジョブワークスペースが
アーカイブされた後は、保存されたジョブの ``workspace`` 成果物から同じファイルが読み込まれます。

ポータブルな Recipe ジョブでクライアントのジョブログのストリーミングを有効にするには、Recipe のログ
ストリーミングヘルパーを使用します。

.. code-block:: python

   # Streams each client's log.json to the server.
   recipe.enable_log_streaming()

   # Or stream a text log file instead.
   recipe.enable_log_streaming("log.txt")

``resources.json.default`` にあるシステムレベルのログ設定は、このジョブレベルのオプトインとは別のものです。
デプロイメントによっては、サーバー側のログレシーバーをグローバルに構成している場合もありますが、Recipe の
ヘルパーを使用すると、POC と本番のデプロイメントの両方でジョブが自己完結したものになります。

``nvflare job logs --format json`` は、利用可能な場合は ``log.json`` を使用し、そうでない場合は
``log.txt`` にフォールバックします。人間向けの出力は読みやすいテキストを表示します。 ``log.json`` のみが
利用可能な場合、CLI は表示のために JSON のログレコードをテキストとして描画します。

``examples/hello-world/hello-log-streaming`` のサンプルは、このパターンを示しています。

実行中のジョブのログ設定を変更します。

.. code-block:: shell

   nvflare job log-config <job_id> DEBUG
   nvflare job log-config <job_id> concise
   nvflare job log-config <job_id> msg_only
   nvflare job log-config <job_id> DEBUG --study cancer_research

``job log-config`` は次の引数を受け付けます。

- 位置引数 ``level``: ``DEBUG`` 、 ``INFO`` 、 ``WARNING`` 、 ``ERROR`` 、 ``CRITICAL``
- ログモード: ``concise`` 、 ``msg_only`` 、 ``full`` 、 ``verbose`` 、 ``reload``
- ``--site``: 対象のサイト名または ``all`` です。既定値は ``all`` で、 ``--site all`` を明示的に
  指定することは省略した場合と同等です。
- ``--study``: ジョブが含まれるスタディです。省略した場合は ``default`` が使用されます。
- ``--schema``: コマンドスキーマを JSON として出力して終了します

実行中のジョブの統計情報を表示します。

.. code-block:: shell

   nvflare job stats <job_id>
   nvflare job stats <job_id> --study cancer_research

``job stats`` は、ジョブが含まれるスタディを選択する ``--study`` と、特定のサイトまたは ``all`` を対象に
する ``--site`` をサポートします。既定のサイトは ``all`` であるため、 ``--site all`` を明示的に指定する
ことは省略した場合と同等です。
また ``--schema`` もサポートします。

***********************************
Recipe ベースのジョブ作成
***********************************

新しいジョブフォルダを作成する推奨の方法は、Job Recipe API を使用するか、 ``--export`` をサポートする
サンプルの ``job.py`` スクリプトを使用することです。

.. code-block:: shell

   python job.py --export --export-dir /tmp/nvflare/hello-pt
   nvflare job submit -j /tmp/nvflare/hello-pt

組み込みの recipe を探索するには、次を使用します。

.. code-block:: shell

   nvflare recipe list

非推奨のコマンド:

- ``nvflare job create``: 互換性のために残されています。 ``python job.py --export`` の後に ``nvflare job submit`` を実行する方法を推奨します。
- ``nvflare job list_templates``: ``nvflare recipe list`` を使用してください。
- ``nvflare job show_variables``: Job Recipe API を使用してください。

現時点の非推奨に関する注意:

- ``nvflare job create`` は、レガシーなワークフローのために、テンプレートおよび設定を指向した引数を
  引き続き公開しています。
- ``nvflare job list_templates`` と ``nvflare job show_variables`` は後方互換性のために引き続き利用
  できますが、recipe の探索やジョブ変数の確認のための推奨インターフェースではありません。

*************************
JSON 出力とヘルプ
*************************

機械可読な出力を得るには、サブコマンドより後の任意の位置に ``--format json`` を追加します。

.. code-block:: shell

   nvflare job meta <job_id> --format json

``--format json`` は、サブコマンド名より後であればコマンド内のどこに置いても構いません。標準出力には単一の
JSON エンベロープが出力され、人間向けの進捗表示と診断情報は標準エラー出力に送られます。

機械可読なコマンド探索には ``--schema`` を使用します。 ``--schema`` は ``--format`` の指定に関係なく常に
JSON を返すため、このフラグを併用する必要はありません。

.. code-block:: shell

   nvflare job submit --schema
   nvflare job wait --schema
   nvflare job monitor --schema

``mutating`` や ``idempotent`` といったスキーマのフィールドは、1 回の呼び出しの実効的な振る舞いではなく、
コマンド全体の性質を表します。たとえば ``job submit`` は ``idempotent: false`` と報告します。これは、
単純な投入ではタイムアウト後にリトライすると重複したジョブが作成されうるためです。また
``retry_token.supported: true`` も報告し、 ``--submit-token`` によって、同じ投入者が同じスタディに同一の
ジョブ内容を投入する場合のリトライが安全になることを示します。 ``job list --submit-token`` は異なります。
そちらでの ``--submit-token`` は単なる検索フィルタであるため、 ``retry_token.supported`` は ``false``
のままです。

人間向けの引数エラーでは、まずコマンドのヘルプが表示され、その後に具体的なエラーとヒントが表示されます。
JSON モードでは JSON のエラーエンベロープのみが出力されます。
