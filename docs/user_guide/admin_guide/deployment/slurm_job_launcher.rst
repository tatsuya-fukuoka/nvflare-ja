.. _slurm_job_launcher:

##########################
Slurm ジョブランチャー
##########################

このガイドにおける *親* とは、長時間稼働する NVFlare のクライアント親プロセス
(CP) またはサーバ親プロセス (SP) を指します。親プロセスは、連合ジョブごとに個別の
クライアントジョブプロセス (CJ) またはサーバジョブプロセス (SJ) を起動します。Slurm
ジョブランチャーは、各 CJ または SJ を Slurm のバッチジョブとして投入します。Slurm が
ノードを選択して割り当てを制御し、NVFlare は現在の親が所有するジョブの投入、監視、
キャンセルを行います。

*prepare ホスト* は ``nvflare deploy prepare`` を実行します。オプションの *投入ホスト* は
``sbatch`` を実行してクライアント親を Slurm 上に配置します。*ランタイム親ホスト* は
CP または SP を実行し、*計算ノード* がその CJ または SJ の割り当てを実行します。
ログインホストやサービスホストは、Slurm の割り当ての外にあるランタイム親ホストです。

``job_launcher.sandbox`` で実行バックエンドを選択します。

.. list-table::
   :header-rows: 1

   * - 値
     - 実行方式
     - 用途
   * - ``apptainer``
     - ランチャーが管理する Apptainer コンテナ
     - 対象クラスタでの検証後の分離
   * - ``pyxis``
     - 読み取り専用の Pyxis/Enroot コンテナ
     - 信頼済みのコンテナパッケージング
   * - ``none``
     - 割り当て内で Python を直接実行
     - サイトが信頼するコード。マルチノードジョブでは必須

前提条件
=========

親を起動する前に、以下を確認してください。

- Slurm 23.02 以降が、ランタイム親ホスト上で動作する ``sbatch``、``squeue``、``sacct``、
  ``scancel`` の各コマンドを提供していること。親のブートストラップはこれらのコマンドを
  解決してバージョンを検証します。ランチャーが ``sbatch --export=NIL`` を使用するため、
  23.02 が最低要件です。本番サイトでは、SchedMD によるサポートが継続している Slurm
  リリースを使用してください。
- ``slurmdbd`` によるアカウンティングが有効で、``sacct`` が応答すること。デフォルトの
  ``AccountingStoreFlags`` で十分です。
- クラスタがフェデレーション構成でなく、投入プラグインがジョブを別のクラスタへ
  リダイレクトしないこと。
- 親が、親ホスト、共有ファイルシステム、計算ノードにおいて一貫した数値 UID と互換性の
  あるグループアクセスを持つ専用のサイトアカウントを使用していること。
- ランタイムワークスペース、イメージ、データセット、シークレットマウント元が、参加する
  すべてのノードで同一の絶対パスから参照できること。共有ファイルシステムは ``O_EXCL``
  とアトミックなリネームをサポートしていること。
- 計算ノードが ``parent_host:internal_port`` に到達できること。あるいは、サイトが代わりに
  共有ファイルのワーカーチャネルを使用していること (:ref:`slurm_shared_file_channel` を参照)。
  マルチノードジョブでは、割り当てられたノード間の接続性も必要です。
- Slurm のパーティション、アソシエーション、QOS、リザベーション、cgroup、デバイスの各
  ポリシーが、サイトのリソース制限を強制していること。

prepare の出力がランタイムワークスペースになります。親とワーカーの割り当ては、これを
同一の絶対パスから参照できなければなりません。すべてのワークスペースはプライベートで、
ランタイムアカウントが所有し、1 つの NVFlare サイトまたはフェデレーション専用である
必要があります。

Apptainer の場合は、非特権のユーザ名前空間を有効にし、対象となるすべてのノードに
Apptainer をインストールしてください。本番クラスタ上で、ファイルシステム、プロセス、
cgroup、GPU の分離を検証してください。Pyxis の場合は、対象となるすべてのノードに
Pyxis/Enroot をインストールして設定し、``setup`` の後に ``srun`` が利用できることを
確認してください。

``python_path`` で選択される環境には、互換性のある NVFlare のインストールとジョブの
依存関係が含まれている必要があります。これは bare モードにおけるホスト環境、および
Apptainer と Pyxis のイメージに適用されます。親の起動時のみに使用される ``PYTHONPATH``
のオーバーライドでは、ワーカー環境に NVFlare はインストールされません。

prepare ホスト、投入ホスト、ランタイム親ホストは異なっていても構いません。prepare ホストに
Slurm のコマンドは不要です。生成された ``submit_command`` を実行する投入ホストでは、
``sbatch`` が ``PATH`` 上にある必要があります。ランタイム親ホストでは、そのサービス環境
または ``parent.environment_setup`` の実行後に、親が使用する 4 つのコマンドすべてが必要です。

サイトの設定
=============

スタートアップキットと同じ場所に ``slurm.yaml`` を作成します。この例では、親をログイン
ホストまたはサービスホスト上で実行します。

.. code-block:: yaml

   runtime: slurm
   job_launcher:
     sandbox: apptainer
     image: /lustre/images/nvflare-prod.sif
     python_path: /usr/bin/python3
     parent_host: nvflare-site1.internal
     sbatch_directives:
       partition: fl-gpu
       account: proj123
       time: "12:00:00"
     setup: |
       source /etc/profile.d/modules.sh
       module load apptainer
     pending_timeout: 600

重要なキーは以下のとおりです。

.. list-table::
   :header-rows: 1
   :widths: 28 72

   * - キー
     - 意味
   * - ``sandbox``
     - 必須。``apptainer``、``pyxis``、``none`` のいずれか。
   * - ``image``
     - Apptainer と Pyxis で必要となる、既存のイメージファイルの絶対パス。bare モードでは
       省略します。
   * - ``python_path``
     - 選択した実行環境における、ワーカーのインタプリタの絶対パス (必須)。互換性のある
       NVFlare がインストールされている必要があります。
   * - ``parent_host``
     - 計算ノードから到達可能な親ホスト。親が Slurm の割り当て内にない場合は必須です。
   * - ``sbatch_directives``
     - サイトのデフォルト値。サポートされるキーは ``partition``、``account``、``qos``、
       ``time``、``constraint``、``reservation`` です。
   * - ``setup``
     - ワーカー起動前にバッチジョブ内で実行される、信頼済みのホスト側 Bash スクリプト。
   * - ``forward_env``
     - ``setup`` 実行後の値をワーカーに渡すべき環境変数の名前。
   * - ``executables``
     - Slurm およびバックエンドコマンドの明示的なパス (任意)。親側のパスはワークスペースに
       保持され、その後ランタイム親ホスト上で解決および検証されます。
   * - ``internal_port``
     - ワーカーから親へのポート。デフォルトは ``8102`` です。
   * - ``poll_interval``
     - スケジューラのポーリング間隔。デフォルトは ``10`` 秒です。
   * - ``pending_timeout``
     - 最初に pending 状態が観測された時点から始まる時間制限。デフォルトは ``600`` 秒です。
       ジョブ側でこれより短くすることができます。

``setup`` は ``sbatch --export=NIL`` による最小限の環境から開始されます。module の初期化は
明示的に source してください。固定値はスタディの ``env`` に設定し、``forward_env`` は
setup によって生成される値にのみ使用してください。

準備と起動
===========

共有ランタイムワークスペースに直接 prepare を行います。

.. code-block:: shell

   nvflare deploy prepare ./site-1 \
       --config ./slurm.yaml \
       --output /lustre/proj123/nvflare/site-1

ワークスペースごとに親を 1 つだけ実行してください。同じ出力先に対して prepare を再実行
すると、ワークスペース全体が置き換えられます。更新する前に親を停止し、必要な実行結果、
スナップショット、サーバのジョブストレージを保存してください。

ログインホストまたはサービスホストで親を起動します。

.. code-block:: shell

   /lustre/proj123/nvflare/site-1/startup/start_slurm.sh

クライアント親を Slurm の割り当て内で実行するには、クライアント専用の ``parent``
ブロックを追加し、``job_launcher.parent_host`` を省略します。

.. code-block:: yaml

   parent:
     sbatch_directives:
       partition: batch
       account: proj123
       time: "7-00:00:00"
     environment_setup: |
       source /lustre/proj123/venv/bin/activate

その後、``sbatch`` が ``PATH`` 上にある投入ホストで、prepare が出力した ``submit_command``
を実行します。次のような形式になります。

.. code-block:: shell

   sbatch --parsable \
       --output=/lustre/proj123/nvflare/site-1/parent-slurm-%j.out \
       /lustre/proj123/nvflare/site-1/startup/parent.slurm

親スクリプトは、NVFlare を起動する前に ``parent.environment_setup`` を実行します。次に
親のブートストラップが ``sbatch``、``squeue``、``sacct``、``scancel`` を一度だけ解決し、
そのプロセスの間、正規化されたパスをメモリ内に保持します。したがって、明示的に設定された
パスは ``.../slurm/current/bin/sbatch`` のような、クラスタが管理する安定したシンボリック
リンクを指していても構いません。クラスタのアップグレード後、再起動された親はワークスペースを
再度 prepare することなく新しいターゲットを解決します。

明示的な ``parent_host`` が常に優先されます。それ以外の場合、割り当て内の親は
``SLURMD_NODENAME`` を使用します。``parent_host`` がなく、かつ割り当ての外にある親は
ジョブを起動できません。NVFlare がホスト名を推測したり解決したりすることはありません。

サーバキットは ``parent`` を受け付けません。サーバ親は、安定した外部 NVFlare
フェデレーションエンドポイントを持つ安定したホスト上で実行してください。

.. _slurm_shared_file_channel:

共有ファイルによるワーカーチャネル
====================================

計算ノードが親ホストへ TCP 接続を確立できないものの、Lustre のような POSIX 一貫性のある
ファイルシステムを親ホストと共有している場合、ワーカーから親へのチャネルを TCP ではなく
共有ファイル上で動作させることができます。prepare を実行する前に、クライアントキットの
``local/comm_config.json`` を設定してください。

.. code-block:: json

   {
     "backbone": {"connect_generation": 1},
     "internal": {
       "scheme": "shared-file",
       "resources": {
         "root_dir": "/lustre/proj123/nvflare/site-1-cellnet",
         "connection_security": "clear"
       }
     }
   }

``root_dir`` は絶対パスであり、親ホストとすべての計算ノードで同一のパスから参照できる
必要があります。``connect_generation: 1`` により、すべてのジョブトラフィックが親を経由して
ルーティングされるため、ワーカーにはネットワーク接続性が一切不要になります。
``nvflare deploy prepare`` はファイルベースの comm 設定をそのまま保持し、TCP のホストと
ポートのパッチを適用しません。この場合、``internal_port`` と ``parent_host`` はワーカー
チャネルには使用されません。

実行時、親は ``root_dir`` の下にリスナーディレクトリを作成し、その ``shared-file://0/...``
URL を変更せずに各ワーカーへ渡します。Apptainer と Pyxis のジョブは、リスナーディレクトリを
コンテナ内の同一パスに読み書き可能な形で自動的にバインドマウントします。bare のジョブは
それを直接使用します。ディレクトリは umask に関係なくモード ``0o770`` で、ログファイルは
``0o660`` で作成されます。このチャネルにおける唯一のアクセス制御はディレクトリのパーミッション
であるため、``root_dir`` は専用のサイトアカウントが所有し、必要以上に広いグループアクセスを
与えないようにしてください。

ポーリング間隔、リースのタイミング、fsync の挙動は ``internal.resources`` マップで調整
できます。パラメータとそのファイルシステムメタデータのコストについては、
``nvflare.fuel.f3.drivers.file_driver`` にある ``FileDriver`` のドキュメントを参照して
ください。デフォルト設定では、アイドル状態の接続はクライアント側で毎秒およそ 1.4 回の
メタデータシステムコールを発行します。``max_poll_interval`` を大きくすると、最初の
メッセージのレイテンシと引き換えにアイドル時の負荷が比例して減少します。データ転送は
ポーリング設定の影響を受けません。

スタディの設定
===============

サイトが所有するスタディの設定は ``local/study_runtime.yaml`` に記述し、起動のたびに
再読み込みされます。

.. code-block:: yaml

   format_version: 2
   studies:
     pathology:
       container:
         image: /lustre/images/pathology.sif
       datasets:
         slides:
           source: /lustre/data/pathology/slides
           mode: ro
       env:
         MODEL_FAMILY: vit-large
       secret_env:
         DB_PASSWORD:
           source: PATHOLOGY_DB_PASSWORD
       slurm:
         partition: fl-gpu-large
         account: pathology-project

各データセットはコンテナ内の ``/data/<study>/<dataset>`` にマウントされます。
上記の例では、``slides`` が ``/data/pathology/slides`` にマウントされます。

スタディは、サイトのサンドボックス、setup、パーティション、アカウント、QOS を上書き
できます。また、イメージ、環境、データセットのマウント、シークレット環境変数、読み取り専用
のシークレットマウントを指定することもできます。Slurm の ``secret_env`` の source は親環境の
変数名を指定します。その値は一時的なプライベートファイルを介して渡され、バッチスクリプトや
スケジューラのコマンドには書き込まれません。

マウント元は、ランタイムワークスペースの外にある絶対パスでなければならず、そのジョブを
実行できる計算ノード上に存在している必要があります。bare モードにはマウント名前空間が
ないため、コンテナ、データセット、シークレットマウントの設定は拒否されます。Slurm を
使用する前に、レガシーの ``local/study_data.yaml`` ファイルを ``study_runtime.yaml`` へ
移行してください。

ジョブの設定
=============

ジョブでは、可搬性のある GPU の総数を ``resource_spec`` に、Slurm 固有の設定を
``launcher_spec`` に記述します。次の例は有効なシングルノードのコンテナ要求です。

.. code-block:: json

   {
     "resource_spec": {
       "site-1": {"num_of_gpus": 1}
     },
     "launcher_spec": {
       "site-1": {
         "slurm": {
           "image": "/shared/images/nvflare-job.sif",
           "cpus_per_node": 8,
           "mem_per_node": 32768,
           "time": "02:00:00",
           "pending_timeout": 300
         }
       }
     }
   }

サポートされるジョブのキーは ``image``、``nodes``、``gpus_per_node``、
``cpus_per_node``、``mem_per_node`` (MiB 単位)、``time``、``pending_timeout`` です。
ジョブのイメージには通常の BYOC 認可が必要で、スタディおよびサイトのイメージよりも
優先されます。実効サンドボックスが ``none`` であるサイトでは、ジョブおよびスタディの
イメージは拒否されます。

マルチノードジョブでは実効的に ``sandbox: none`` である必要があり、イメージを指定しては
なりません。マルチノードで ``num_of_gpus`` が正の値の場合は ``gpus_per_node`` が必要です。
両方が指定される場合、``num_of_gpus`` は ``nodes * gpus_per_node`` と等しくなければ
なりません。マルチノードのプロセス展開はアプリケーション側の責任です。

セキュリティと運用
===================

ワーカーから親への内部チャネルは、Kubernetes ランチャーと同様に平文の TCP です。あるいは
ファイルトランスポートが設定されている場合は平文の共有ファイル I/O になります
(:ref:`slurm_shared_file_channel` を参照)。信頼されたサイトネットワークまたはファイル
システム、あるいは分離されたそれらの上でのみ使用してください。これは外部 NVFlare
フェデレーションチャネルに設定されたセキュリティを変更するものではありません。

アカウンティングが正しく動作していることは必須です。``sacct`` が利用できない場合、親は
起動を拒否します。その後にスケジューラやアカウンティングの障害が発生した場合、影響を
受けたジョブは非終了状態のまま残されリトライされます。観測できないことをジョブが停止した
ことの根拠とみなすことは決してありません。

NVFlare は一時的な起動アーティファクトを ``<prepare-output>/.nvflare_slurm`` の下に保存
します。稼働中の親は、起動失敗時または終了完了時にジョブのアーティファクトを削除します。
ユーザによる中止と pending タイムアウトは、``scancel`` の前に所有権を検証します。通常の
フレームワークのシャットダウンでは、Slurm ランチャーが起動の受け付けを閉じる前に、同じ
経路を通じて実行中のハンドルを終了させます。

Slurm のジョブ名には NVFlare のサイト名の先頭 32 文字と短いジョブハッシュが含まれるため、
1 つの Slurm ユーザを共有するサイトどうしをオペレータが区別できます。ジョブの出力は
``<run-dir>/slurm-<slurm-job-id>.out`` です。稼働中の状態は ``squeue`` で、完了したジョブは
``sacct`` で確認してください。調査のため、ワークスペース全体と関連するスケジューラの記録を
保存してください。

親のクラッシュ後、ランチャーは残存する割り当てをキャンセルしたり、そのジョブアーティファクトを
削除したりしません。これは Docker および Kubernetes のランチャーと同じ挙動です。
残されたジョブディレクトリは、古い割り当てがそれを使用していないことをオペレータが確認して
ディレクトリを削除するまで、そのジョブ ID の再起動をブロックします。

``sbatch`` のタイムアウトは、Slurm がジョブを受理していたとしても FL のディスパッチを失敗
させます。アーティファクトの削除により、同じジョブ ID が開始前に再起動されない限り、
pending 状態の割り当てが開始されるのを防ぎます。

ワークスペースを変更するには、親を停止して新しい ``--output`` へ prepare を実行します。
既存の出力先に prepare を実行すると置き換えられるため、残す必要のあるデータは prepare の
実行前にコピーしてください。

サーバジョブの開始後、クライアントはその SJ が利用可能になるのを待ちます。SJ の Slurm の
キュー待ち時間と起動時間は、クライアント側の ``max_runner_sync_timeout`` (デフォルトでは
60 秒) 以内に収まる必要があります。サイトでより長い上限が必要な場合は、ジョブのクライアント
側 ``config_fed_client.json`` でこの値を設定してください。:ref:`timeout_troubleshooting`
を参照してください。

本番で使用する前に、対象クラスタで以下をテストしてください。

#. 成功、失敗、タイムアウト、pending 中の中止、実行中の中止の各ジョブ。
#. 稼働中のジョブがある状態での親の再起動、およびワークスペース置き換え時の挙動。
#. 計算ノードから親への接続性、および使用する場合はマルチノードの集団通信。
#. Slurm のアカウンティング、アソシエーション、QOS、パーティション、cgroup、GPU の制御。
#. 選択したバックエンドのファイルシステムビュー、環境、シークレットの扱い、終了ステータス、
   CPU/GPU デバイスの可視性。

これらの分離チェックがデプロイ先のクラスタで合格するまで、Apptainer を本番用のサンドボックス
として説明しないでください。
